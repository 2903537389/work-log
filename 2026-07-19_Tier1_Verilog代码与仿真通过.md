# 工作日志 - 2026年7月19日

## 智能仿核信号发生器 · Tier 1 完成（实际时间：07-18 21:20 ~ 07-19 01:43 北京时间）

今天完成了 FPGA 项目从零到 Tier 1 全部通过仿真的关键一天。

### 硬件方案彻底澄清

- 确认 BX71 ZYNQ 核心板自带网口 / HDMI / SD 卡槽 / 电源，无需底板设计
- ACM9767 DAC 模块通过 36 针排针直插核心板 P2 头（DE2 兼容布局），Pin1 对 Pin1
- 完整梳理了 ACM9767 与 BX71 P2 排针的每一根线映射，写入 `docs/pin_map.md`
- 关键接口：CLK1/WRT1 → D19/D20，CLK2/WRT2 → E18/E19，14bit 数据 P1_DB[13:0] 和 P2_DB[13:0] 分别映射到 P2 排针的两组 GPIO
- 结论：无需任何 PCB 设计，纯直插 + GPIO 飞线

### 项目结构建立

在 `D:\.newmam_project_all\` 创建完整目录，包含：
- `pl/rtl/` 五个 Verilog 源文件
- `pl/tb/` 三个 testbench
- `pl/constraints/pl_top.xdc` 引脚约束
- `pl/data/gen_exp_coe.py` + 生成的 `exp_pulse.coe`
- `scripts/create_project.tcl` 一键 Vivado 工程生成
- `docs/pin_map.md` 完整引脚映射表

### Tier 1 Verilog 代码（5 个模块，共 ~300 行）

- `dac_driver.v`：AD9767 双通道 14bit 接口驱动，CLK/WRT 短接同源时钟（参考 ACZ702 例程简化方案）
- `dds_core.v`：32bit 频率控制字 + 12bit 相位偏移的 DDS 相位累加器
- `exp_lut.v`：BRAM Block Memory Generator IP 包装，4096×14bit ROM
- `waveform_fsm.v`：三模式波形状态机（OFF / ONCE / PERIOD），支持首次等待一个完整周期后再触发
- `pl_top.v`：顶层集成，含 MMCM 时钟（50MHz→125MHz）、LED 心跳、双通道 DDS+LUT+DAC

### 波形数据生成

用 Python 脚本 `gen_exp_coe.py` 生成双指数核脉冲 LUT：
- 4096 点，14bit 无符号
- 峰值 14745（约 14bit 满量程的 90%，留余量给噪声）
- 基线 0

### Vivado 仿真验证（3 步走）

1. **tb_dds_core**：验证相位累加器每周期地址递增速率约 3.28（与 fword=3435974 完全匹配）
2. **tb_waveform_fsm**：验证 PERIOD 模式脉冲宽度 1.6μs 和周期 8μs（testbench 参数）
3. **tb_pl_top**：500μs 顶层仿真，看到 5 次周期性双指数脉冲，dac_a_data / rom_data_a 完全同步

### 遇到的坑（写入 FPGA 文档）

**坑 1：Vivado BRAM IP 的 Read_Width_A 是 disabled 参数**
- Tcl 里设 `Read_Width_A = 14` 被 Vivado 忽略（因为 Single_Port_ROM 下这个参数自动跟随 Write_Width_A）
- 实际位宽保持默认 16，导致端口宽度与 Verilog 输出不匹配
- 修复：Tcl 里改用 `Write_Width_A = 14`，或 GUI 手动改 Port A Width

**坑 2：Vivado BRAM IP 的 ena 端口必须显式连接**
- 通过 `Register_PortA_Output_of_Memory_Primitives = true` 生成的 IP 带 ena 端口
- 若 Verilog 实例化时不连 ena，行为不定，仿真表现为输出恒 0
- 修复：`.ena(1'b1)` 恒使能

**坑 3：MMCM 锁相环上电需要 ~1μs 才 lock**
- 全局复位 `rst_n = rst_n_key & mmcm_locked`，导致仿真前 1μs 所有输出为 0
- 这是硬件事实，不可避免
- 优化建议：波形观察时跳过 0-1μs 区间

### FSM 首次触发行为改进

原设计：MODE_PERIOD 上电后立即触发第一次脉冲
用户希望：上电后先等一个完整周期，再触发第一次脉冲
实现：加了 `first_pulse` 标志，控制 ST_WAIT 首次等待 period_cnt 拍，之后等 period_cnt - pulse_len 拍

### 最终仿真结果

500μs 仿真跑完，看到 5 次周期性脉冲（约 101、201、301、401、501μs 位置），每次 10μs 双指数衰减，双通道同频同相，DAC 数据链路完整通过。

### 明天/后续待办

- Tier 1 上板验证（综合 → 实现 → 生成 bitstream → 烧录 → 示波器测量）
- Tier 2：amplitude_scaler + noise_adder + LFSR + Box-Muller 高斯噪声
- Tier 3：HDMI 显示 + AXI-Lite PS 端参数接口 + 能谱采样 + 核素数据库
