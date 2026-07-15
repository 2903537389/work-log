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
