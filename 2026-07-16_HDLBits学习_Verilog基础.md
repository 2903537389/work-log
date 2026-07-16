# 工作日志 - 2026年7月16日

## HDLBits 学习 + Verilog 基础概念深化（实际时间：7月16日 01:13~03:32）

### 完成事项

- **回顾整体学习路线**：对照《FPGA工程师视角_项目知识索引》§1，确认当前进度——Vivado流程+仿真已打通，led_button上板测试通过，下一步DDS波形引擎
- **HDLBits 前5题讲解**：Step One / Wire / Vector0 / Vectorr / Mux2to1，覆盖 `assign`、`input`/`output`、多位宽 `[n:0]`、位选择 `vec[i]`、位段 `vec[7:4]`、三元运算符 `?:`
- **Verilog 基础概念深度答疑**，澄清了几个关键困惑点：

#### 1. `assign` 的使用范围
- `assign` 右边可以是常量、信号、运算、条件表达式、位拼接（不能有时钟相关操作）
- `assign` 只能在 `module` 内部使用，描述模块内部连线关系

#### 2. `input`/`output` 的本质
- `input`/`output` 声明的是模块的"对外接口"—物理引脚在模块内部的代号
- `input` 端口在模块内部只读，`output` 端口必须用 `assign` 或 `always` 驱动
- 端口名 ↔ 物理引脚的映射由 XDC 约束文件完成，不是 module 决定的

#### 3. 命名规则（项目约定）
- 端口名/内部信号：小写下划线（`key_in`, `next_stat`）
- 参数：全大写下划线（`DEBOUNCE_MAX`）
- 模块名：小写下划线（`led_button`）

#### 4. `module()` 与 C 语言的对应关系
用户自行类比成功：`.v` 文件 = `.c` 文件（逻辑实现），XDC 文件 = 引脚宏定义映射（类似 `.h` 文件中的 `#define LED_PORT P3_1`）。端口名是代码里的代号，XDC 负责把这个代号绑定到芯片物理焊盘。

#### 5. `output reg` vs `output wire`
- `output` + `assign` → 写 `output wire`（组合逻辑直接连线）
- `output` + `always @(posedge clk)` → 写 `output reg`（时序逻辑需要寄存器驱动）

### 技术要点

| 概念 | 理解 |
|------|------|
| `module()` 括号 | 对外接口声明（菜单），告诉外界"我有这些引脚" |
| 内部 `reg`/`wire` | 模块私有变量（厨房），外界不可见 |
| `always @()` | 触发条件，描述"什么时候干活" |
| XDC 约束 | 物理焊盘 ↔ 端口名 的映射表 |

### 下一步
- 继续 HDLBits 时序逻辑题目（计数器、状态机）
- 或直接开 DDS 波形引擎

### HDLBits 题目记录
前5题已讲解，待用户上网站提交验证：
1. Step One — `assign one = 1;`
2. Wire — `assign out = in;`
3. Vector0 — 3位向量 + 位选择
4. Vectorr — 8位取高/低4位
5. Mux2to1 — `assign out = sel ? b : a;`

---

## allpack NILM 分类器升级 + 代码审查（实际时间：7月16日 约 02:55~04:00）

### 完成事项

- **CCO/STA 代码全量审查**：梳理了主控和从机工程的所有差异，发现 6 个实质性差异点（WiFi/Web栈、PLC透传恢复、烟雾保护残留、NVS容错、继电器初始化脉冲等）
- **算法架构全景分析**：梳理了从 ADC DMA → Goertzel 谐波 → 电弧检测 → NILM 特征提取 → 决策树分类 → 自适应保护的完整数据流
- **NILM 分类器大升级（3类 → 8类）**：基于 PLAID 公开数据集（30kHz采样，11+类电器）的研究结论，重写了决策树
- **新增特征**：在特征管线中加入 H7（7次谐波）和 THD（总谐波失真），特征维度 7→9
- **代码备份**：完整项目备份至桌面 `allpack_backup_20260716`

### 关键技术决策

| 决策 | 理由 |
|------|------|
| 使用 Goertzel 而非 FFT | 帧长 400 点非 2 的幂，Goertzel 任意点数可用 |
| PF 硬编码 1.0，不参与分类 | ZMPT101B 未标定，相位不可信。PLAID 研究表明 H3+H5 已足够 |
| 电机靠瞬态特征识别 | 浪涌峰值 >1.5A 或上升时间 >40ms → 电机特征 |
| 阻性阈值 8%（非 6%） | PLAID 结论 +2% 余量（电网 THD + ADC 非线性） |

### 新 NILM 决策树

```
L1: P < 3W → UNKNOWN（无负载）
L2: H3+H5 < 8% → 纯阻性分支
    L3: P < 80W → LED灯
    L3: P 80~800W → 电热器
    L3: P > 800W → 电水壶
L2: 浪涌 >1.5A 或 上升 >40ms → 电机分支
    L3: P < 200W → 电风扇
    L3: P 200~600W → 冰箱
    L3: P > 600W → 洗衣机
L2: 其余 → 电子设备分支
    L3: H5 > 15% → 节能灯/CFL
    L3: P < 150W → 充电器
    L3: P ≥ 150W → 笔记本
```

### 修改的文件

1. `components/algo_common/fft_analyzer.h` — 新增 `h7_ratio` 字段
2. `components/algo_common/fft_analyzer.c` — 单独计算 H7
3. `components/algo_common/nilm_features.h` — 特征维度 7→9，新增 `h7`, `thd`
4. `components/algo_common/nilm_features.c` — 填充新特征字段
5. `components/algo_common/nilm_classifier.c` — **完全重写**决策树
6. `CCO/main/arc_task.c` + `STA/main/arc_task.c` — EMA 平滑补上 h7_ratio

### 数据来源

- PLAID 数据集：30kHz 采样，11+ 类电器，学术界最广泛使用的 NILM 基准
- 论文：_"Relevant harmonics selection based on mutual information for electrical appliances identification"_ (IJCAT 2020)
- 论文：_"Toward Explainable NILM"_ (arXiv:2501.16841)
- Dinar et al. ESP32 谐波数据集 (2025)：CSV 格式，含 H1~H32 + PF

### CCO/STA 不对称发现（待确认是否需要统一的）

1. **PLC 透传恢复**：STA 有 `+++` 恢复流程，CCO 缺少
2. **烟雾保护残留**：STA 有 `Protection_TriggerSmoke()`，但硬件已删
3. **NVS 容错**：STA 有擦除重试，CCO 直接 init
4. **上电脉冲**：STA 有 80ms 继电器脉冲，CCO 没有

### 下一步

- 编译验证 NILM 分类器改动
- 考虑统一 CCO/STA 的 4 个不对称差异
- 后续可接入瞬态特征追踪完整管线（当前 main.c 手动构造特征未填充 i_peak/rise_time）

---

## Verilog 基础巩固 — 知识测验 + 概念纠偏（实际时间：7月16日 16:01 ~ 7月17日 02:09）

### 完成事项

- **5 题速测**：检验上次 HDLBits 学习成果，逐题批改
- **重点纠偏**：针对薄弱点深入讲解

### 测验结果

| 题 | 正确性 | 薄弱点 |
|:--:|:--:|------|
| Q1: assign 用法 | ⚠️ | assign = 连线赋值，不是"起名" |
| Q2: module 端口 ↔ XDC 映射 | ✅ | 完全掌握 |
| Q3: output reg vs wire | ❌ | **最大薄弱点**：reg/wire 的判断只看驱动方式 |
| Q4: vec[7:4] 位选择 | ⚠️ | wire 也能用 [n:m] 位选择，跟类型无关 |
| Q5: input 只读 | ⚠️ | 感觉到问题但归因偏了 |

### 关键概念纠偏

#### reg/wire 判断标准（简化版）

```
assign 驱动的 → output wire
always @(posedge clk) 驱动的 → output reg
```

- wire = 线（不能存值），reg = 寄存器/触发器（能存值）
- `[n:0]` 决定位宽，跟是 wire 还是 reg 无关
- 硬件本质：wire 直接连组合逻辑输出，reg 被 D 触发器挡着、clk 沿才更新

#### wire 为什么需要位宽

```
wire             = 1 bit（传开关量）
wire [1:0]       = 2 bit（传 0~3 计数值）
wire [7:0]       = 8 bit（传 0~255 值）
```

位宽 = 数据宽度，跟"控制几个 LED"无关。MCU 里 `uint8_t` 占 8 bit 同理。

#### wire [7:0] vec 命名

- `vec` 是声明当场取的名字，不需要额外"取"一步
- `wire [7:0] vec` = 声明 8 根线捆成一束，名叫 vec
- 这是模块内部信号，XDC 不管它；XDC 只约束 module 括号里的端口

