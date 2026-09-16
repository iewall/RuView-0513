# 手机 Wi‑Fi 芯片 CSI 闭环优化方案

> 落地目标：手机 SoC 内的 Wi‑Fi 协议栈、固件与低功耗核（LPP），以及与 AP 的空口协同。  
> 本仓库（RuView / 802.11bf 协议模型、CSI 帧、CIR、BFLD）仅作**技术可行性参考**，不代表已在消费级手机硅片上完成 MEASURED 闭环。  
> 准确率/功耗数字未在本文件中作为产品承诺；上线须以现网速率机与电量仪对照。

---

## 1. 范围与角色

手机是 **STA**（Station，站点），不是家用路由。CSI 有两条合法出口，不能混用：

| 出口 | 对端 | 形态 | 用途 |
|---|---|---|---|
| **空口上报** | **AP**（Access Point，接入点） | 802.11 管理/控制帧：CBFR / BFI，未来 802.11bf Sensing Measurement Report | 下行波束、MU 调度、AP 侧 MCS；可选感知会话 |
| **片内上报** | **LPP**（Low-Power Processor，低功耗核） | 共享内存 / mailbox / GLink 类 IPC，不出应用层 | 运动门控、拥塞/干扰/遮挡分类、预测式 MCS、TWT/OMN 决策 |

商店 App **不得**拿原始 CSI。LPP 只输出有界事件（占位/运动/病因/推荐 MCS 上界），由 Wi‑Fi 固件执行。

```
                    ┌─────────────────────────────────────────┐
                    │              手机 SoC                    │
  AP ◄──CBFR/bf──►  │  Wi‑Fi PHY ─► MAC/FW ─► 主机 Wi‑Fi 栈   │
                    │       │              │                   │
                    │       │ CSI 快照      │ 执行器            │
                    │       ▼              ▼                   │
                    │     LPP/SLPI/CDSP   OMN/TWT/LA/NSS/BW    │
                    │       ▲                                  │
                    │       └── IMU / 流量水位 ───────────────┘
                    └─────────────────────────────────────────┘
```

---

## 2. CSI 从 Wi‑Fi 协议栈上报：两条路径

### 2.1 路径 A：协议栈 → AP（空口）

这是 802.11ac/ax/be **已经在发**的波束成形反馈，不是新发明。

1. AP 发 **NDP**（Null Data Packet，空数据包）或数据前导中的 **LTF / HE-LTF**（Long Training Field，长训练场）。  
2. STA PHY 估计信道矩阵 \(H(f)\)。  
3. STA 做 **SVD**（Singular Value Decomposition，奇异值分解），把右奇异向量压成 Givens 角 \(\phi/\psi\)。  
4. STA 发 **CBFR**（Compressed Beamforming Report，压缩波束成形报告），即 **BFI**（Beamforming Feedback Information）。

约束：

- CBFR 是管理/动作帧，格式、Ng、码本比特由能力协商决定；STA **不能**把任意私有 CSI blob 塞进标准 CBFR。  
- 上报周期主要由 **AP 的 sounding 调度**决定；STA 可做的是：降精度、合并、阈值不上报、用 OMN 声明更窄的接收模式以减少被探测。  
- **802.11bf-2025** 增加独立的 Sensing Measurement Setup / Instance / Report，以及 **SBP**（Sensing by Proxy，感知代理：STA 请 AP 代测再回传有界报告）。消费级硅片暴露该接口前，空口仍以 CBFR 为主。本仓库 ADR-153/310 对上述过程做了类型化模型（非认证实现）。

### 2.2 路径 B：协议栈 → LPP（片内）

LPP 指手机上常开或浅睡的低功耗核（各家商品名不同：Sensor Hub / SLPI / CDSP / Tiny 核），用来跑轻量分类，避免每次 CSI 都唤醒 AP 应用处理器。

推荐数据面（与本仓库 ADR-356「CSI 回调不得在 Wi‑Fi 任务里做重计算」同构）：

1. **Wi‑Fi ISR / Wi‑Fi 任务**：只做合法性检查、拷贝或 DMA 到环形缓冲、置位 mailbox。禁止在此做 ISTA/CIR/NN。  
2. **LPP**：消费环形缓冲，算特征与状态机，写回「决策结构体」。  
3. **Wi‑Fi 固件执行器**：在下一个 TXOP / 下一次 OMN/TWT 更新点应用决策，**不得**与正在发送的 PPDU 中途改 MCS/BW。

### 2.3 双路径协调原则

| 原则 | 说明 |
|---|---|
| 一份 CSI，两套视图 | PHY 只估一次 \(H\)；空口走标准 BFI；LPP 走内部快照或压缩特征。 |
| 空口优先保证连通 | LPP 建议与 AP 调度冲突时，**连通与 MU 反馈时限优先**。 |
| 原始 \(H\) 不出 LPP | 出核只有特征或有界事件；CBFR 按 802.11 规范，不另附私有 CSI。 |
| 人因与通信解耦 | 「人体遮挡」只改本机 MCS/NSS；不把人体当脏信道去催 AP 切主信道。 |

---

## 3. 上报周期

周期必须分层：**PHY 捕获**、**LPP 消费**、**空口 BFI**、**空口 802.11bf** 四套时钟，禁止假设「1 Hz CSI = 1 Hz CBFR」。

### 3.1 推荐周期表（手机 STA，工程初值）

| 状态 | PHY 捕获（片内） | LPP 推断 | 空口 BFI / sounding 响应 | 802.11bf Report（若有） |
|---|---|---|---|---|
| Idle-static（静置、无业务） | 1–5 Hz 或事件触发 | 1–2 Hz | 跟随 AP 下限；STA 侧粗 Ng、可丢变化不足的反馈 | 阈值上报，周期 200–1000 ms 量级 |
| Traffic-static（在传、相对静） | 与数据 PPDU LTF 同生 | 5–10 Hz | 满足 AP MU/波束；反馈可变粗 | 100–200 ms 或阈值 |
| Moving / 遮挡突发 | 20–50 Hz（受固件上限） | 20 Hz 量级 | 细 BFI、及时响应 NDP | 20–50 ms，EveryInstance |
| Uncertain（CSI 无效） | 维持捕获但打质量位 | **不改决策** | 正常应答 AP，避免被踢 | 不上报或带质量失败 |

本仓库旁证（可行性，非手机 MEASURED）：

- ADR-081：快环 ~200 ms（探测/速率），中环 ~1 s（信道），慢环 ~30 s（基线）。  
- ADR-347：请求探测与**交付** CSI 速率会因争用下降（例：请求 50 Hz、实得约 28–37 Hz）——交付率本身是拥塞计。  
- ADR-153 模型：测量周期合法范围 10 ms～1 h；阈值上报仅在相对变化超过 `delta_percent` 时出报告。

### 3.2 周期由谁决定

```
AP sounding 定时 ──► STA 必须在规范时限内回 CBFR（硬实时）
STA 运动状态机 ──► 调整 Ng / 码本比特 / 是否抑制「重复」BFI（软）
LPP ───────────► 只决定「要不要处理这一份 CSI」和「执行器何时改 OMN/TWT/MCS」
业务 R_app ────► 无流量时禁止为了感知而提高 sounding
```

自适应 sounding 间隔（LPP 建议，固件钳位）：

\[
T_{\mathrm{sound,des}} \propto \frac{1}{\hat{\nu}+\varepsilon}
\]

\(\hat{\nu}\) 为信道变化率（CSI 时域差分或 CIR 主径短时起伏）。必须设上下限，且 **不得违反** AP 当前 MU sounding 合同。

### 3.3 与省电的关系

降低「上报周期」不等于省电。收益从大到小通常是：

1. **TWT**（Target Wake Time，目标唤醒时间）拉长，射频整段关机；  
2. **MIMO PS** 关掉多余 RX 链；  
3. **OMN** 声明更小 BW/NSS，AP 少催反馈；  
4. BFI 变粗、变稀；  
5. LPP 降推断频率。

静止降 sounding 时，MCS **只许持有或略保守**，禁止同时冲最高阶调制。

---

## 4. CSI 必须包含的参数

分三套清单：**片内完整快照（LPP）**、**空口 BFI（AP）**、**LPP 决策输入的最小特征**（带宽不够时降级）。缺元数据的复数矩阵不能用于 MCS/GI/拥塞判决。

### 4.1 片内 CSI 快照（协议栈 → LPP）— 必选

下列字段与本仓库 ADR-018 / `CsiMetadata` / ADR-119 BFI 头、ADR-345 链路归属同构，手机 HAL 应能对上。

#### A. 身份与时间（无则无法做差分和门控）

| 参数 | 说明 |
|---|---|
| `t_capture` | 单调时钟（µs），与 MAC TSF 可对齐的时间戳 |
| `seq` | 捕获序号，检测丢帧/乱序 |
| `link_id` | 本 STA 关联的 BSSID + 本机 MAC 哈希（不要明文外送） |
| `quality_flags` | 首字/前导无效、AGC 饱和、部分子载波损坏（对标 `first_word_invalid` / 消毒位） |

#### B. 射频配置（无则无法解释 \(H\) 的维度）

| 参数 | 说明 |
|---|---|
| `fc_mhz` | 中心频率 |
| `bw_mhz` | 20/40/80/160/320 |
| `puncturing_bitmap` | Wi‑Fi 7 打孔图；无则全 0 |
| `band` | 2.4 / 5 / 6 GHz |
| `ppdu_type` | HT / VHT / HE-SU / HE-MU / HE-TB / EHT（决定子载波网格与导频位置） |
| `gi` 或 `cp_us` | 本 PPDU 保护间隔，便于对照 delay spread |
| `nss_tx`, `nss_rx` | 本帧空间流 / 接收链数 |
| `mcs_used` | **本 PPDU 实际 MCS**（监督速率机、算 PER 对齐） |
| `stbc` / `ldpc` | 编码与 STBC 标志 |

#### C. 能量与 MAC 侧写（无 CSI 也能做拥塞，有则做融合）

| 参数 | 说明 |
|---|---|
| `rssi_dbm[]` | 每天线或合成 RSSI |
| `noise_floor_dbm` | 噪声底 |
| `agc_gain` | 自动增益；幅度特征必须先去 AGC，否则运动检测会被增益跳变污染 |
| `cca_busy_frac` | 近窗口 CCA 忙比例 |
| `retry_cnt` / `ba_fail` | 近窗口重传与块确认失败 |
| `delivered_vs_offered_rate` | 交付速率相对请求（拥塞） |

#### D. 信道本体（LPP 闭环最低需要其一）

**完整模式（预测 MCS / CIR）：**

| 参数 | 说明 |
|---|---|
| \(H[n_{\mathrm{rx}}, n_{\mathrm{tx}}, k]\) | 复数，按子载波 \(k\)；量化 i8/i16 即可，须标明 Q 格式 |
| `k_map` | 子载波逻辑索引（含 DC/导频/打孔后有效音） |
| `pilot_mask` | 导频位置，CIR 求解应排除导频（本仓库 ADR-134） |

**压缩模式（仅运动/遮挡/省电）：**

| 参数 | 说明 |
|---|---|
| 每子载波幅度 \(\|H_k\|\) 或 4～16 bin 的幅度剖面 | 可不传相位 |
| 或 BFI 角 \(\phi/\psi\) + 每流 SNR | 与空口 CBFR 同源，LPP 可直接复用 |

相位在部分廉价前端上公共模式接近随机（本仓库 ADR-345 **MEASURED** 于 ESP32-C6）。手机闭环：**MCS/GI 用幅度与 delay spread；不要用原始相位做 ToF 测距。**

### 4.2 空口 → AP — CBFR/BFI 必选（规范字段）

STA 必须按 802.11 填满，缺了 AP 无法波束成形：

| 参数 | 说明 |
|---|---|
| MIMO 控制 | Nc、Nr、带宽、Ng（子载波分组）、码本信息、反馈类型（SU/MU） |
| 每流 SNR | 压缩报告中的平均 SNR |
| \(\phi/\psi\) 量化角 | Givens 旋转；11ac 与 11ax 比特数不同 |
| Sounding 序列 / 对话 token | 与 NDP 对齐 |
| STA/AP 地址 | 帧头 |

**不要**在 CBFR 里附带私有 CIR 或人体标签。感知走 802.11bf Report 或根本不送 AP。

### 4.3 空口 → AP — 802.11bf 测量报告（硅片支持时）

本仓库 `CsiReportPayload` / `MeasurementSetupParams` 形状可作 HAL 对照：

| 参数 | 说明 |
|---|---|
| setup_id / instance_id | 会话与实例 |
| 带宽、周期 `period_ms`、burst | 协商结果 |
| `ReportingConfig` | EveryInstance 或 ThresholdBased（相对变化门限） |
| 幅度/相位或截断 CIR、PDP | 原生测量类型（ADR-310：不要一律拍扁成 CSI 矩阵） |
| `ConsentMode` | 同意策略；Disabled 则不得开会话 |

SBP：STA 作客户端时，本机 LPP 仍用片内 CSI；AP 回传的是 **有界测量**，不要把代理报告当成「手机测到的原始 \(H\)」。

### 4.4 LPP 最小特征集（建议固定 32 维以内，INT8）

即使不传完整 \(H\)，下列统计足够驱动第 6 节各方案：

1. 子载波 SNR 的 p10 / p50 / p90 与凹坑宽度  
2. \(\mathrm{SNR}_{\mathrm{eff}}\)（EESM 或子载波互信息和）  
3. RMS delay spread、主导径比、有效抽头数（由 CIR 或自相关近似）  
4. AGC 归一化幅度的短时方差（运动）  
5. CCA 忙比、retry EMA  
6. 质量位、IMU 动静、业务水位 \(R_{\mathrm{app}}\)

---

## 5. 如何与 Wi‑Fi 协议栈协同（执行契约）

LPP **不直接改 PHY 寄存器抢包**。只写「建议」；固件在安全点执行。

### 5.1 决策结构体（LPP → Wi‑Fi FW）

```text
LinkOptDecision {
  valid, reason_code,          // 拥塞/干扰/人体/墙/正常/不可靠
  t_expire_us,                 // 过期则 FW 丢弃，防旧决策
  mcs_cap_ul, nss_cap,         // 上行 MCS 上界、流数
  bw_mhz_req, gi_req,          // OMN 带宽、GI 偏好
  puncture_bitmap,             // 可选
  twt_wake_us, rx_chains,      // 省电
  bfi_ng, bfi_bits, bfi_suppress, // 反馈变粗/抑制
  roam_hold, scan_hold,        // 人体期间冻结扫描
  sounding_hint_ms             // 仅提示，AP 合同优先
}
```

`reason_code = Uncertain` 时全部字段 **fail-open**（保持上一配置）。

### 5.2 安全应用点（与 MAC 状态机对齐）

| 动作 | 允许应用的时机 | 禁止 |
|---|---|---|
| 上行 MCS/NSS | 下一 MPDU 聚合开始前、速率机决策点 | PPDU 发送中途 |
| OMN（BW/NSS 声明） | 空闲、无未完成 TXOP，按规范组管理帧 | 每包发 OMN |
| TWT 重协商 | 与 AP TWT 更新窗口 | 随意失约导致掉线 |
| BFI Ng/码本 | 下次 CBFR 组装时 | 改变已在空中的报告 |
| 漫游/扫频 | roam_hold=0 且非人体短时遮挡 | 遮挡时 ACS |
| 热点 ACS | 仅 SoftAP 角色 | 作为 STA 改家用 AP 主信道 |

### 5.3 冲突优先级

1. 监管 / DFS / SAR  
2. AP 要求的强制 sounding 与关联保活  
3. 实时业务（低时延 tid）的 MCS 下限  
4. LPP 的预测 MCS 上界、OMN、TWT  
5. 感知类探测（无业务时最低优先级）

### 5.4 各优化方向的协同矩阵

| 优化方向 | LPP 用 CSI 做什么 | 上报给谁 | 协议栈执行 |
|---|---|---|---|
| 静降/动升 sounding | 运动状态机 | 片内决策；BFI 变粗。周期提示不可压过 AP | Ng/码本/抑制重复 CBFR；TWT；OMN |
| 拥塞检测 | CCA+retry+交付率；CSI 平坦且 RSSI 尚可 | 一般不报 AP | 降 BW、换 BSS、减探测；**不**提高 sounding |
| 干扰检测 | 凹坑形状、频谱扫描、BT PTA | 不把原始谱给 AP | puncturing 请求、换频段、共存 |
| 人体/墙 | CIR 主导径比、\(\sigma_\tau\)、多普勒+IMU | **不**报人体给 AP | 人体：只降 MCS/锁 NSS、roam_hold；墙：长 GI、可漫游 |
| 选信道/频道 | 有效容量打分 | STA 不改 AP 主信道 | 11k/v/r 漫游；OMN 选 BW；SoftAP 才 ACS |
| 预测 MCS/BW/调制/码率 | \(\mathrm{SNR}_{\mathrm{eff}}\) 查表 + CIR→GI | 上行：本机 LA；下行：BFI+OMN 间接约束 AP | 查表设 MCS；先定 BW/GI 再查 MCS |

### 5.5 主机 Wi‑Fi 栈（WPA 超集 / 连接管理器）

- 暴露只读诊断：病因枚举、不确定度、是否 roam_hold。  
- 提供 IMU 与 \(R_{\mathrm{app}}\) 给 LPP。  
- 不订阅原始 CSI。  
- 802.11k 邻区、802.11v BTM、802.11r FT 由连接管理器执行 LPP 的「建议漫游」，并遵守 roam_hold。

---

## 6. 优化方向与方案（汇总）

### 6.1 功耗：人静止降 sounding，人移动升频率

**状态机：** Idle-static / Traffic-static / Moving / Uncertain。入静要数百毫秒～数秒迟滞；入动要快。

**技术：** AGC 归一化 CSI 方差、BFI 角差分、IMU、流量水位；TWT、MIMO PS、OMN、BFI Ng/比特、802.11bf 阈值上报。

### 6.2 链路质量

**拥塞：** CCA、QBSS Load、retry、CW、交付/请求速率。动作：换 AP/频段、OMN 缩带宽、减探测。

**干扰：** CSI 连续凹坑 vs 梳状多径 vs 周期非 Wi‑Fi；ACI/CCI/PTA。动作：puncturing、换频段、BT 共存，而非盲降 MCS。

**人体/墙：** 人体=时变主导径下跌+多普勒；墙=稳定大 delay spread、低 K 因子。人体禁止扫频；墙才考虑漫游与长 GI。不用相位测距。

**信道选择：** STA 主闭环是选频段/选 AP/选本机 BW；热点模式才 ACS。打分用 \(\widehat{C}(\mathrm{CSI})\) 减占用与切换代价，人体期间冻结。

### 6.3 链路预测：CSI 提前调 MCS / 带宽 / 调制 / 码率

CSI 预测的是 **信道能力** \(\widehat{C}(t+\Delta)\)，不是用户要多少 Mbps。执行：

\[
(\mathrm{MCS},BW,GI,N_{ss})=\arg\max \min(\widehat{C},R_{\mathrm{app}})
\]

1. 子载波 SNR → \(\mathrm{SNR}_{\mathrm{eff}}\)（EESM/RBIR）。  
2. CIR：\(\sigma_\tau\) 选 GI；凹坑图选 BW/puncturing。  
3. **先定 BW/GI，再查 MCS 表**（调制+码率在 MCS 编号里）。  
4. 变差立刻降、变好 100–300 ms 再升。  
5. 小模型只分类病因；MCS 数字必须查表。

下行 MCS 在 AP：手机用及时 BFI 与 OMN 约束接收模式，不能由 App 指定。

---

## 7. 参考实现对照（仓库，非手机规格）

| 主题 | 仓库位置 | 手机侧用法 |
|---|---|---|
| CSI 元数据 + I/Q | ADR-018，`CsiMetadata` | LPP 快照字段模板 |
| 无效前缀消毒、真实采样率 | ADR-356 | 质量位、fail-open |
| CIR、delay spread、主导径比 | ADR-134，`cir.rs` | GI/遮挡/墙；LPP 可用更轻的自相关近似 |
| 按链路而非混帧 | ADR-345 | `link_id`；相位慎用 |
| BFI 帧与隐私 | ADR-118/119 | 空口用标准 CBFR；人体标签不出空口 |
| 802.11bf 周期/阈值/SBP | ADR-153/310，`ieee80211bf` | 硅片就绪后的空口测量合同 |
| 自适应环 | ADR-081、309、314 | 快/中/慢环与信息增益，映射 TWT/探测 |
| 原始 RF 不出信任边界 | ADR-277 | LPP 只出有界决策 |

---

## 8. 缩略词表

| 缩略词 | 全称 | 含义 |
|---|---|---|
| Wi‑Fi | Wireless Fidelity | IEEE 802.11 无线局域网 |
| STA | Station | 站点（手机） |
| AP | Access Point | 接入点 |
| SoC | System on Chip | 系统级芯片 |
| LPP | Low-Power Processor | 低功耗核（传感器/常开 DSP） |
| ISR | Interrupt Service Routine | 中断服务例程 |
| IPC | Inter-Process Communication | 进程/核间通信 |
| DMA | Direct Memory Access | 直接内存访问 |
| CSI | Channel State Information | 信道状态信息 \(H(f)\) |
| CFR | Channel Frequency Response | 频域信道响应 |
| CIR | Channel Impulse Response | 信道冲激响应（时延域） |
| PDP | Power Delay Profile | 功率时延谱 |
| NDP | Null Data Packet | 探测用空包 |
| LTF / HE-LTF | (High Efficiency) Long Training Field | 长训练场 |
| PPDU | PHY Protocol Data Unit | 物理层协议数据单元 |
| MPDU | MAC Protocol Data Unit | MAC 帧 |
| TXOP | Transmission Opportunity | 发送机会 |
| BFI | Beamforming Feedback Information | 波束成形反馈 |
| CBFR | Compressed Beamforming Report | 压缩波束成形报告 |
| SVD | Singular Value Decomposition | 奇异值分解 |
| Ng | Grouping | 子载波分组 |
| SU / MU-MIMO | Single/Multi-User MIMO | 单/多用户多天线 |
| MIMO | Multiple Input Multiple Output | 多入多出 |
| NSS | Number of Spatial Streams | 空间流数 |
| MCS | Modulation and Coding Scheme | 调制编码方案 |
| QAM | Quadrature Amplitude Modulation | 正交幅度调制 |
| GI / CP | Guard Interval / Cyclic Prefix | 保护间隔 / 循环前缀 |
| ISI | Inter-Symbol Interference | 符号间干扰 |
| OFDM / OFDMA | Orthogonal Frequency-Division (Multiple Access) | 正交频分（多址） |
| EESM / RBIR | Exponential Effective SINR Mapping / Received Bit Information Rate | 等效 SNR 压缩 |
| SNR / SINR / RSSI | Signal-to-Noise / Interference-plus-Noise / Received Signal Strength | 信噪比 / 信干噪比 / 接收强度 |
| PER | Packet Error Rate | 丢包率 |
| ACK / BA | Acknowledgement / Block Ack | 确认 / 块确认 |
| LA | Link Adaptation | 链路自适应 |
| CCA | Clear Channel Assessment | 空闲信道评估 |
| NAV | Network Allocation Vector | 网络分配矢量 |
| EDCA | Enhanced Distributed Channel Access | 增强分布式接入 |
| CW | Contention Window | 竞争窗口 |
| ACI / CCI | Adjacent / Co-Channel Interference | 邻频 / 同频干扰 |
| PTA | Packet Traffic Arbitration | Wi‑Fi/蓝牙共存仲裁 |
| DFS | Dynamic Frequency Selection | 动态频率选择 |
| SAR | Specific Absorption Rate | 比吸收率 |
| OMN | Operating Mode Notification | 操作模式通知 |
| TWT | Target Wake Time | 目标唤醒时间 |
| MIMO PS | MIMO Power Save | MIMO 省电 |
| WMM-PS | Wi-Fi Multimedia Power Save | 多媒体省电 |
| QBSS | QoS BSS | 带负载信息的 QoS 基本服务集 |
| BSS / BSSID | Basic Service Set / Identifier | 基本服务集及其标识 |
| OBSS | Overlapping BSS | 重叠 BSS |
| BTM | BSS Transition Management | 切换管理（802.11v） |
| FT | Fast BSS Transition | 快速漫游（802.11r） |
| ACS | Automatic Channel Selection | 自动信道选择 |
| IMU | Inertial Measurement Unit | 惯性测量单元 |
| AGC | Automatic Gain Control | 自动增益控制 |
| TSF | Timing Synchronization Function | 定时同步功能 |
| SBP | Sensing by Proxy | 802.11bf 感知代理 |
| HAL | Hardware Abstraction Layer | 硬件抽象层 |
| IE | Information Element | 信息元素 |
| NLOS / LOS | Non- / Line of Sight | 非视距 / 视距 |
| fail-open | — | 测量不可靠时不改配置 |

---

## 9. 文档修订

| 日期 | 说明 |
|---|---|
| 2026-09-16 | 初稿：手机 STA 闭环方向、CSI→AP/LPP 上报周期与必选参数、协议栈协同 |

**证据纪律：** 仓库硬件数字不得直接标为手机芯片 MEASURED。闭环效果以吞吐、P95 时延、重传率、毫安时相对现网 LA/TWT 基线评估。
