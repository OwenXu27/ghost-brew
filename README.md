# GHOST//BREW

**手冲咖啡幽灵曲线教练** — 一个连接 AtomHeart Eclair 蓝牙电子秤的单文件网页应用。
倒一杯好咖啡，全程不用碰手机/App：连接、称豆、开始、停止，全部自动。

![screenshot](screenshot.png)

## 这是什么

对着一条"幽灵曲线"冲咖啡：App 把目标注水曲线（或你某次完美冲煮的历史曲线）画在图上作为影子，你实际的注水曲线实时叠上去，配合容差带、阶段提示和语音/蜂鸣提醒，让每一杯都复刻理想节奏。

整个 App 就是**一个 `index.html`**——没有构建工具、没有后端、没有依赖安装，数据全部存在浏览器本地（IndexedDB）。

## 零操作工作流

配合 Eclair 秤的手冲模式，完整流程只需要动秤，不需要动 App：

1. 打开页面 —— 自动连接已配对过的秤（Web Bluetooth `getDevices()`）
2. 在秤上称豆 —— App 被动嗅探稳定重量，自动识别粉量（`~15.2` = 自动识别）
3. 直接注水 —— 秤检测到流速 > 0.3 g/s 自动开始计时，App 通过蓝牙计时状态消息（`C`）**跟随秤的状态自动开始记录**
4. 移开滤杯 —— 秤自动停止计时，App 自动停止并弹出存档

## 功能

- **幽灵曲线教练**：目标曲线 + 容差带；脉冲式配方的阶段提示**按称重判定**——注水/闷蒸段在实际克重达到目标的瞬间才播报（½ 克容差吸收秤的延迟），等待段从上段结束起按时长倒计时；蜂鸣 + 中文/英文语音播报
- **比例配方**：配方可设「设计粉量」（如 OREA 按 12g 设计），实际粉量不同（12 克 / 16 克）时全部目标克数自动等比缩放，一份配方通吃
- **五宫格大读数**：粉量 · 时间 · 重量 · 流速 · 水粉比（水粉比基于自动识别的粉量实时计算）
- **跟随秤状态机**：订阅秤的计时状态通知（`C 0x43`），双向同步开始/停止；流速启发式作为兜底
- **粉量自动嗅探**：空闲时检测 5–100g 的稳定重量平台并自动采用（滤杯+分享壶 >100g 不会误触发）；也保留手动「锁定粉量」
- **配方管理**：多配方（目标/编辑/复制/删除）、JSON 导入导出、文本快速导入（粘贴「闷蒸 0:45 50g…」直接解析成配方）；内置 CLASSIC 1:15 和 OREA 两个预设
- **冲煮存档**：每次冲煮自动记录（重量/流速采样、粉量、时长），可命名保存，历史曲线可作为"影子"再次引导
- **中英双语**、**暗色/亮色主题**、自动重连，偏好全部持久化
- **演示模式**：没有秤也能玩 —— 打开 `index.html?demo` 会模拟一台秤并自动冲一壶

## 使用方法

Web Bluetooth 要求 Chrome / Edge，且页面必须通过 `localhost` 或 HTTPS 访问：

```bash
cd ghost-brew
python3 -m http.server 8000
# 打开 http://localhost:8000
```

首次点击「连接」选择你的 `ECLAIR-XXXX` 秤；之后打开页面即自动连接。

## BLE 协议摘要

基于官方公开的 [Eclair 蓝牙协议](https://ef.atomheart.cn/eclair-scale-bluetooth-protocol.html)（设备名前缀 `ECLAIR-`）：

| 方向 | 消息 | 头字节 | 内容 |
|---|---|---|---|
| 秤 → App | `W` | `0x57` | 重量 (int32 LE, mg) + 计时 (uint32 LE, ms) |
| 秤 → App | `F` | `0x46` | 流速 (int32 LE, mg/s) + 计时 (ms) |
| 秤 → App | `C` | `0x43` | 计时状态：`0x01` 开始 / `0x00` 停止（每次变化发 3 次） |
| 秤 → App | `B` | `0x42` | 电量百分比 |
| App → 秤 | `T` | `0x54` | 去皮（FW 2.1.0+ 不复位计时） |
| App → 秤 | `R` | `0x52` | 复位计时（计时运行中会被忽略，需先停止） |
| App → 秤 | `S` | `0x53` | 去皮 + 复位 + 开始计时 |
| App → 秤 | `E` | `0x45` | 停止计时 |

数据通道：`W`/`F` 走 DATA 特征值；`C`/`B` 与命令共用 CONFIG 特征值（记得同时订阅两个特征的 notify）。包格式 = 头字节 + 负载 + 负载 XOR 校验。

> 注：秤的手冲子页面状态（称豆/确定粉量/冲煮）与粉量**不会**通过蓝牙广播——协议里没有此类消息。App 的粉量自动嗅探正是为此设计的。

## 技术

- 单文件 HTML/CSS/JS，Canvas 绘制曲线（容器查询自适应字号，一屏显示）
- IndexedDB 三个仓库：`brews`（存档）、`recipes`（配方）、`kv`（偏好）
- 图表锚定压缩设计：时间轴从 0 锚定、窗口只增不减，全程曲线（含目标线）始终完整在屏——无滑动丢历史，也无跳变；停止后定格为全量复盘视图

---

## English

**Ghost Brew** — a ghost-curve coach for pour-over coffee, built as a single-file web app for the AtomHeart Eclair BLE scale.

- Target/shadow curves with a tolerance band, pulse-pour recipes whose cues are **weight-driven** (pour stages announce the moment the scale hits the target grams, waits count down by duration), beep & voice cues
- **Ratio-based recipes**: set a design dose (e.g. OREA = 12g) and every stage target scales to your actual dose — one recipe fits both 12g and 16g
- Follows the scale's own state machine over BLE (`C` timer-state notifications): pouring starts the recording, lifting the dripper stops it — zero app interaction needed
- Passive dose sniffing (stable-plateau detection) powers a live brew-ratio readout
- Multi-recipe management with JSON import/export and a text quick-import; brew archive (IndexedDB); zh/en UI, dark/light themes
- Try it without a scale: open `index.html?demo`

**Run:** serve the folder (`python3 -m http.server 8000`) and open it in Chrome/Edge — Web Bluetooth requires localhost or HTTPS.

Protocol credit: [AtomHeart Eclair BLE protocol](https://ef.atomheart.cn/eclair-scale-bluetooth-protocol.html).

## License

MIT
