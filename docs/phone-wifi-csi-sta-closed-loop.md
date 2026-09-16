# 手机 Wi‑Fi 芯片 CSI 闭环优化方案

> **落地目标：** 手机 SoC 内的 Wi‑Fi 协议栈、固件、低功耗核（LPP）与连接管理器，以及与 AP 的空口协同。闭环落在固件 / 驱动 / 连接管理器，**不是**商店 App。  
> **仓库用法：** RuView / 802.11bf 协议模型、CSI 帧、CIR、BFLD 等仅作**技术可行性旁证**，不代表已在消费级手机硅片上完成 MEASURED 闭环。  
> **证据纪律：** 文中仓库硬件数字不得直接标为手机芯片 MEASURED。上线须以现网速率机与电量仪对照吞吐、P95 时延、重传率、毫安时。

本文合并两轮整理：

1. **三类优化方向**（功耗 / 链路质量 / 链路预测）：优化目标 → 手机侧旋钮 → 方案与算法 → 和通信栈怎么接。  
2. **CSI 上报与协同：** 从 Wi‑Fi 协议栈报到 AP 或 LPP 的路径、周期、必选参数，以及如何与协议栈一起完成上述优化。

---

## 目录

- [0. 落地边界](#0-落地边界先定谁能改什么)
- [一、Wi‑Fi 功耗优化](#一wi-fi-功耗优化)
- [二、Wi‑Fi 链路质量优化](#二wi-fi-链路质量优化)
- [三、Wi‑Fi 链路预测](#三wi-fi-链路预测)
- [四、手机固件总环](#四推荐的手机固件总环)
- [五、CSI 从协议栈上报到 AP 或 LPP](#五csi-从-wi-fi-协议栈上报到-ap-或-lpp)
- [六、上报周期](#六上报周期)
- [七、CSI 必须包含的参数](#七csi-必须包含的参数)
- [八、与 Wi‑Fi 协议栈的协同契约](#八与-wi-fi-协议栈的协同契约)
- [九、方案 + 技术对照表](#九方案--技术对照表)
- [十、仓库可行性对照](#十仓库可行性对照非手机规格)
- [十一、缩略词表](#十一缩略词表)
- [十二、文档修订](#十二文档修订)

---

## 0. 落地边界（先定谁能改什么）

手机是 **STA**（Station，站点），不是家用路由。

| 决策 | 手机芯片能否闭环 | 典型接口 |
|---|---|---|
| 本机发射 MCS、NSS、带宽、GI | 能（上行速率机 + OMN） | 固件 LA、`Operating Mode` |
| Sounding/BFI 的计算与上报密度 | 能部分做；周期常由 AP 发起，STA 可降精度、合并、阈值上报 | CBFR、Ng、码本比特、802.11bf 阈值 |
| 休眠/唤醒（对功耗最大） | 能 | TWT、WMM-PS、MIMO PS |
| 漫游、选频段、选 AP | 能 | 扫描、802.11k/v/r、band steering |
| 改家里 AP 的主信道 | **不能** | 只能建议或切到别的 BSS |
| SoftAP / 热点 / Wi‑Fi Direct | 能选信道 | 手机当临时 AP 时的 ACS |

因此：

- **功耗**和**预测式 MCS**是手机主场。  
- 「选信道」在 STA 上主要是 **选频段 / 选 AP / 选带宽**，不是改路由器主信道。  
- CSI 有两条合法出口，不能混用：空口给 AP（标准 CBFR / 未来 802.11bf Report）；片内给 LPP（快照或特征，不出应用层）。

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

商店 App **不得**拿原始 CSI。LPP 只输出有界事件（占位/运动/病因/推荐 MCS 上界），由 Wi‑Fi 固件执行。

---

## 一、Wi‑Fi 功耗优化

### 1. 人静止后降低 Sounding 频率，人移动后提高

#### 优化目标

Sounding（信道探测）和 CSI/BFI 处理是手机 Wi‑Fi 里偏贵的一块：射频收 NDP、做 SVD、组 CBFR、再发射。静止时信道的空间结构变化慢，高频探测几乎只烧电；移动时若不加探测，波束和 MCS 会过时，掉包后又重传，电量同样差。

#### 手机侧能拧的旋钮（不只「少探一次」）

按省电收益大致从大到小：

1. **TWT / 休眠**：静止且无流量 → 拉长唤醒间隔，整段射频关机（往往比少做几次 sounding 更省电）。  
2. **MIMO PS / 关接收链**：静止、低吞吐 → 只留 1 条 RX 链；移动或要冲速率再开 2×2。  
3. **OMN**：通知 AP 当前接收 NSS 和带宽（例如静置 20 MHz × 1ss，拿起 80 MHz × 2ss），AP 少做 MU、少催反馈。  
4. **BFI 变稀/变粗**：增大子载波分组 Ng、降低 \(\phi/\psi\) 量化比特、对变化小于门限的反馈不上报。  
5. **本机 CSI 处理降频**：即使 AP 仍在 sounding，DSP/AI 核（LPP）按 1–5 Hz 跑运动检测即可。仓库旁证 ADR-347：边缘 DSP 不必和原始回调同频。

802.11bf 若芯片暴露：**阈值上报**（信道变化不到门限不报）+ 协商测量周期，比「盲周期 sounding」更贴这个方向。

#### 方案与算法

**A. 运动/静止状态机（低功耗，先于大模型）**

输入建议全在芯片里已有或易取：

- CSI 幅度时间方差（AGC 归一化，避免把自动增益当成运动）  
- BFI 角序列的短时差分  
- IMU（加速度/陀螺）作门控：手机自己在动 vs 环境里有人在动  
- 流量：有无 Tid 积压  

输出四态，避免抖动：

| 状态 | 含义 | Sounding/反馈 | 射频策略 |
|---|---|---|---|
| Idle-static | 屏幕外、无业务、信道稳 | 最低：长周期或阈值上报 | 长 TWT、1ss、窄带 |
| Traffic-static | 在看视频但人/机都相对静 | 中等周期，粗 BFI（大 Ng） | 按吞吐开带宽，少 MU |
| Moving | 走路/挥手/环境多普勒高 | 提高 sounding、细 BFI | 开满链、缩短 TWT |
| Uncertain | CSI 不可靠 | **维持上一档**，禁止乱省电 | fail-open |

迟滞：入静止要连续数百毫秒～数秒；入移动要快（几十毫秒），否则第一脚路就掉波束。

**B. 自适应探测周期**

把周期当成对「信道变化率」的跟踪，而不是两个固定值：

\[
T_{\mathrm{sound}} \propto \frac{1}{\hat{\nu} + \varepsilon}
\]

\(\hat{\nu}\) 可用 CSI 的时域差分能量或 CIR 主径的短时多普勒。上限/下限必须钳位（例如静 200–500 ms～数秒，动 20–50 ms），以免违抗 AP 的 MU 调度。

仓库旁证：ADR-081 快环 ~200 ms 调探测、中环 ~1 s 调信道；ADR-314 用「降不确定度 / (算力+电量+空口)」选下一次探测。手机上同一思想，代价改成本机 mA 和 CBFR 空口时间。ADR-288 的吞吐模型也写明 sounding + feedback 会吃掉容量，静时减探测是双赢。

**C. 和链路自适应的互锁**

静止降 sounding 时，MCS 只能 **持有或略保守**，不能同时冲最高阶调制。移动提高 sounding 的目的是给预测式 MCS 提供新鲜 \(H(f)\)，不是为了感知姿态。

#### 和通信栈怎么接

- **Wi‑Fi MAC/PHY 固件：** 运动标志 → TWT 参数、RX 链、OMN、CBFR 抑制。  
- **连接处理器 / LPP：** IMU + 应用流量融合。  
- **不要**把原始 CSI 送 APK。  
- LPP 给出的 `sounding_hint_ms` 只是提示，**AP 的 sounding 合同优先**。

---

## 二、Wi‑Fi 链路质量优化

下列四项在手机上应合成 **一个「链路病因」分类器**，再映射到不同动作。病因搞错会互相拆台（例如把人体遮挡当成脏信道去扫频，又耗电又抖）。

建议统一输出（6 类就够）：

`拥塞 | 同频/邻频干扰 | 非 Wi‑Fi 干扰 | 人体遮挡 | 墙体/远距 | 正常`

---

### 1. 链路拥塞检测

#### 优化目标

区分「频谱被别的 BSS 占满」和「本链路 SNR 差」。拥塞时降 MCS、加探测都没用，只会更挤。

#### 方案与算法

手机侧特征（多数驱动已有，不必等 CSI）：

| 特征 | 含义 |
|---|---|
| CCA 忙时间 / 信道利用率 | 空口被能量占用的比例 |
| 信标 QBSS Load / BSS 负载 IE | AP 报的占用和关联数 |
| 本机 retry、碰撞、CW 变大 | 争用而非衰减 |
| 交付速率 vs 请求速率 | 仓库旁证 ADR-347：探针 50 Hz、实得 28–37 Hz 即争用 |
| NAV / OBSS 检测 | 别的 802.11 网络在说话 |
| EDCA 队列堆积但 RSSI 仍好 | 典型拥塞 |

CSI 补充：子载波 SNR **整体尚可且较平**，但 MAC 层排队——更像拥塞而不是深衰落。

#### 手机侧旋钮与动作

- **不要**为拥塞提高 sounding（会更堵）。  
- 切到更空的 **频段/AP**（5/6 GHz、邻 BSS 利用率低的 BSSID）。  
- 收窄本机 Tx/Rx 带宽（OMN 降到 40/80），减少与邻区重叠。  
- 拉长 AIFS/TXOP 不现实；可降本机背景扫描、暂停非必要探测。  
- TWT 对齐到 AP 空窗（若 AP 支持）。  
- 应用层：提示「当前网络拥挤」，避免误导成信号差。

---

### 2. 链路干扰检测

#### 优化目标

把干扰从「SNR 低」里拆出来：同频 Wi‑Fi、邻频泄漏、非 Wi‑Fi（微波炉、蓝牙、USB3）。动作完全不同。

#### 方案与算法

**频域 CSI 形状（主特征）**

- **宽带平坦下跌**：距离/墙/人体，不像窄带干扰。  
- **连续一块 20 MHz 凹坑**：邻 BSS 重叠或雷达/Wi‑Fi 6E 打孔候选。  
- **梳状、多深坑**：频率选择性多径（障碍物），不是单频干扰。  
- **窄凹坑 + 时间周期性（约 50/60 Hz 包络）**：微波炉类非 Wi‑Fi（**CLAIMED** 级经验，需芯片频谱扫描交叉验证）。

**时间/MAC**

- CRC 错但 CCA 显示「非 Wi‑Fi 能量」（部分芯片有 spectral scan / FFT 扫描）。  
- 蓝牙共存计数：BT 高占空时 Wi‑Fi 错包，CSI 未必有固定凹坑。

仓库旁证：ADR-073 金属体造成约 19% 子载波长期为空——说明「凹坑图」可作稳定指纹；通信侧用来标 **puncturing 候选** 或避开该 20 MHz 子信道。

#### 手机侧旋钮与动作

| 干扰类型 | 手机闭环 |
|---|---|
| 邻区重叠（CCI） | 漫游到非重叠信道的 AP；OMN 缩小带宽；Wi‑Fi 7 请求 puncturing |
| 邻频（ACI） | 降本机发射频谱边带（降带宽/功率）；避开贴边信道 |
| 非 Wi‑Fi | 换频段（2.4→5/6）；微波炉场景避免 2.4 GHz 高 MCS |
| BT 共存 | 固件 PTA：时分/跳频协同，而不是改 MCS 猛降 |

手机 **改不了** 家用 AP 主信道；能做的是 **换 BSS、换频段、改本机占用宽度**。原始频谱不要上报给 AP。

---

### 3. 人体、墙等障碍物检测

#### 优化目标

遮挡会掉主径、催生反射，表现为突发 PER。若当成「信道变脏」去切 AP/切频段，人一走又切回来。正确做法：短时保守 MCS、必要时锁 1ss，**不触发漫游/ACS**。

#### 方案与算法（CSI/CIR + IMU）

从 CSI 恢复 CIR（仓库 ADR-134：ISTA 稀疏 CIR，产出 RMS delay spread、主导径比、有效抽头数）。手机 20/40/80 MHz 都能算这些量；**20 MHz 不够测距，够分类**。LPP 上也可用更轻的自相关近似，不必每包跑完整 ISTA。

| | 人体（动态） | 墙/家具（静态） |
|---|---|---|
| 多普勒 / 幅度短时方差 | 高 | 低 |
| 主导径比 | 突然下降，可恢复 | 长期偏低 |
| Delay spread | 中等、时变 | 偏大、稳定 |
| IMU | 可区分「手机在动」 | 无关 |
| 时间尺度 | 0.2–3 s | 分钟～小时 |

K 因子（直射/散射功率比）低且稳定 → 隔墙或远距 NLOS。  
K 因子骤降 + 运动头 → 人挡在 Fresnel 区。

仓库旁证：ADR-345 **MEASURED** 廉价芯片上公共相位接近随机，手机闭环 **不要用 ToF/相位测距**；用幅度、delay spread、主导径比。人体门控与 ACS 解耦。

**不要**把人体标签通过 CBFR 送给 AP。

#### 手机侧旋钮与动作

**人体、短时：**

- 预测式降 MCS、锁 1ss、短时可改用更稳的接收波束（若 STA 有多天线选择）  
- **禁止** 扫频、漫游、改 AP（`roam_hold` / `scan_hold`）  
- 可轻微升 TX 功率（受 SAR/法规上限）  

**墙/远距、长期：**

- 优先漫游到更近 AP；5 GHz 若穿透差则评估 2.4 的稳 vs 5 的快  
- OMN 降带宽、长 GI  
- 对用户：「隔墙，建议靠近路由或用 Mesh」——20 MHz 不要报米级距离  

---

### 4. 信道、频道选择

「信道/频道」在手机上拆成三层，避免写成「STA 改家庭 AP 主信道」。

#### 4.1 选频段与选 AP（主闭环）

扫描候选 BSSID，对每个候选估 **有效容量** 而不只比 RSSI：

\[
\mathrm{Score} = \widehat{C}(\mathrm{CSI}) - \lambda_1 U_{\mathrm{CCA}} - \lambda_2 \mathbf{1}_{\mathrm{人体遮挡}} - \lambda_3 \mathrm{SwitchCost}
\]

- \(\widehat{C}\)：子载波 SNR 的互信息和/等效 SNR  
- \(U_{\mathrm{CCA}}\)：该信道利用率  
- 人体遮挡期间 **冻结** 漫游（SwitchCost 极大）  
- 802.11k 邻区报告 + 802.11v BTM 减少盲扫（省电）  
- 802.11r 让切 AP 的中断变短  

#### 4.2 选本机工作带宽（OMN）

同一 AP 上：80/160 是否值得，看 CSI 凹坑和 delay spread（与预测式带宽合并）。这是手机 **立刻能发帧通知 AP** 的闭环。

#### 4.3 仅当手机是 SoftAP/热点

此时手机才做 ACS：用 CSI+CCA 给 20 MHz 主信道打分，避开高占用和深凹坑。这是唯一「手机决定主信道」的场景。

仓库旁证：ADR-081 中环用 CSI 方差、RSSI、占用做 `set_channel`——逻辑可搬到热点模式；关联他人 AP 时只能搬「打分+漫游」部分。

---

## 三、Wi‑Fi 链路预测

### 1. 用 CSI 预测链路能力，提前调 MCS、带宽、调制、码率

先纠正产品表述：**CSI 预测的是「信道此刻能支撑的速率」**，不是应用层「用户需要多少 Mbps」。两者要分开：

| | 预测对象 | 用途 |
|---|---|---|
| 信道容量 \(\widehat{C}(t+\Delta)\) | PHY 能可靠传多快 | 提前设 MCS/带宽/GI/NSS |
| 业务需求 \(R_{\mathrm{app}}\) | 播放器码率、下载窗口 | 决定要不要冲满 PHY、能不能 TWT 睡觉 |

闭环应是：

\[
\mathrm{MCS,BW,GI,N_{ss}} = \arg\max\ \min(\widehat{C}, R_{\mathrm{app}})
\]

且满足 PER 目标。无大流量时即使 \(\widehat{C}\) 很高也不开 160 MHz×2ss（省电）。

#### 现网 vs 预测式

现网 LA 看 ACK/PER，是事后的。预测式用本包前导或最近 NDP/BFI 的 \(H(f)\)，在 ACK 前给出 **上界**。

手机上两条速率环：

1. **上行（手机发）：** 固件速率机直接改 Tx MCS/NSS/BW——闭环最干净。  
2. **下行（AP 发）：** MCS 在 AP。手机通过 **更及时/更准的 BFI、BA 统计、OMN 声明的接收带宽和 NSS** 间接约束 AP。把「预测」写进 CBFR 质量与 OMN，而不是由 App 指定下行 MCS。

#### 方案与算法（先物理量，再小模型）

**步骤 1 — 频域有效 SNR**

\[
\mathrm{SNR}_k \propto |H(f_k)|^2,\quad
\mathrm{SNR}_{\mathrm{eff}} = \mathrm{EESM}\text{ 或 }\sum_k \log_2(1+\mathrm{SNR}_k)
\]

深衰落子载波会显著拉低 \(\mathrm{SNR}_{\mathrm{eff}}\)（不是 RSSI 那种算术平均）。

**步骤 2 — 时延域约束带宽和 GI（多径 × 带宽）**

OFDM 符号后的 GI/CP 必须长于信道时延扩展，否则 ISI。相干带宽大约是 delay spread 的倒数：\(\sigma_\tau\) 大时 80/160 MHz 里一定有深坑，名义峰值速率可能低于更窄、更稳的配置。

| CIR | 动作 |
|---|---|
| \(\sigma_\tau\) 小、主导径比高 | 短 GI、允许 80/160 |
| \(\sigma_\tau\) 大 | 长 GI（11n/ac 800 ns；11ax 1.6/3.2 µs），禁止短 GI 冲峰值 |
| 凹坑集中在某 20 MHz | 11be puncturing 或 OMN 降到 80/40 |
| 子载波方差大、梳状衰落 | 先缩带宽再查 MCS，避免在 160 MHz 上算出一个「低 MCS 却仍 ISI」 |

顺序必须是 **先定带宽和 GI，再查 MCS**。否则会出现：在 160 MHz 短 GI 上算出很低 MCS，看起来预测对了，其实换 80 MHz 长 GI 就能用高两档 MCS。

**步骤 3 — 查表得到调制和码率**

MCS 本来就是「调制 + 码率 + NSS」的编号，例如 64-QAM 5/6、256-QAM 3/4。应由 \(\mathrm{SNR}_{\mathrm{eff}}\) + GI/BW 查标准表，**不要让网络直接吐 MCS 号**。小模型只负责：人体 vs 干扰 vs 拥塞（决定能不能升速、能不能漫游）。

**步骤 4 — 升降不对称**

- 预测变差：立即降（保时延、少 retry，也省因重传浪费的电）  
- 预测变好：延迟 100–300 ms 再升（人体晃动）  
- CSI 质量门控（首字无效、速率突变）：维持原 MCS。仓库旁证 ADR-356：不可靠样本 fail-open。

#### 和功耗方向的衔接

- \(R_{\mathrm{app}}\) 低且静止 → 即使 \(\widehat{C}\) 高也 OMN 降 BW/NSS，并降 sounding。  
- \(R_{\mathrm{app}}\) 高且开始移动 → 先加密 sounding，再按新 CSI 升 MCS，避免「睡着时用旧的高 MCS 打第一下」。

#### 和通信栈怎么接

- 上行：LPP 写 `mcs_cap_ul` / `nss_cap`，固件在下一聚合开始前交给 LA。  
- 下行：提高/变粗 BFI、发 OMN，不幻想 App 指定 MCS。  
- **本仓库没有手机芯片上的 MCS 闭环 MEASURED 数据**；对照基线必须是现网 Minstrel/厂商 LA。

---

## 四、推荐的手机固件总环

把三类方向收成一条流水线：

```
每包/LTF:  子载波 SNR、CCA、retry
10–50ms:   运动/遮挡标志（CSI方差 + IMU）
50–200ms:  病因分类（拥塞/干扰/人体/墙）
           ├─ 人体     → 只改 MCS/NSS，冻结漫游与 sounding 以外的扫描
           ├─ 拥塞     → 降 BW、换 AP/频段，降低 sounding
           ├─ 干扰     → puncturing/换频段/BT PTA
           └─ 墙/远距  → 长 GI、保守 MCS、考虑漫游
同时:      SNR_eff → 上行 MCS；OMN → BW/NSS
静止:      TWT↑、BFI 变粗、链数↓
移动+有业务: sounding↑，预测 MCS 跟手
```

信息增益思想（ADR-314）：没有业务且静止时，**整段 Wi‑Fi 可睡**，比「仍以 10 Hz sounding 做感知」更符合手机电量。

---

## 五、CSI 从 Wi‑Fi 协议栈上报到 AP 或 LPP

### 5.1 路径 A：协议栈 → AP（空口）

这是 802.11ac/ax/be **已经在发**的波束成形反馈，不是新发明。

1. AP 发 **NDP** 或数据前导中的 **LTF / HE-LTF**。  
2. STA PHY 估计信道矩阵 \(H(f)\)。  
3. STA 做 **SVD**，把右奇异向量压成 Givens 角 \(\phi/\psi\)。  
4. STA 发 **CBFR**（即 BFI）。

约束：

- CBFR 是管理/动作帧，格式、Ng、码本比特由能力协商决定；STA **不能**把任意私有 CSI blob 塞进标准 CBFR。  
- 上报周期主要由 **AP 的 sounding 调度**决定；STA 可做的是：降精度、合并、阈值不上报、用 OMN 声明更窄的接收模式以减少被探测。  
- **802.11bf-2025** 增加独立的 Sensing Measurement Setup / Instance / Report，以及 **SBP**（STA 请 AP 代测再回传有界报告）。消费级硅片暴露该接口前，空口仍以 CBFR 为主。本仓库 ADR-153/310 对上述过程做了类型化模型（非认证实现）。

### 5.2 路径 B：协议栈 → LPP（片内）

LPP 指手机上常开或浅睡的低功耗核（各家商品名不同：Sensor Hub / SLPI / CDSP / Tiny 核），用来跑轻量分类，避免每次 CSI 都唤醒应用处理器。

推荐数据面（与本仓库 ADR-356「CSI 回调不得在 Wi‑Fi 任务里做重计算」同构）：

1. **Wi‑Fi ISR / Wi‑Fi 任务：** 只做合法性检查、拷贝或 DMA 到环形缓冲、置位 mailbox。禁止在此做 ISTA/CIR/NN。  
2. **LPP：** 消费环形缓冲，算特征与状态机，写回「决策结构体」。  
3. **Wi‑Fi 固件执行器：** 在下一个 TXOP / 下一次 OMN/TWT 更新点应用决策，**不得**与正在发送的 PPDU 中途改 MCS/BW。

### 5.3 双路径协调原则

| 原则 | 说明 |
|---|---|
| 一份 CSI，两套视图 | PHY 只估一次 \(H\)；空口走标准 BFI；LPP 走内部快照或压缩特征。 |
| 空口优先保证连通 | LPP 建议与 AP 调度冲突时，**连通与 MU 反馈时限优先**。 |
| 原始 \(H\) 不出 LPP | 出核只有特征或有界事件；CBFR 按 802.11 规范，不另附私有 CSI。 |
| 人因与通信解耦 | 「人体遮挡」只改本机 MCS/NSS；不把人体当脏信道去催 AP 切主信道。 |

---

## 六、上报周期

周期必须分层：**PHY 捕获**、**LPP 消费**、**空口 BFI**、**空口 802.11bf** 四套时钟，禁止假设「1 Hz CSI = 1 Hz CBFR」。

### 6.1 推荐周期表（手机 STA，工程初值）

| 状态 | PHY 捕获（片内） | LPP 推断 | 空口 BFI / sounding 响应 | 802.11bf Report（若有） |
|---|---|---|---|---|
| Idle-static（静置、无业务） | 1–5 Hz 或事件触发 | 1–2 Hz | 跟随 AP 下限；STA 侧粗 Ng、可丢变化不足的反馈 | 阈值上报，周期 200–1000 ms 量级 |
| Traffic-static（在传、相对静） | 与数据 PPDU LTF 同生 | 5–10 Hz | 满足 AP MU/波束；反馈可变粗 | 100–200 ms 或阈值 |
| Moving / 遮挡突发 | 20–50 Hz（受固件上限） | 20 Hz 量级 | 细 BFI、及时响应 NDP | 20–50 ms，EveryInstance |
| Uncertain（CSI 无效） | 维持捕获但打质量位 | **不改决策** | 正常应答 AP，避免被踢 | 不上报或带质量失败 |

仓库旁证（可行性，非手机 MEASURED）：

- ADR-081：快环 ~200 ms（探测/速率），中环 ~1 s（信道），慢环 ~30 s（基线）。  
- ADR-347：请求探测与**交付** CSI 速率会因争用下降（例：请求 50 Hz、实得约 28–37 Hz）——交付率本身是拥塞计。  
- ADR-153 模型：测量周期合法范围 10 ms～1 h；阈值上报仅在相对变化超过 `delta_percent` 时出报告。

### 6.2 周期由谁决定

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

必须设上下限，且 **不得违反** AP 当前 MU sounding 合同。

### 6.3 与省电的关系

降低「上报周期」不等于省电。收益从大到小通常是：

1. **TWT** 拉长，射频整段关机；  
2. **MIMO PS** 关掉多余 RX 链；  
3. **OMN** 声明更小 BW/NSS，AP 少催反馈；  
4. BFI 变粗、变稀；  
5. LPP 降推断频率。

静止降 sounding 时，MCS **只许持有或略保守**。

---

## 七、CSI 必须包含的参数

分三套清单：**片内完整快照（LPP）**、**空口 BFI（AP）**、**LPP 决策输入的最小特征**。缺元数据的复数矩阵不能用于 MCS/GI/拥塞判决。

### 7.1 片内 CSI 快照（协议栈 → LPP）— 必选

字段与本仓库 ADR-018 / `CsiMetadata` / ADR-119 BFI 头、ADR-345 链路归属同构，手机 HAL 应能对上。

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
| `agc_gain` | 自动增益；幅度特征必须先去 AGC |
| `cca_busy_frac` | 近窗口 CCA 忙比例 |
| `retry_cnt` / `ba_fail` | 近窗口重传与块确认失败 |
| `delivered_vs_offered_rate` | 交付速率相对请求（拥塞） |

#### D. 信道本体（LPP 闭环最低需要其一）

**完整模式（预测 MCS / CIR）：**

| 参数 | 说明 |
|---|---|
| \(H[n_{\mathrm{rx}}, n_{\mathrm{tx}}, k]\) | 复数，按子载波 \(k\)；量化 i8/i16 即可，须标明 Q 格式 |
| `k_map` | 子载波逻辑索引（含 DC/导频/打孔后有效音） |
| `pilot_mask` | 导频位置，CIR 求解应排除导频（ADR-134） |

**压缩模式（仅运动/遮挡/省电）：**

| 参数 | 说明 |
|---|---|
| 每子载波幅度 \(\|H_k\|\) 或 4～16 bin 的幅度剖面 | 可不传相位 |
| 或 BFI 角 \(\phi/\psi\) + 每流 SNR | 与空口 CBFR 同源，LPP 可直接复用 |

相位在部分廉价前端上公共模式接近随机（ADR-345 **MEASURED** 于 ESP32-C6）。手机闭环：**MCS/GI 用幅度与 delay spread；不要用原始相位做 ToF 测距。**

### 7.2 空口 → AP — CBFR/BFI 必选（规范字段）

STA 必须按 802.11 填满，缺了 AP 无法波束成形：

| 参数 | 说明 |
|---|---|
| MIMO 控制 | Nc、Nr、带宽、Ng（子载波分组）、码本信息、反馈类型（SU/MU） |
| 每流 SNR | 压缩报告中的平均 SNR |
| \(\phi/\psi\) 量化角 | Givens 旋转；11ac 与 11ax 比特数不同 |
| Sounding 序列 / 对话 token | 与 NDP 对齐 |
| STA/AP 地址 | 帧头 |

**不要**在 CBFR 里附带私有 CIR 或人体标签。感知走 802.11bf Report 或根本不送 AP。

### 7.3 空口 → AP — 802.11bf 测量报告（硅片支持时）

本仓库 `CsiReportPayload` / `MeasurementSetupParams` 形状可作 HAL 对照：

| 参数 | 说明 |
|---|---|
| setup_id / instance_id | 会话与实例 |
| 带宽、周期 `period_ms`、burst | 协商结果 |
| `ReportingConfig` | EveryInstance 或 ThresholdBased（相对变化门限） |
| 幅度/相位或截断 CIR、PDP | 原生测量类型（ADR-310：不要一律拍扁成 CSI 矩阵） |
| `ConsentMode` | 同意策略；Disabled 则不得开会话 |

SBP：STA 作客户端时，本机 LPP 仍用片内 CSI；AP 回传的是 **有界测量**，不要把代理报告当成「手机测到的原始 \(H\)」。

### 7.4 LPP 最小特征集（建议固定 32 维以内，INT8）

即使不传完整 \(H\)，下列统计足够驱动第一～三节方案：

1. 子载波 SNR 的 p10 / p50 / p90 与凹坑宽度  
2. \(\mathrm{SNR}_{\mathrm{eff}}\)（EESM 或子载波互信息和）  
3. RMS delay spread、主导径比、有效抽头数（由 CIR 或自相关近似）  
4. AGC 归一化幅度的短时方差（运动）  
5. CCA 忙比、retry EMA  
6. 质量位、IMU 动静、业务水位 \(R_{\mathrm{app}}\)

---

## 八、与 Wi‑Fi 协议栈的协同契约

LPP **不直接改 PHY 寄存器抢包**。只写「建议」；固件在安全点执行。

### 8.1 决策结构体（LPP → Wi‑Fi FW）

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

### 8.2 安全应用点（与 MAC 状态机对齐）

| 动作 | 允许应用的时机 | 禁止 |
|---|---|---|
| 上行 MCS/NSS | 下一 MPDU 聚合开始前、速率机决策点 | PPDU 发送中途 |
| OMN（BW/NSS 声明） | 空闲、无未完成 TXOP，按规范组管理帧 | 每包发 OMN |
| TWT 重协商 | 与 AP TWT 更新窗口 | 随意失约导致掉线 |
| BFI Ng/码本 | 下次 CBFR 组装时 | 改变已在空中的报告 |
| 漫游/扫频 | roam_hold=0 且非人体短时遮挡 | 遮挡时 ACS |
| 热点 ACS | 仅 SoftAP 角色 | 作为 STA 改家用 AP 主信道 |

### 8.3 冲突优先级

1. 监管 / DFS / SAR  
2. AP 要求的强制 sounding 与关联保活  
3. 实时业务（低时延 tid）的 MCS 下限  
4. LPP 的预测 MCS 上界、OMN、TWT  
5. 感知类探测（无业务时最低优先级）

### 8.4 各优化方向如何靠上报完成

| 优化方向 | LPP 用 CSI 做什么 | 上报给谁 | 协议栈执行 |
|---|---|---|---|
| 静降/动升 sounding | 运动状态机 | 片内决策；BFI 变粗。周期提示不可压过 AP | Ng/码本/抑制重复 CBFR；TWT；OMN |
| 拥塞检测 | CCA+retry+交付率；CSI 平坦且 RSSI 尚可 | 一般不报 AP | 降 BW、换 BSS、减探测；**不**提高 sounding |
| 干扰检测 | 凹坑形状、频谱扫描、BT PTA | 不把原始谱给 AP | puncturing 请求、换频段、共存 |
| 人体/墙 | CIR 主导径比、\(\sigma_\tau\)、多普勒+IMU | **不**报人体给 AP | 人体：只降 MCS/锁 NSS、roam_hold；墙：长 GI、可漫游 |
| 选信道/频道 | 有效容量打分 | STA 不改 AP 主信道 | 11k/v/r 漫游；OMN 选 BW；SoftAP 才 ACS |
| 预测 MCS/BW/调制/码率 | \(\mathrm{SNR}_{\mathrm{eff}}\) 查表 + CIR→GI | 上行：本机 LA；下行：BFI+OMN 间接约束 AP | 查表设 MCS；先定 BW/GI 再查 MCS |

### 8.5 主机 Wi‑Fi 栈（连接管理器）

- 暴露只读诊断：病因枚举、不确定度、是否 roam_hold。  
- 提供 IMU 与 \(R_{\mathrm{app}}\) 给 LPP。  
- 不订阅原始 CSI。  
- 802.11k 邻区、802.11v BTM、802.11r FT 由连接管理器执行 LPP 的「建议漫游」，并遵守 roam_hold。

---

## 九、方案 + 技术对照表

| 方向 | 核心技术 | 手机闭环执行器 | 注意 |
|---|---|---|---|
| 静降 / 动升 sounding | 运动状态机、阈值 BFI、TWT、Ng/码本、802.11bf 阈值上报 | 固件探测与休眠 | 降探测时 MCS 不得冒进 |
| 拥塞检测 | CCA、QBSS、retry、交付/请求速率 | 换 AP/频段、降 BW、减探测 | 勿当 SNR 问题猛降 MCS 仍挤在原信道 |
| 干扰检测 | CSI 凹坑形状、频谱扫描、BT PTA | puncturing、换频段、共存 | 与多径梳状衰落分型 |
| 人体/墙检测 | CIR 主导径比、\(\sigma_\tau\)、多普勒、IMU | 人体：只调 MCS；墙：漫游/长 GI | 禁用相位测距；人体冻结 ACS |
| 信道/频道选择 | 有效容量打分、11k/v/r、OMN 带宽 | 选 AP/频段/BW；热点才 ACS | STA 改不了家用 AP 主信道 |
| CSI 预测 MCS/带宽/调制/码率 | \(\mathrm{SNR}_{\mathrm{eff}}\) 查表、CIR→GI/BW、升降迟滞 | 上行 LA + 下行 BFI/OMN | 预测的是信道能力，需再和业务码率取 min |

---

## 十、仓库可行性对照（非手机规格）

| 主题 | 仓库位置 | 手机侧用法 |
|---|---|---|
| CSI 元数据 + I/Q | ADR-018，`CsiMetadata` | LPP 快照字段模板 |
| 无效前缀消毒、真实采样率 | ADR-356 | 质量位、fail-open |
| CIR、delay spread、主导径比 | ADR-134，`cir.rs` | GI/遮挡/墙；LPP 可用更轻的自相关近似 |
| 按链路而非混帧；相位慎用 | ADR-345 | `link_id`；不用 ToF 测距 |
| BFI 帧与隐私 | ADR-118/119 | 空口用标准 CBFR；人体标签不出空口 |
| 802.11bf 周期/阈值/SBP | ADR-153/310，`ieee80211bf` | 硅片就绪后的空口测量合同 |
| 自适应环、信息增益 | ADR-081、309、314 | 快/中/慢环映射 TWT/探测 |
| sounding 吃空口 | ADR-288 | 静时减探测是容量与电量双赢 |
| 子载波长期空坑 | ADR-073 | puncturing / 避开子信道 |
| 原始 RF 不出信任边界 | ADR-277 | LPP 只出有界决策；`ChannelDiagnostics` 可作事件名 |

---

## 十一、缩略词表

| 缩略词 | 全称 | 含义 |
|---|---|---|
| Wi‑Fi | Wireless Fidelity | 基于 IEEE 802.11 的无线局域网 |
| STA | Station | 站点，手机端角色 |
| AP | Access Point | 接入点，路由器/热点 |
| SoC | System on Chip | 系统级芯片 |
| LPP | Low-Power Processor | 低功耗核（传感器/常开 DSP） |
| ISR | Interrupt Service Routine | 中断服务例程 |
| IPC | Inter-Process Communication | 进程/核间通信 |
| DMA | Direct Memory Access | 直接内存访问 |
| CSI | Channel State Information | 各子载波复数信道响应 \(H(f)\) |
| CFR | Channel Frequency Response | 与 CSI 频域形式同类 |
| CIR | Channel Impulse Response | 时延域多径（由 CSI 反演） |
| PDP | Power Delay Profile | 各时延抽头的功率分布 |
| NDP | Null Data Packet | 无数据载荷的探测包，用于测信道 |
| LTF / HE-LTF | (High Efficiency) Long Training Field | 前导中的长训练场，CSI 多从此估 |
| PPDU | PHY Protocol Data Unit | 物理层协议数据单元 |
| MPDU | MAC Protocol Data Unit | MAC 帧 |
| TXOP | Transmission Opportunity | 发送机会 |
| BFI / CBFR | Beamforming Feedback / Compressed Beamforming Report | 手机把信道压成 Givens 角 \(\phi/\psi\) 回给 AP |
| SVD | Singular Value Decomposition | 分解信道矩阵以得到波束方向 |
| Ng | Grouping | 子载波分组 |
| SU / MU-MIMO | Single/Multi-User MIMO | 单用户/多用户多天线 |
| MIMO | Multiple Input Multiple Output | 多入多出 |
| NSS | Number of Spatial Streams | 空间流数 |
| MCS | Modulation and Coding Scheme | 调制与编码方案编号（含阶数与码率） |
| QAM | Quadrature Amplitude Modulation | 正交幅度调制（16/64/256/1024-QAM） |
| QPSK / BPSK | Quad/Binary PSK | 低阶相移键控 |
| GI / CP | Guard Interval / Cyclic Prefix | 保护间隔/循环前缀，抗多径 ISI |
| ISI | Inter-Symbol Interference | 符号间干扰 |
| OFDM / OFDMA | Orthogonal Frequency Division (Multiple Access) | 正交频分（多址） |
| RU | Resource Unit | OFDMA 资源单元 |
| EESM / RBIR | Exponential Effective SINR Mapping / Received Bit Information Rate | 把一串子载波 SNR 压成等效 SNR 的方法 |
| SNR / SINR / RSSI | Signal to (Interference plus) Noise Ratio / Received Signal Strength Indicator | 信噪比、信干噪比、接收强度 |
| PER / BER | Packet/Bit Error Rate | 包/误码率 |
| ACK / BA | Acknowledgement / Block Ack | 确认/块确认 |
| LA | Link Adaptation | 链路自适应（选 MCS 等） |
| CCA | Clear Channel Assessment | 空闲信道评估（听信道忙不忙） |
| NAV | Network Allocation Vector | 根据别人报的时长推迟发送 |
| EDCA | Enhanced Distributed Channel Access | 增强型分布式信道接入 |
| CW | Contention Window | 竞争窗口 |
| AIFS | Arbitration Inter-Frame Space | 仲裁帧间隔 |
| ACI / CCI | Adjacent/Co-Channel Interference | 邻频/同频干扰 |
| PTA | Packet Traffic Arbitration | Wi‑Fi/蓝牙共存仲裁 |
| DFS | Dynamic Frequency Selection | 动态选频（避雷达） |
| OMN | Operating Mode Notification | 操作模式通知（告知 AP 当前 BW/NSS） |
| TWT | Target Wake Time | 目标唤醒时间（802.11ax 省电） |
| WMM-PS | Wi-Fi Multimedia Power Save | 多媒体省电 |
| MIMO PS | MIMO Power Save | 关多余接收链省电 |
| DTIM | Delivery Traffic Indication Map | 投递流量指示，影响休眠醒来 |
| IE | Information Element | 管理帧里的信息元素 |
| QBSS | QoS BSS | 带负载等信息的 QoS BSS |
| BSS / BSSID | Basic Service Set / Identifier | 基本服务集及其标识 |
| OBSS | Overlapping BSS | 重叠的邻小区 |
| BTM | BSS Transition Management | 802.11v 引导切换 AP |
| FT | Fast BSS Transition | 802.11r 快速漫游 |
| ACS | Automatic Channel Selection | 自动信道选择 |
| Puncturing | Preamble Puncturing | Wi‑Fi 6/7：挖掉被干扰的 20 MHz 子信道仍用宽频 |
| NLOS / LOS | Non-/Line of Sight | 非/视距 |
| Doppler | — | 多普勒，反映相对运动 |
| AGC | Automatic Gain Control | 自动增益控制 |
| IMU | Inertial Measurement Unit | 惯性测量（加速计/陀螺） |
| SAR | Specific Absorption Rate | 比吸收率，限制近体发射功率 |
| TSF | Timing Synchronization Function | 定时同步功能 |
| SBP | Sensing by Proxy | 802.11bf：STA 请 AP 代为感知 |
| HAL | Hardware Abstraction Layer | 硬件抽象层 |
| 802.11k/v/r | IEEE 修正案 | 测量/切换管理/快速漫游 |
| 802.11ax/be/bf | Wi‑Fi 6 / 7 / WLAN Sensing | 高效无线、极高吞吐、标准化感知 |
| DSP | Digital Signal Processor | 数字信号处理器 |
| fail-open | — | 测量不可靠时不改配置，保持上一策略 |

---

## 十二、文档修订

| 日期 | 说明 |
|---|---|
| 2026-09-16 | 初稿：偏 CSI→AP/LPP 上报，三类优化方向仅作摘要 |
| 2026-09-16 | 合并两轮对话：补全功耗/质量/预测全文，保留上报周期、必选参数与协议栈协同 |

**评估方式：** 闭环效果以吞吐、P95 时延、重传率、毫安时相对现网 LA/TWT 基线评估，不用感知 PCK。
