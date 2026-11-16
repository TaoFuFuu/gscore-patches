# gscore-patches

对 GsCore（早柚核心）及其插件的本地改动补丁集。

这些改动不是上游的，是这台机器上为了把「QQ 官方机器人 + GsCore + GenshinUID / StarRailUID / ZZZeroUID」跑通而做的。
放在这里是为了**升级后能照着重打**，以及让改动可追溯。

- 记录时的版本：GsCore v0.11.3，GenshinUID 7.0.0
- 环境：Ubuntu 24.04，Python 3.12，部署目录 `/opt/gsuid_core`
- 上游：https://github.com/Genshin-bots/gsuid_core
- 记录时间：2026-11-14

---

## 补丁清单

### 1. `gsuid_core/utils/database/api.py`

**问题**：QQ 官方机器人消息里 @ 机器人时，`ev.at` 会被填成 bot 自身标识（如 `qq_official`）。
框架的 `get_uid()` 直接把它当 `user_id` 查绑定表，于是永远查不到，星铁等插件一直回「你还没有绑定过uid」。

GenshinUID 自带的 `get_uid` 早就排除了这种情况，框架公共版没有。

**改动**：

```python
# 改前
user_id = ev.at if ev.at else ev.user_id

# 改后
_at = ev.at if ev.at and ev.at not in (ev.bot_id, ev.bot_self_id) else None
user_id = _at if _at else ev.user_id
```

**影响**：所有用框架 `get_uid` 的插件（StarRailUID、ZZZeroUID 等）。
GenshinUID 用自己的那份，不受影响。

**症状对照**：私聊 `*绑定uid` 成功，群里 `*强制刷新` 却说没绑定。

---

### 2. `gsuid_core/buildin_plugins/core_command/__init__.py`

**问题**：`core_command` 的 `allow_empty_prefix=False` 写死在代码里。
插件初始化早于配置文件加载时，`allow_empty_prefix` 会退回代码默认值 `False`，
导致 JSON 里改成 `true` 也不生效，`绑定设备{...}` 这类空前缀命令始终注册不上。

**改动**：`allow_empty_prefix=False` → `True`。

**注意**：只改代码不够，还要保证 `data/plugins_configs/core_command.json` 里
`allow_empty_prefix` 是 `true`、`disable_force_prefix` 是 `false`
（`disable_force_prefix: true` 会把 `core` 前缀也去掉，结果哪条路都没有）。

---

### 3. `gsuid_core/buildin_plugins/core_command/user_login/__init__.py`

**问题**：`SV("添加CK", area="DIRECT")` 让「绑定设备」「设备登录」「mys绑定设备」只能在私聊触发。
群里发同样的内容会 `sv_ok=False`，连命令匹配都进不去。

**改动**：`area="DIRECT"` → `area="ALL"`。

**注意**：这一条 **代码和配置要一起改**。`data/plugins_configs/core_command.json` 的
`sv.添加CK.area` 也必须改成 `ALL`，否则配置会盖掉代码。
改完群聊能用，但设备指纹 JSON 会在群里明文可见，看场合。

---

### 4. `gsuid_core/plugins/GenshinUID/GenshinUID/genshinuid_enka/__init__.py`

**需求**：让 `#胡桃面板` 等同于 `#查询胡桃`。

**改动**：文件末尾追加一个后缀触发器，剥掉前缀和结尾的「面板」，
把角色名交给原来的 `send_char_info()`，出图和按钮逻辑完全复用。

```python
@sv_get_enka.on_suffix("面板", block=True)
async def _panel_by_suffix(bot: Bot, ev: Event):
    """「胡桃面板」等同于「查询胡桃」。"""
    raw = ev.raw_text.strip()
    if not raw.endswith("面板"):
        return
    body = raw[: -len("面板")].strip()
    for _p in ("#", "gs", "GS", "Gs", "gs#"):
        if body.startswith(_p):
            body = body[len(_p) :].strip()
            break
    if not body:
        return
    ev.text = body
    await send_char_info(bot, ev)
```

**注意**：单独发「面板」不触发，必须有角色名。

---

### 5. `gsuid_core/plugins/StarRailUID/StarRailUID/starrailuid_charinfo/__init__.py`

**需求**：让 `*流萤面板` 等同于 `*查询流萤`。做法与第 4 条相同，前缀换成 `*` / `sr`。

```python
@sv_get_char_info.on_suffix("面板", block=True)
async def _panel_by_suffix(bot: Bot, ev: Event):
    """「流萤面板」等同于「查询流萤」。"""
    raw = ev.raw_text.strip()
    if not raw.endswith("面板"):
        return
    body = raw[: -len("面板")].strip()
    for _p in ("*", "sr", "SR", "Sr"):
        if body.startswith(_p):
            body = body[len(_p) :].strip()
            break
    if not body:
        return
    ev.text = body
    await send_char_info(bot, ev)
```

**同一个文件还有第二处改动**：`send_char_info` 开头加一段纯 UID 拦截。

**问题**：发 `*查询102059463` 会同时触发两个命令 —— `command` 匹配走 `sr查询信息`（正常），
`prefix` 匹配走角色面板，把 UID 当角色名。后者必然查不到，还要白打一次 API，
并吐一条 `本地角色缓存读取失败` 的警告。

**做法**：参数里去掉数字和 `换` / `拿` / `带` 之后什么都不剩，说明用户只给了 UID，
不是要查角色面板，直接放行交给 `sr查询信息`。

```python
async def send_char_info(bot: Bot, ev: Event):
    name = ev.text.strip()
    if not name:
        return
    # 只有 UID、没有角色名时，交给「sr查询信息」处理。
    # 拿一串数字当角色名去查面板必然查不到，还会多打一次 API。
    residue = re.sub(r"\d+", "", name)
    for kw in ("换", "拿", "带"):
        residue = residue.replace(kw, "")
    if not residue.strip():
        return
```

`*查询流萤`、`*查询流萤换` 有角色名，不受影响。

**ZZZeroUID 不用改**：`zzzerouid_char_detail/__init__.py` 第 45-47 行本来就有
`on_suffix("面板")`，`%安比面板` 原生支持。

---

### 6. `gsuid_core/plugins/StarRailUID/StarRailUID/starrailuid_charinfo/get_char_img.py`

**需求**：修星铁面板缓存的类型不匹配。

**现象**：日志里反复出现

```
[sr面板] UID102059463 本地角色缓存读取失败:
players/102059463/102059463.json, error=Expected `str`, got `int` - at `$.uid`
```

**原因**：这份缓存是从米游社 API 原样落盘的，`uid` 是 JSON 数字（`"uid": 102059463`，
没有引号）。而 `MihomoCharacter` 模型里 `uid` 声明成 `str`，msgspec 严格解析直接抛错，
**整份缓存作废**，每次都重新请求 API。

**做法**：解析前把 `uid` 统一成字符串，两种写法都能吃。

```python
    try:
        raw = path.read_bytes()
        # 米游社返回的 uid 是数字（JSON int），而 MihomoCharacter 里声明成 str。
        # 直接 decode 会报 Expected `str`, got `int`，整份缓存就作废了。
        # 解析前先把 uid 统一成字符串，兼容两种写法。
        try:
            _d = msgjson.decode(raw)
            if isinstance(_d, dict) and isinstance(_d.get("uid"), int):
                _d["uid"] = str(_d["uid"])
                raw = msgjson.encode(_d)
        except Exception:
            pass
        char_data = msgjson.decode(raw, type=MihomoCharacter)
        source = msgjson.decode(raw).get("source", "cache")
```

**顺带说明 JSON 的类型**：JSON 里 `102059463` 和 `"102059463"` 是两种东西，
前者是数字，后者才是字符串。加不加引号决定的，不是"写在文件里就是字符串"。

---

## 怎么让补丁生效

按顺序做完这五步，少一步都可能不生效。

### 第 0 步：确认版本

```bash
cd /opt/gsuid_core
grep -r "__version__" gsuid_core/version.py 2>/dev/null || cat pyproject.toml | grep -m1 version
```

补丁基于 GsCore v0.11.3 和 GenshinUID 7.0.0 记录。
版本差太远可能打不上，**先跑第 2 步的 dry-run 再决定要不要硬打**。

### 第 1 步：备份

改哪个先备哪个，出问题直接拷回去：

```bash
cd /opt/gsuid_core
cp gsuid_core/utils/database/api.py api.py.bak
cp gsuid_core/buildin_plugins/core_command/__init__.py core_init.py.bak
cp gsuid_core/buildin_plugins/core_command/user_login/__init__.py user_login.py.bak
cp gsuid_core/plugins/GenshinUID/GenshinUID/genshinuid_enka/__init__.py gs_enka.py.bak
cp gsuid_core/plugins/StarRailUID/StarRailUID/starrailuid_charinfo/__init__.py sr_charinfo.py.bak
cp gsuid_core/plugins/StarRailUID/StarRailUID/starrailuid_charinfo/get_char_img.py sr_char_img.py.bak
```

### 第 2 步：先 dry-run，别直接打

```bash
cd /opt/gsuid_core
for p in /path/to/gscore-patches/patches/*.patch; do
    echo "=== $p"
    patch -p1 --dry-run < "$p"
done
```

输出里只要出现 `FAILED` 或 `Hunk #N FAILED` 就停手，说明那个文件的上下文变了，
版本对不上，需要手工改（`rej` 文件里有冲突片段可以参考）。

全部 `succeeded` 才继续。

### 第 3 步：真正应用

```bash
cd /opt/gsuid_core
for p in /path/to/gscore-patches/patches/*.patch; do
    echo "=== $p"
    patch -p1 < "$p"
done
```

补丁会生成 `.orig` 备份文件，打完可以删。

### 第 4 步：改配置（这一步不做，代码改了也白改）

一共 4 个键，三个文件：

**`data/plugins_configs/core_command.json`**

```json
{
    "allow_empty_prefix": true,
    "disable_force_prefix": false,
    "sv": {
        "添加CK": {
            "area": "ALL"
        }
    }
}
```

`allow_empty_prefix` 和 `disable_force_prefix` 必须同时满足，`添加CK.area` 必须和代码里的 `area="ALL"` 一致，
否则配置盖代码，群里还是不响应。

**`data/configs/pic_gen_config.json`**

```json
{
    "PicQuality": {
        "data": 65
    }
}
```

原值是 85，降到 65 图小一圈、发送快一些。面板细节会损失一点，嫌糊就调回 75。
这一项是**可选**的，只影响速度不影响功能。

改完检查 JSON 别写坏（多一个逗号后台就起不来）：

```bash
python3 -c "import json,pathlib;[json.loads(pathlib.Path(p).read_text(encoding='utf-8-sig')) for p in ['/opt/gsuid_core/data/plugins_configs/core_command.json','/opt/gsuid_core/data/configs/pic_gen_config.json']];print('JSON OK')"
```

### 第 5 步：重启

```bash
systemctl restart gscore
sleep 20
systemctl is-active gscore
```

**必须真重启，不能只 reload。** 原因是两处都只在 import 时读一次：

- 命令前缀表 `SV.TL` 是模块导入时构建的，配置改了不重建
- `pic_quality` 是 `convert.py` 里的模块级变量，进程不换就还是旧值

判断有没有真的重启过：

```bash
systemctl show gscore -p ActiveEnterTimestamp
ps -o pid,lstart -p $(systemctl show gscore -p MainPID --value)
```

两个时间对得上才算换过进程。

### 第 6 步：验证

按这张表逐条过，都通过才算装好。

| 补丁 | 怎么测 | 期望结果 |
|---|---|---|
| 1 `get_uid` | 群里发 `*强制刷新` | 出星铁面板，不再回「你还没有绑定过uid哦」 |
| 2 `allow_empty_prefix` | 私聊发 `绑定设备{"deviceModel":"...",...}` | 回「设备绑定成功!」 |
| 3 `添加CK` area | 群里 @ 机器人发同样的 `绑定设备{...}` | 同上，群里也能回 |
| 4 GenshinUID 面板 | 发 `#胡桃面板` | 出胡桃面板图，和 `#查询胡桃` 一样 |
| 5 StarRailUID 面板 | 发 `*流萤面板` | 出流萤面板图，和 `*查询流萤` 一样 |
| 5b 纯 UID 不误匹配 | 发 `*查询102059463` | **只回一条** UID 信息，不再同时冒出一条面板报错 |
| 6 缓存 uid 类型 | 发 `*面板流萤` 后看日志 | 不再出现 `Expected str, got int - at $.uid`；第二次查同一个 UID 走缓存 |
| PicQuality | 对比修改前后的图 | 体积变小，肉眼还能看 |

命令里带 `{...}` 的部分必须是合法 JSON，且不能有换行。`绑定设备` 后面什么都没有是**不会触发**的，
`on_prefix` 要求关键词后面还有内容。

排查用这两条日志：

```bash
journalctl -u gscore --since "5 min ago" | grep '命令触发'   # 有没有匹配上
journalctl -u astrbot --since "5 min ago" | grep '即将发送'   # 有没有回出来
```

### 第 7 步：回滚

单个文件回滚：

```bash
cd /opt/gsuid_core
cp api.py.bak gsuid_core/utils/database/api.py
systemctl restart gscore
```

整批回滚就是第 1 步那五条反过来拷一遍。
配置回滚把第 4 步的键改回原值（`allow_empty_prefix: false`、`area: "DIRECT"`、`PicQuality: 85`）。

---

## 升级后要重新打

GsCore 升级会覆盖 `gsuid_core/` 下的全部文件，插件升级会覆盖 `plugins/` 下的全部文件。
所以**每次升级完，从第 2 步重新走一遍**。

升级后最先坏的一般是补丁 4 和 5（插件改得勤），其次是 1（框架核心文件）。

---

## 已知代价

- 第 2、3 条改的是内置插件源码，升级会被覆盖
- 第 4、5 条改的是第三方插件源码，插件更新会被覆盖
- `PicQuality` 降到 65 后面板图细节会有损失，觉得糊就调回 75 左右
- 群聊「绑定设备」会让设备指纹公开，介意就改回 `DIRECT`
