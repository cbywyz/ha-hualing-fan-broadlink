# Home Assistant 接入华凌风扇（WAHIN WH-FGA2401）完整教程

> Broadlink RM3 红外学码方案，附**全套遥控器编码库**（8 键，Broadlink Base64 格式），拿来即用。
> 实测环境：HA 2026.9 + Broadlink RM mini 3 + 华凌直流风扇 WH-FGA2401（2026-09）。

姊妹篇：[ha-midea-hualing-ac](https://github.com/cbywyz/ha-midea-hualing-ac)（美的/华凌**空调**云端接入教程）。

---

## 一、为什么华凌风扇只能走红外

华凌风扇（WAHIN 系列）**没有 WiFi 模块**，美居 App 里控制它靠的是「万能遥控器」——那只是 App 里存的一组红外码配置，不是真实联网设备：

- 云端设备列表里查不到它，任何云端集成（xiaomi_home / midea_auto_cloud 等）都读不出来；
- 美居 App 的码库接口是加密的，码**导不出来**；
- SmartIR 官方码库（17 个风扇码）没有美的/华凌系，GitHub 全网搜 `WH-FGA2401` 也没有现成码。

所以唯一入口是**红外学习**：用 Broadlink RM3（mini 3 / RM4 等都行）把遥控器学一遍。

## 二、准备：Broadlink RM3 接入 HA

已接入的跳过。RM3 在 HA 里走 Broadlink 集成，**RM3 和 HA 同网段的话**（绝大多数家庭的情形）：配置 → 添加集成 → Broadlink，自动发现直接点添加就完事，得到一个 `remote.xxx` 实体。

> <details><summary>特殊环境：RM3 不在 HA 所在网段怎么办（点开）</summary>
>
> 广播发现跨不过网段，但只要主路由有到该网段的路由（单播可达），配置流里**手动填 IP** 即可：
>
> ```yaml
> # 添加集成 → Broadlink → 手动 IP
> host: 192.168.8.102   # 你的 RM3 IP
> timeout: 10
> ```
>
> 我们的环境是 RM3 挂在旁路由的 192.168.8.x 下，iStoreOS 加一条 `192.168.8.0/24 via 旁路由` 静态路由，实测 9ms 稳通，一次握手成功。
>
> </details>

## 三、学码（两种姿势）

### 姿势 A：原装遥控器对着 RM3 按（最稳）

```yaml
# 逐键学习，每键一次调用，35 秒内按键
action: remote.learn_command
target:
  entity_id: remote.pang_lu_rm3yao_kong   # 你的 RM3 实体
data:
  device: hualing_fan      # 码组名，随意
  command: power           # 键名：power / speed / speed_down / osc / osc_ud / timer / circle_wind / custom_wind
  command_type: ir
```

### 姿势 B：没有原装遥控器？手机当发射器

我们就是这么干的——遥控器早找不着了，但**任何能发红外码的手机 App（美的美居万能遥控器、米家、遥控精灵等）对着 RM3 按键**，RM3 学习模式照样能录。原理：RM3 不关心码是谁发的，收到红外信号就录。

> 美居 App 里「随便选个华凌风扇型号都能控制」——码库兼容性很好，选 WH-FGA2401 或同品牌任意落地扇都行。

> ⚠️ 小提示：`learn_command` 是**阻塞等待**的——调用后 HA 会一直等到学到码或超时（默认 30 秒）。如果窗口内没按键，会弹「Timeout」错误，**这是正常现象**，重新调用再按就行；手慢可以把 `timeout: 60` 调大些。

### 学完的码存在哪

`/config/.storage/broadlink_remote_<MAC去冒号>_codes`，结构：

```json
{ "data": { "hualing_fan": { "power": "JgAiAAAB...", "speed": "...", ... } } }
```

## 四、不想学？直接用我们的编码库

[`codes/hualing_whfga2401.json`](codes/hualing_whfga2401.json) 收录了全遥控器 **8 键**码（Broadlink Base64，Power 协议）：

| 键名 | 功能 | 键名 | 功能 |
|------|------|------|------|
| `power` | 开/关（切换） | `osc_ud` | 上下摇头 |
| `speed` | 风速+（循环升档） | `timer` | 定时 |
| `speed_down` | 风速− | `circle_wind` | 循环风 |
| `osc` | 左右摇头 | `custom_wind` | 自选风 |

码不用学习、也不用导入任何文件——把 json 里的码直接作为 command 值发即可，任何人的 HA、任何一台 Broadlink 都通用：

```yaml
action: remote.send_command
target:
  entity_id: remote.pang_lu_rm3yao_kong   # 换成你的 remote 实体
data:
  command: >-
    JgAiAAABJJUTExITExMSExEUExMRFBMTEjgSNxM3EzcTNxMADQUAAAAAAAA=
```

> 注意：不同批次遥控器红外码可能有差异，先发 `power` 试试，风扇有反应再批量用；万一某键无效，用第三节姿势 A 自己学一遍（一分钟的事）。
>
> **`device` 参数什么时候要填**：只有发「你自己学进 storage 的码」时才要 `device: hualing_fan`（它用来定位码组）；直接发裸码（上面这种）**不需要** `device`，发给任何一台 Broadlink 都行。

## 五、面板遥控（Lovelace）

完整弹窗示例见 [`examples/lovelace-fan-popup.yaml`](examples/lovelace-fan-popup.yaml)：Bubble Card 弹窗 + Mushroom chips，两行 9 颗按钮，核心就是每颗 chip 调一次 `remote.send_command`。

**两个必须知道的限制**：

1. **红外单向，无状态反馈**——面板不知道风扇实际开没开、几档风。想要状态只能外挂智能插座（测功率判断档位）；
2. **风速是相对键**（+/−），不是绝对档位——按几下自己数。

## 六、折腾实录（踩坑路径）

1. 先想走云端：美居账号接进 HA（`midea_auto_cloud`，见姊妹篇），结果账号下根本没有风扇——它是 App 内虚拟设备；
2. 查 SmartIR 官方 fan 码库：17 个码无一美的系；GitHub 搜型号：0 结果；
3. 想扒美居 App 本地数据库：博联版 `_UnifyApp.db` 文件头都不是 SQLite（加密），放弃；
4. 转念：手机美居遥控能控制风扇 = 手机在发红外 = 对着 RM3 发也行 → 学习模式一次录完 4 键；
5. 用户指出漏键，分两批补齐到全 9 键（8 颗独立码），发射验证通过。

**一句话总结**：无 WiFi 的风扇，别在云端和码库里找了，RM3 + 学习模式（或本仓库现成码）是唯一正路。

## 七、相关仓库（同一系列教程）

- [cbywyz/ha-midea-hualing-ac](https://github.com/cbywyz/ha-midea-hualing-ac) —— 美的/华凌**空调**云端接入教程（midea_auto_cloud）：为什么华凌空调只能走云端、美居账号配置流程、实体对照表、与本地 midea_ac_lan 共存。空调有 WiFi 走云端，本仓库的风扇没 WiFi 走红外，正好互补。
- [cbywyz/gree-yapqf-broadlink-smartir](https://github.com/cbywyz/gree-yapqf-broadlink-smartir) —— 格力空调（YAPQF）Broadlink + SmartIR 接入教程：与本仓库同源的 Broadlink 红外路线，SmartIR 码表排坑经验很全。

## 八、参考

- [SmartIR](https://github.com/smartHomeHub/SmartIR)（空调/电视码库可参考，风扇无美的系）
- [Broadlink 集成文档](https://www.home-assistant.io/integrations/broadlink/)
- 姊妹篇：[ha-midea-hualing-ac](https://github.com/cbywyz/ha-midea-hualing-ac) —— 美的/华凌**空调**云端接入
