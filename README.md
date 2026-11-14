# gscore-patches

对 GsCore（早柚核心）及其插件的本地改动补丁集。

这些改动不是上游的，是这台机器上为了把「QQ 官方机器人 + GsCore + GenshinUID / StarRailUID / ZZZeroUID」跑通而做的。
放在这里是为了**升级后能照着重打**，以及让改动可追溯。

- 环境：GsCore v0.11.3，Ubuntu 24.04，Python 3.12，服务部署在 `/opt/gsuid_core`
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

**ZZZeroUID 不用改**：`zzzerouid_char_detail/__init__.py` 第 45-47 行本来就有
`on_suffix("面板")`，`%安比面板` 原生支持。

---

## 纯配置改动（没有代码补丁，但要一起做）

| 文件 | 键 | 改成 | 作用 |
|---|---|---|---|
| `data/plugins_configs/core_command.json` | `allow_empty_prefix` | `true` | 允许空前缀命令 |
| `data/plugins_configs/core_command.json` | `disable_force_prefix` | `false` | 保留 `core` 前缀 |
| `data/plugins_configs/core_command.json` | `sv.添加CK.area` | `"ALL"` | 群聊可用「绑定设备」 |
| `data/configs/pic_gen_config.json` | `PicQuality` | `65`（原 85） | 降低出图体积，加快发送 |

`PicQuality` 是 GsCore 的全局出图质量，改动后需重启（模块级变量）。

---

## 应用方法

每个 `.patch` 是标准 unified diff，`a/` `b/` 前缀对应仓库根目录。

```bash
cd /opt/gsuid_core

# 先确认能不能干净打上
patch -p1 --dry-run < /path/to/patches/xxx.patch

# 实际应用
patch -p1 < /path/to/patches/xxx.patch

# 改完重启
systemctl restart gscore
```

升级 GsCore 或插件后，这些文件会被覆盖，需要重新打一遍。
打完务必检查上面「纯配置改动」那张表仍然成立。

---

## 已知代价

- 第 2、3 条改的是内置插件源码，升级会被覆盖
- 第 4、5 条改的是第三方插件源码，插件更新会被覆盖
- `PicQuality` 降到 65 后面板图细节会有损失，觉得糊就调回 75 左右
- 群聊「绑定设备」会让设备指纹公开，介意就改回 `DIRECT`
