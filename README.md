# servo-frame-probe

> **Why separate wire framing from host jitter**: On STM32/CAN-or-UART servo links (from Formula SAE drivetrains to tabletop actuators), dropped frames look identical to USB-serial polling jitter unless sequence-covered CRCs and host receive timestamps are logged cleanly. This repo gives both the 8+N binary frame parser (`CRC-16/CCITT-FALSE` over `seq..payload`) and a reproducible P50/P95/P99 jitter report generator.

**1 kHz 关节伺服回路的实时性与时延分解。**
*1 kHz joint servo loop — jitter and end-to-end latency decomposition.*

这个仓库解决一个具体问题：**你说「跑 1 kHz」，凭什么？**
答案不是"我用了定时器"，而是三张表：周期抖动（P99）、端到端时延的四段分解、以及三种调度配置的对比。

---

## 结果一览 / At a glance

| 项 | 现在 |
| --- | --- |
| 帧格式 | `0xAA55` + seq + cmd + len + payload + CRC-16/CCITT-FALSE；CRC 覆盖 seq，可识别重复/乱序帧 |
| 容错 | 接收端逐字节重同步：一帧坏掉不会废掉整条串流（单测覆盖垃圾前缀与截断尾帧） |
| 测试 | 9 个单测：往返、CRC 失败、非法长度、流式重同步、frozen dataclass |
| 度量 | `tools/jitter_report.py` 从原始 CSV 复算 jitter P50/P90/P99 与周期标准差 |
| 待补 | STM32 1 kHz 双环固件、逻辑分析仪测点、CFS / SCHED_FIFO / PREEMPT_RT 三配置实测、端到端时延四段 |

---

## 目录 / Layout

```
servo-frame-probe/
├── docs/protocol.md          # 帧格式（含 CRC16）与命令表
├── tools/frame.py            # 帧编解码（可离线单测，无需硬件）
├── tools/jitter_report.py    # CSV → P50/P90/P99 抖动表（Markdown）
├── tools/make_sample_csv.py  # 生成示例数据（用于验证工具链）
├── tests/test_frame.py       # 编解码往返 / CRC 失败 / 流式切帧
└── firmware/                 # STM32 固件（待补：1 kHz TIM 中断 + 编码器 + PWM）
```

## 快速开始

```bash
python tools/make_sample_csv.py            # 生成 sample_jitter.csv
python tools/jitter_report.py sample_jitter.csv
python -m pytest tests -q                  # 帧编解码单测
```

## 实验设计（要填的表）

| 配置 | 内核 | 调度 | 期望 jitter P99 | 实测 |
| --- | --- | --- | --- | --- |
| ① | Ubuntu 普通内核 | CFS（默认） | 数百 μs ~ ms | _待填_ |
| ② | Ubuntu 普通内核 | `SCHED_FIFO` + `mlockall` | 几十 μs | _待填_ |
| ③ | PREEMPT_RT 内核 | `SCHED_FIFO` + CPU 隔离 | 个位数 μs | _待填_ |

端到端时延分解（面试必问）：

```
编码器采样 → 总线传输 → 控制计算 → PWM 更新 → 机械响应
  __ μs        __ μs       __ μs       __ μs       __ ms
```

## 状态

- [x] 帧协议 + 编解码（含 CRC16）+ 单测
- [x] jitter 报告工具（CSV → Markdown 表）
- [ ] STM32 固件（1 kHz 位置-速度双环）
- [ ] 逻辑分析仪测点（GPIO 翻转）+ 实测三配置对比
- [ ] 端到端时延分解（四段）

> 数据口径：所有数字由 `tools/jitter_report.py` 从原始 CSV 复算，README 不手填。
> **仓库内的 `sample_jitter.csv` 是 `tools/make_sample_csv.py` 生成的合成数据**，只用于验证工具链；
> 真实三配置对比要等固件上板后用逻辑分析仪采。上面那张表的实测列在拿到真实数据前保持"待填"。

## 相关

- [nav2-mission-fsm](https://github.com/cccjj55/nav2-mission-fsm) — ROS 2 Humble 仓储自主导航与解耦任务状态机
- [sim-claims-ledger](https://github.com/cccjj55/sim-claims-ledger) — 具身仿真吞吐与控制声明核验账本

## License

MIT
