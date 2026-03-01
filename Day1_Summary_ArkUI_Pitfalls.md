# 期权计算器 MVP 阶段一：开发回顾与 ArkUI 踩坑总结

## 1. 昨天工作回顾
我们初步完成了一个基于 HarmonyOS NEXT (ArkTS/ArkUI) 的期权交易辅助工具的基础构建。目前已实现以下核心模块：
- **T型报价板 (`QuoteBoard`)**：支持多标的、多到期月份的实时行情拉取，实现了期权价格展示，并在此基础上加入了**理论参考价**的对标以及**平值合约的红绿底色高亮锚定**。
- **BSM 定价与希腊字母计算器 (`CalculatorPanel`)**：
  - 支持通过输入标的价格、行权价、剩余天数、波动率和无风险利率，实时计算出期权的理论价格。
  - 支持计算 Delta, Gamma, Theta, Vega, Rho 等希腊字母。
  - 实现了基于 Newton-Raphson 迭代法的**隐含波动率 (IV) 倒算**功能（同时分别计算认购和认沽的 IV，避免相互污染）。
- **实时数据联动**：报价板点击某个合约，能够立刻跳转计算器面板，并将对应的合约代码、行权价、期权最新价（认购和认沽）带过去，同时自动触发网络请求拉取最新的 ETF 标的现价与无风险利率。

## 2. 踩坑与经验总结

在开发过程中，遇到的最大挑战集中在 **ArkUI 的状态管理与渲染机制** 上，同时也遇到了一些 ArkTS 的语法限制。

### 坑 1：ArkTS 语法的严苛限制
- **现象**：在使用 `Promise.all` 发起并发网络请求时，习惯性地使用了数组解构赋值：`const [jsonStr, etfJson] = await Promise.all(...)`，编译器直接报错 `arkts-no-destruct-decls`。
- **原因**：HarmonyOS 针对 ArkTS 的 AOT (Ahead-Of-Time) 编译进行了极致的性能优化，目前**明令禁止了数组解构和对象解构赋值**。
- **解决**：改用最原始的数组索引方式（如 `const results = await ...; const jsonStr = results[0];`）进行变量赋值。

### 坑 2：状态刷新混乱（数据刷新了，但页面死活不更新）
- **现象**：当从报价板点击合约跳到计算器时，日志显示所有的期权参数（S, K, T, v, r）都已更新，且内部的理论价和希腊字母已经计算出了正确结果，但是 **UI 界面仍然停留在旧数据或直接变成了 `0`**。
- **原因深度剖析**：
  1. **多重 `@Watch` 带来的时序混乱**：父组件在 `handleOptionClick` 中依次更新多个变量（标的、行权价、目标价格等）。因为每个被 `@Link` 的变量都挂载了 `@Watch`，导致修改每一个参数都触发一次重算。在异步网络请求未返回（现价 S 未更新）的情况下，前几次重算拿到了"残缺的"上下文，生成了错误数据甚至阻断了计算流。
  2. **`@Builder` 与值传递的陷阱**：原本的输入框和结果展示块封装在内部的 `@Builder` 中，并将 `@State`/`@Link` 类型的基础变量（`string`, `number`）作为参数传进去。ArkUI 对于基础类型传参是**按值传递**（pass-by-value）的，父组件的数据虽然变化了，但底层的 Builder 内部未检测到引用地址变更，因此**不触发重新渲染**。
- **目前的调整方法（阶段性解决）**：
  1. **引入“防抖”机制（Debounce）**：去掉所有零散的 `@Watch('calculateAll')`。将所有输入的变动都指向 `requestCalculate()` 方法。该方法内部用 `setTimeout` 延迟 50ms。这保证了哪怕多个参数在同一毫秒内被连续赋值，或者网络请求和参数传递并发进行，最终也只会执行**最后一次、参数最完整的**计算。
  2. **内联展开与 `$$` 双向绑定**：彻底移除了可能会阻断渲染的 `@Builder` 函数。把所有的 `TextInput` 和 `Text` 直接内联写在主 `build()` 树中，并且对 `TextInput` 强制使用了官方推荐的原生双向绑定语法（例如 `$$this.spotPrice`），省去了手写的 `onChange`，大幅度提升了数据追踪的敏锐度。
  
### 3. 未彻底修复的隐患
尽管通过上述“内联展开 + 强绑定 + 防抖重算”的方式暂时压制住了不渲染的问题，但这其实是一种妥协。
**潜在问题**：
当数据结构变得更加复杂，或者组件嵌套更深时，这种依赖顶层 `build` 暴力追踪基础属性的方式会显得脆弱，仍有可能出现“更新了深层状态但局部 UI 不重绘”的情况。

**未来的终极解法预留**：
1. **使用对象化状态管理**：不再传递一堆零散的 `string`/`number`，而是将整个期权计算参数封装为一个 `class`，结合 `@Observed` 类装饰器和 `@ObjectLink` 属性装饰器，让框架精准监听到对象属性的改变。
2. **事件总线 (`Emitter`)**：在跨层级或跨 Tab 同步强一致性要求较高的数据时，不完全依赖 `@Link`，而是考虑使用系统的 `Emitter` 广播事件，在特定生命周期钩子中强行接管数据刷新。

## 下一步开发计划
1. 深入研究 `@Observed` + `@ObjectLink` 重构现在的 Calculator 参数结构，彻底巩固状态更新机制。
2. 引入 `Highcharts` 或者原生的 ECharts 组件，实现所选合约及对冲策略的收益曲线（Payoff Chart）的实时绘制。
3. 梳理持仓与对冲分析模块的 UI 布局。

---

## 4. ArkUI 状态管理装饰器深度解析与重构预演

经过昨日的实战踩坑与讨论，我们明确了 ArkUI 的设计取舍：**牺牲运行时的动态深度追踪，换取 AOT 静态编译与底层渲染的极高流畅度**。它将“准确追踪数据”的负担，通过一套严格的装饰器语法转嫁给了开发者。

为了彻底摆脱“数据变了 UI 没变”的困境，必须深入理解并精确使用以下核心装饰器：

### 4.1 基础装饰器：`@State`, `@Link`, `@Prop` (适用于简单数据类型)
*   **`@State` (组件内私有状态)**：
    *   **作用**：定义组件内部的独立状态。改变它会触发当前组件的重新 `build()`。
    *   **局限**：只能**浅层监听**。如果是一个对象，只有给对象重新赋值（`this.obj = newObj`）或修改其第一层属性（`this.obj.prop = 1`）时才能触发。如果修改嵌套更深的属性，或者修改数组内部的元素属性，**UI 不会刷新**。
*   **`@Link` (双向同步)**：
    *   **作用**：在子组件中定义，用于与父组件的 `@State` 建立双向绑定。子改父变，父改子变。
    *   **坑点**：**不能在 `@Builder` 自定义构建函数中作为引用传递**。如果在 `@Builder` 中传入基础类型的 `@Link` 变量，它会退化为按值传递（Pass-by-value），从而切断响应式链路（这就是我们昨天踩的最大的坑）。
*   **`@Prop` (单向同步)**：
    *   **作用**：父组件向子组件单向传递数据。父组件改变会同步给子组件，但子组件内部修改 `@Prop` 不会同步回父组件，且会在下次父组件渲染时被覆盖。
    *   **适用场景**：只读的显示组件，如列表项的基础信息。

### 4.2 跨层级装饰器：`@Provide`, `@Consume` (适用于避免 Props drilling)
*   **作用**：类似于 React 的 Context。祖先组件用 `@Provide` 提供数据，其任意层级的后代组件用 `@Consume` 声明接收，即可建立双向绑定，省去了逐层传递 `@Link` 的麻烦。
*   **局限**：存在性能开销。因为它们基于字符串 Key 进行匹配绑定，且同样受限于浅层监听的缺陷。不建议用于高频刷新的复杂数据（如毫秒级跳动的行情数据），否则可能引起整个组件树的无效重绘。

### 4.3 深度对象装饰器：`@Observed`, `@ObjectLink` (应对复杂数据结构的终极武器)
这是处理多层嵌套对象（例如期权计算器中一整套包含十几个参数的配置对象，或者持仓列表中的单个合约详情）的唯一正确解法。

*   **`@Observed` (类装饰器)**：
    *   **作用**：必须装饰在 ES6 的 `class` 声明上。它告诉 ArkUI 编译器：“这个类实例化出来的对象，你需要给我生成一套代理（Proxy），监听它的内部属性变化”。
    *   **关键点**：仅靠 `@Observed` 本身是不够的，它只是打了个标记，必须配合 `@ObjectLink` 使用。
*   **`@ObjectLink` (属性装饰器)**：
    *   **作用**：用在子组件中，接收由父组件传递过来的、被 `@Observed` 装饰的类实例。
    *   **核心优势**：只要对象内部被 `@Observed` 代理的属性发生改变，拥有该 `@ObjectLink` 实例的**具体子组件就会发生精准的局部重绘**，而不需要像 `@State` 那样重绘整个父组件树。这完美契合了 ECS 架构中“精准打脏标记（Dirty）然后批量渲染”的思路。

### 4.4 针对 Calculator 面板的重构预演
基于上述理解，我们目前的 `CalculatorPanel` 接收了近 10 个 `@Link` 基础变量，这非常脆弱且容易出现时序问题（即前文提到的导致数据闪烁的问题）。

**未来的重构方向**：
1.  **定义专属配置类**：
    ```typescript
    @Observed
    export class CalculatorParams {
      spotPrice: number = 0;
      strikePrice: number = 0;
      daysToMaturity: number = 0;
      targetCallPrice: number = 0;
      targetPutPrice: number = 0;
      // ... 其他参数
    }
    ```
2.  **父组件（Index）实例化**：
    在 `Index.ets` 中只需维护一个单一状态：
    `@State calcParams: CalculatorParams = new CalculatorParams();`
    点击合约时，直接修改这个对象内部的属性。
3.  **子组件（CalculatorPanel）精准接收**：
    在 `CalculatorPanel.ets` 中：
    `@ObjectLink params: CalculatorParams;`
    在子组件的 UI 中直接绑定 `this.params.spotPrice`。当父组件更新这个对象的任何属性时，子组件将自动、精准、无错乱地完成局部重绘，从而可以彻底废弃掉目前的“防抖 `setTimeout` + 手动控制 `updateTrigger`”的权宜之计。

### 坑 3：TextInput 的 InputType.Number 吞噬小数点问题
- **现象**：在将网络请求获取的无风险利率（如 `1.59`）双向绑定给 `TextInput` 后，计算器有时会收到荒谬的 `159`（再除以 100 变成 `1.59` 代入公式），导致期权理论价格计算严重失真（本来应该是 `0.0159`）。
- **原因**：在 ArkUI 中，如果给 `TextInput` 设置了 `.type(InputType.Number)`，它会严格限制输入内容只能是纯整数（0-9）。当传入带有小数点的字符串 `"1.59"` 时，组件内部的非法字符过滤机制会**直接吞掉小数点**，将其变成 `"159"`，进而触发数据流的连锁错误。
- **解决**：对于任何需要输入或显示浮点数的输入框，**必须**明确使用 `.type(InputType.NUMBER_DECIMAL)`，这样才能正确保留并处理小数点。

### 坑 4：hvigor 编译器对重复声明的极端零容忍 (Identifier has already been declared)
- **现象**：在修改代码时不小心重复声明了同名变量（如写了两次 `const rawS = ...`，哪怕是在同一个作用域内被无意复制的），普通 JS/TS 引擎在某些宽泛环境下可能只报 warning 或者运行时覆盖，但 ArkTS 编译直接崩溃，抛出超长的 `RollupError` 报错栈，导致 Previewer 完全停止工作。
- **原因**：HarmonyOS 使用 hvigor 和 Rollup 组合的构建链，因为要强保证 AOT 的产物安全性，对作用域、变量重复声明执行了**最严格的词法级阻断**。只要 AST（抽象语法树）解析出重复定义，直接当做致命 `PARSE_ERROR` 抛出。
- **解决**：务必保持代码块的干净整洁。如果预览器突然罢工并报此类底层 Rollup 错，第一反应应是检查当前修改文件的上下文是否存在低级的语法错误（如重复声明、解构赋值等 ArkTS 禁用的特性）。

---

## 5. 行情模块开发与 Highstock 接入总结

为了实现专业的金融时间序列数据可视化（如期权标的的日内分时对比图），我们选择了业内成熟的 **Highstock** 图表库。由于 ArkUI 原生不支持直接渲染基于 DOM 的图表库，我们采用了 **Native + Web 混合渲染架构**。

### 5.1 混合架构设计实现
1. **宿主容器 (`MarketChartPanel.ets`)**：使用 HarmonyOS 原生的 `Web` 组件作为图表容器，并通过 `$rawfile('highstock.html')` 加载本地 HTML 文件。
2. **数据通讯桥梁 (Native to Web)**：
   - 所有的网络请求（跨域获取上交所 API 数据）全部在原生端 ArkTS 层使用 `ApiClient` 完成，从而完美避开了 Web 容器内的跨域（CORS）问题。
   - 数据拉取成功后，通过 `webviewController.runJavaScript()` 将 JSON 数据以字符串形式动态注入到 Web 层的 JavaScript 环境中执行渲染。

### 5.2 图表数据处理的核心要点
在接入 50ETF 和上证50 这种量级差异极大的分时数据时，我们解决了以下几个核心难点：

1. **绝对数值量级差异极大**：
   - 50ETF 价格在 3.11 左右，上证50 在 3030 左右。如果用普通双 Y 轴不仅刻度混乱，直观性也差。
   - **解法**：启用了 Highstock 自带的**百分比对比模式** (`plotOptions.series.compare = 'percent'`)，将两条线的起点强行锚定在 0% 轴上，Y轴只显示涨跌幅比例。这是金融软件做多标的走势叠加对比的最标准解法。
2. **时间序列与“午间休市断层”处理**：
   - A股在 11:30 到 13:00 之间是休市的。普通的图表引擎会将这段时间用一条直线连起来，或者留出一块巨大的空白。
   - **解法**：利用 Highstock 强大的 `xAxis.breaks` 属性，在 JS 处理数据时，动态解析当日 11:30 到 13:00 的 Timestamp 区间，并将其切除 (`breakSize: 0`)。这使得上午和下午的分时线能够完美、平滑地拼接在一起。

### 5.3 踩坑记录：对象字面量类型限制 (arkts-no-untyped-obj-literals)
- **现象**：在给 `Select` 组件传入下拉菜单项时，使用了常见的 JS 写法 `Select(PAIR_LIST.map(p => ({ value: p.name })))`，导致编译直接报错。
- **原因**：ArkTS 极度强调静态类型，为了保证 AOT 编译阶段能确定内存结构，**严禁使用无类型的对象字面量**。这与标准 TypeScript 允许匿名对象推导不同。
- **解决**：在匿名对象后增加明确的类型断言 `as SelectOption`，即 `Select(PAIR_LIST.map(p => ({ value: p.name } as SelectOption)))`，满足编译器对对象形状的严格审查。

### 5.4 Highstock 图表进阶交互：动态图例与坐标轴切换
- **需求**：默认双标的以百分比显示，但如果用户点击图例隐藏其中一条线，Y轴应该自动切换为该标的的**绝对数值**（保留实际价格参考意义）。同时X轴需要固定在全天 `09:30 - 15:00`，而不是只根据当前有数据的时间自动收缩。
- **实现方案**：
  1. **固定 X 轴区间**：在传入的 JSON 数据中解析出当前日期，通过 `chart.xAxis[0].update({ min: ..., max: ... })` 将 X 轴两端强制固定为当日的 09:30 和 15:00。
  2. **监听图例点击切换 Y 轴**：利用 Highstock 的 `plotOptions.series.events.show` 和 `hide` 事件。在事件回调中通过 `setTimeout(..., 10)` 延迟判断当前 `visibleSeries` 的数量：
     - 如果只有 1 条线显示：关闭 `compare` 模式 (`compare: undefined`)，并把 Y 轴的 `formatter` 改为直接返回原始数值 `this.value`。
     - 如果有 2 条线显示：重新开启 `compare: 'percent'`，把 Y 轴 `formatter` 改回显示百分比。

---

## 6. 业务逻辑纠错：期权到期日规则
- **错误发现**：前期为了快速搭建 MVP，简单粗暴地假设所有期权合约都在每月的 25 日到期（`new Date(year, month - 1, 25)`），这导致了随着临近月末，剩余天数（T）计算严重失真。
- **业务规则科普**：上交所 ETF 期权和中金所股指期权的标准到期日为：**每个月的第四个星期三**（遇法定节假日顺延）。
- **算法修正**：在 `ArkTS` 的日期处理逻辑中进行了如下调整：
  ```typescript
  // 获取本月第一天
  let firstDay = new Date(year, month - 1, 1);
  let dayOfWeek = firstDay.getDay(); // 0(周日) ~ 6(周六)
  // 计算距离第一个星期三(3)还有几天
  let offset = (3 - dayOfWeek + 7) % 7; 
  // 第一个星期三是 1 + offset 号，第四个星期三就是再加上 3周(21天)
  let expireDate = new Date(year, month - 1, 22 + offset).getTime();
  ```
  这一修正大幅提升了 BSM 理论价格和 Greeks 在临近交割日时的准确性。

## 7. 高级状态管理实战：对象更新不触发 UI 刷新的终极解法
在开发行情页面的快照数据面板时，遇到了“网络请求成功但 UI 一直显示‘暂无数据’”的经典问题。

### 7.1 问题根源分析
1. **类型匹配严格 (`arkts-no-any-unknown`)**：
   原生的 `JSON.parse` 在严格模式下返回的是 `any` 类型。ArkTS 不允许存在这种运行时推断的 `any`，必须强制赋予一个明确的类型（如通过 `as SnapResponse` 接口断言）。
2. **`@State` 浅层监听与 `@Builder` 的局限性**：
   最初将快照数据定义为普通的 `interface SnapData`，并在组件中使用 `@State etfSnap: SnapData | null` 声明。随后使用 `@Builder` 函数渲染 UI。
   当网络请求返回数据后，如果只修改对象的深层属性，`@State` 无法监听到（它只能监听到变量被重新赋值或第一层属性修改）。同时，`@Builder` 本身不具备独立的组件生命周期和刷新能力，导致底层面板无法重新渲染。

### 7.2 终极解决方案：`@Observed` + `@ObjectLink`
为了实现精准的局部刷新，我们将数据结构彻底重构为响应式对象：

1. **将数据结构彻底类化**：
   废弃普通的 `interface`，将其改为带有 `@Observed` 装饰器的类。
   ```typescript
   @Observed
   export class SnapDataObj {
     last: number = 0;
     volume: number = 0;
     hasData: boolean = false;
     // ...
   }
   ```
2. **抽离独立的子组件**：
   将原本的 `@Builder SnapCard` 升级为独立的 `@Component struct SnapCardView`，并通过 `@ObjectLink data: SnapDataObj` 接收数据。
3. **精准修改属性**：
   网络请求完成后，不再进行对象的重新赋值替换（这往往容易破坏引用导致状态丢失），而是直接调用更新函数修改类内部的各个属性：
   ```typescript
   target.last = snap[1];
   target.hasData = true;
   ```
这样一来，只要网络数据一到，ArkUI 框架就会精准捕获到 `SnapDataObj` 里每个数值的变动，并驱动底层的两个小面板立刻、局部地刷新。这一经验彻底验证了我们在第 4 节中总结的状态管理理论。

## 8. ArkUI 语法严苛限制：`build()` 方法内禁止声明局部变量
在重构行情面板的数据卡片以支持动态精度（指数保留2位小数，ETF保留4位小数）时，遇到了一个经典的 `ArkTS Compiler Error`。

### 8.1 现象与报错
为了动态设置精度，最初尝试在 `@Component` 的 `build()` 函数开头声明一个局部变量：
```typescript
build() {
  const precision = this.isIndex ? 2 : 4;
  Column() {
    Text(this.data.last.toFixed(precision))
    // ...
  }
}
```
这会导致编译报错：`In an '@Entry' decorated component, the 'build' method can have only one root node, which must be a container component.`

### 8.2 原因与解决方案
**原因**：与 React 的 `render()` 函数允许任意逻辑代码不同，ArkUI 赋予了 `build()` 方法极其严格的语法树结构限制。**`build()` 方法的第一层节点必须并且只能是一个容器组件**（如 `Column`, `Row`, `Stack` 等）。在组件节点之前不允许有任何形式的局部变量声明或逻辑前置计算。

**解决**：将局部前置逻辑抽离为一个独立的类方法：
```typescript
getPrecision(): number {
  return this.isIndex ? 2 : 4;
}

build() {
  Column() {
    Text(this.data.last.toFixed(this.getPrecision()))
    // ...
  }
}
```
这种设计强制开发者将“逻辑”与“声明式 UI 结构”彻底分离，虽然初期会觉得束手束脚，但正是这种严格的静态结构，保证了 ArkUI 底层渲染树的构建速度。

---

## 9. 模块升级规划：从“静态查询”到“期权智能体 (Options AI Agent)”

为了大幅提升 App 的壁垒与差异化体验，我们决定将原本枯燥的“查询”模块，重构为一个**具备期权专业知识的 Tool-Calling Agent**。

### 9.1 核心能力设计 (L1 - L3 进阶)
1. **L1 智能问答与解读**：识别用户意图（如“50ETF下月平值认购多贵”），结合实时行情与本地 BSM 模型，给出带有人性化解读的行情播报（IV 高低、Greeks 异常等）。
2. **L2 策略构建推荐**：根据用户的资金量和市场观点（如“温和看涨防回撤”），智能推荐期权组合（如牛市价差、备兑看涨），并计算盈亏平衡点和最大回撤。
3. **L3 持仓体检 (压力测试)**：解析用户现有头寸（现货 + 期权多空），计算整体组合的 Portfolio Greeks，并模拟大盘涨跌 5% 情况下的盈亏场景，给出对冲建议。

### 9.2 UI/UX 交互形态 (Rich Media Chat)
摒弃传统的表单结构，采用类似 ChatGPT 的对话流，但**强依赖富媒体卡片**：
- **快捷胶囊区 (Quick Prompts)**：提供“一键构建温和看涨”、“测算备兑收益”等高频意图入口。
- **互动式合约卡片**：LLM 输出的不仅是文本，遇到特定合约代码时，渲染为 ArkUI 可点击卡片，点击直达“计算器”或“报价板”。
- **内嵌 Payoff 图表**：当推荐组合策略时，对话气泡内嵌 Highcharts 画出的组合到期收益曲线（Payoff Chart）。

### 9.3 核心流转机制：App 主导计算 + LLM 负责逻辑总结
**绝对禁止让大模型直接进行数学计算**（以避免严重的幻觉和精度丢失）。
1. **Tool-Calling**：向 LLM (如 MiniMax 2.5 或本地 Nemotron3) 发起请求时，注册本地工具能力声明（如 `calculate_options_greeks(code, strike)`）。
2. **App 截获与计算**：当 LLM 决策需要调用工具时，返回 Function Call 指令。App 截获该指令，调用原生代码中的 `ApiClient` 拉取行情，并运行本地的 `BSMModel.ets` 得出精准的 IV 和价格。
3. **数据回传与渲染**：App 将绝对精准的计算结果（JSON）作为 Tool Result 扔回给 LLM。LLM 基于准确数据生成格式化解释文本和图表标记，最后交由 ArkUI 进行气泡或图表组件渲染。

**落地分期**：
- **Phase 1**：搭建 ArkUI 聊天流界面 (`List` + `TextInput`)。
- **Phase 2**：连通 LLM API（支持 SSE 流式输出）。
- **Phase 3**：注入上下文提示词与本地实时行情。
- **Phase 4**：实现 Function Calling 与富文本卡片/图表的解析渲染。

---

## 10. 踩坑记录：LLM 接口联调中的 ArkTS 强类型约束

在将期权智能体接入大模型（MiniMax 2.5，采用 Anthropic 接口规范）的过程中，我们在构建 HTTP 请求体（JSON payload）和处理响应时，遭遇了 ArkTS 编译器连珠炮式的拦截报错。

### 10.1 现象与报错信息
在尝试使用常规 JavaScript/TypeScript 语法快速构建请求结构时（如 `messages: [{ role: 'user', content: text }]`），Hvigor 抛出了如下一连串致命错误：
1. `arkts-no-obj-literals-as-types`: Object literals cannot be used as type declarations（对象字面量不能用作类型声明）。
2. `arkts-no-noninferrable-arr-literals`: Array literals must contain elements of only inferrable types（数组字面量必须包含可推断类型的元素）。
3. `arkts-no-untyped-obj-literals`: Object literal must correspond to some explicitly declared class or interface（对象字面量必须对应某个显式声明的类或接口）。
4. `arkts-no-any-unknown`: Use explicit types instead of "any", "unknown"（使用显式类型替代 any 或 unknown）。

### 10.2 问题根源
这是 ArkTS 为保障 AOT (Ahead-Of-Time) 编译极致性能所施加的**最严格的静态类型限制**：
- **禁止隐式 Any**：像 `JSON.parse()` 这样在原生 TS 中返回 `any` 的函数，在 ArkTS 中被视为类型不安全，必须被强行断言。
- **禁止匿名对象与结构推导**：在标准 TS 中，我们可以直接用 `{ a: 1, b: 2 }` 的形式快速构造嵌套对象，或用它作为泛型参数（如 `Array<{role: string}>`）。但在 ArkTS 中，这种缺乏类或接口模板支撑的数据结构无法在编译期确定固定的内存布局，因此被彻底禁用。

### 10.3 标准修复方案
为了向大模型发送复杂的嵌套 JSON，我们不能偷懒，必须按照严格的面向对象范式进行构建：

1. **为所有请求/响应的 JSON 节点声明确切的 `interface` 或 `class`**：
   ```typescript
   export interface LLMMessage {
     role: string;
     content: string;
   }
   interface MinimaxContent { type: string; text: string; }
   interface MinimaxMsg { role: string; content: MinimaxContent[]; }
   interface MinimaxRequest {
     model: string;
     max_tokens: number;
     system: string;
     messages: MinimaxMsg[];
     stream: boolean;
   }
   ```
2. **在解析 JSON 时执行严格断言**：
   ```typescript
   // 错误：let resultObj = JSON.parse(response.result as string);
   // 正确：
   let resultObj = JSON.parse(response.result as string) as MinimaxResponse;
   ```
3. **在组装数组或对象时，用显式的变量和类型兜底**（避免隐式类型推断失败）：
   ```typescript
   // 错误：messages.map(m => ({ role: m.role, content: [{ type: "text", text: m.content }] }))
   // 正确：
   messages.map(m => {
     let contentObj: MinimaxContent = { type: "text", text: m.content };
     let msgObj: MinimaxMsg = { role: m.role, content: [contentObj] };
     return msgObj;
   });
   ```
### 10.4 动态响应解析坑：JSON 嵌套与数组查找
即使请求成功返回（HTTP 200），如果不严格按照结构解析响应，依然会导致拿不到数据。例如，MiniMax Anthropic 接口返回的是：
```json
{"content":[{"thinking":"...","type":"thinking"},{"text":"...","type":"text"}]}
```
由于 ArkTS 无法直接对 `Object` 类型调用 `find` 等数组方法，如果强行把 `resultObj` 断言为带有 `content` 数组的接口，但内部元素的类型没有严格定义，会在运行时获取到 `undefined`。

**解决**：
```typescript
let resultStr = response.result as string;
let resultObj = JSON.parse(resultStr) as Record<string, Object>;

let contentArr = resultObj['content'] as Array<Record<string, string>>;
if (contentArr && Array.isArray(contentArr)) {
  // 寻找真正的回答文本块（跳过 thinking 块）
  let textBlock = contentArr.find((block) => block['type'] === 'text');
  if (textBlock && textBlock['text']) {
    return textBlock['text'];
  }
}
```
通过使用 `Record<string, Object>` 和具体的字段类型断言，彻底打通了从动态 JSON 中安全提取深层字段的通路。

## 十一、 高阶实践：大模型 Function Calling 与富文本卡片渲染
在项目的 Phase 4，我们成功为期权 AI 智能体接入了实时行情数据获取的能力，并实现了原生 UI 级别的精美策略卡片渲染。以下是三大核心技术点的实现拆解：

### 11.1 通过 HarmonyOS 调用 LLM 实现对话
**相关文件：** `entry/src/main/ets/network/ApiClient.ets` & `entry/src/main/ets/components/AgentPanel.ets`

由于 ArkTS 不支持直接引入 Node.js 的第三方 SDK（如官方的 openai 或 anthropic 库），我们需要通过原生的 `@ohos.net.http` 直接对接大模型的 RESTful API。
1. **接口类型防坑**：如同前文所述，为了避免 `arkts-no-untyped-obj-literals` 报错，我们在 `ApiClient.ets` 顶部定义了极其严格的接口（`MinimaxRequest`, `MinimaxMsg`, `MinimaxContent` 等），并手动映射聊天记录。
2. **异步通信与 UI 更新**：在 `AgentPanel.ets` 中，维护了 `@State messages: Array<AgentMessage>`。用户点击发送时，触发 `handleSend` 方法，该方法会将用户消息推入数组，显示“正在思考”状态，然后 `await ApiClient.sendChatMessage(chatHistory)`。拿到结果后再 `push` 到数组中，利用 ArkUI 的 `@State` 机制自动刷新对话列表，并调用 `scrollToBottom` 滚动到底部。

### 11.2 让 LLM 在 Harmony 框架内实现 Tool Calling (Function Calling)
**相关文件：** `entry/src/main/ets/network/ApiClient.ets`

为了让 LLM 拿到上交所的实时数据，我们让 App 接管了 Tool Calling 的完整闭环：
1. **注册工具 (Tool Declaration)**：在向 LLM 发送的请求体中，通过 `tools` 字段注册了本地工具 `get_atm_option_chain`，并利用 JSON Schema 告诉 LLM 这个工具需要 `stockId` 作为参数。
2. **拦截并递归 (Intercept & Recursion)**：重构了 `runLLMWithTools` 方法，设置了 `maxDepth` 防止死循环。
   - 当 API 返回 200 且解析到 `response.content` 中存在 `type === 'tool_use'` 时，**我们不会将这个结果返回给 UI**。
   - 而是记录下 LLM 的思考过程和函数调用指令，将其作为 `role: 'assistant'` 的消息推入历史记录。
3. **本地执行 (Local Execution)**：触发本地静态方法 `executeTool`。在该方法内部，程序偷偷并发调用了 `getEtfSnap`（获取标的现价）、`getStockExpire`（获取当月月份）、`getTQuote`（获取T型报价）三个接口，在本地将庞大的数据清洗、排序，仅提取平值附近上下两档的 10 个合约。
4. **喂回数据并追问**：将清洗后的精简 JSON 组装成 `type: 'tool_result'`，放入 `role: 'user'` 的消息中，**递归调用** `runLLMWithTools` 再次请求大模型：“数据拿到了，请基于这些数据回答用户”。最终 LLM 结合真实数据生成策略文本。

### 11.3 截取卡片信息并利用 ArkTS 渲染原生组件
**相关文件：** `entry/src/main/ets/components/AgentPanel.ets`

为了突破纯文本 Markdown 渲染表格体验不佳的问题，我们采用了“大模型结构化输出 + 本地原生组件渲染”的混合方案：
1. **Prompt Engineering 强制输出 JSON**：在系统提示词中，明确规定大模型在推荐期权策略时，**必须在回复末尾附带一段严格格式的 JSON**，并用 ` ```json ` 包裹（例如包含 `name`, `maxProfit`, `breakeven`, `legs` 数组等字段）。
2. **消息解析拦截 (Message Parsing)**：在 `AgentMessage` 类的构造函数中调用 `parseContent()`。利用正则表达式 `/```json\n([\s\S]*?)\n```/` 扫描大模型返回的全文。
   - 如果抓取到了 JSON，使用 `JSON.parse` 解析并存入类的 `strategyData: Record<string, Object>` 属性中。
   - 将原文本中的 JSON 字符串剔除，剩余的纯文本存入 `textPart`。
3. **简易 Markdown 引擎与原生表格**：在 `parseMarkdown` 方法中，手动按行解析纯文本，剔除 `**`、`###` 等 Markdown 标记，并将包含 `|` 的表格行提取到多维数组中。
4. **横向滚动防溢出渲染**：在 `@Builder MessageBubble` 中：
   - 纯文本通过 `Text` 直接渲染。
   - 提取的表格数据传入 `@Builder MarkdownTable`，利用 `Row` 和 `Column` 结合 `Scroll` 容器渲染为原生表格。


## 十二、 架构思考与跨平台演进：ArkTS vs iOS/Android/Web

在完成了基于 ArkTS 的 MVP (包含复杂的状态计算、WebView 混合图表、以及 LLM Function Calling) 之后，我们需要站在架构层面审视当前的技术栈，并评估将此核心资产向其他平台迁移的可行性。

### 12.1 现代端侧开发的“大一统”：声明式 UI
当前的客户端开发，在宏观范式上已经实现了高度统一：
- **HarmonyOS**: ArkUI (`Row { Column { Text() } }`)
- **iOS**: SwiftUI (`HStack { VStack { Text() } }`)
- **Android**: Jetpack Compose (`Row { Column { Text() } }`)
- **Web (跨端)**: React / React Native / Flutter

它们的核心逻辑惊人地一致：**`UI = f(State)`**。
不再需要获取 DOM 节点或 `findViewById` 去手动修改属性，只需定义好数据（State）和 UI 的映射关系，框架会自动接管界面的刷新。这种高度的思维同构性，是跨平台转译或重写的基础。

### 12.2 渲染机制与底层逻辑的异同
虽然范式相同，但由于底层运行时的区别，其渲染机制有着本质的不同，这也是我们在 ArkTS 开发中踩坑的根源：

1. **Web (React/Vue) 的 Virtual DOM**
   - **机制**：JS 引擎维护内存对象树（VDOM），状态改变时计算 Diff，再跨语言桥接通知 C++ 渲染引擎更新真实 DOM。
   - **痛点**：跨语言通信有性能损耗，深层 Diff 耗时。

2. **iOS/Android 原生引擎 (SwiftUI/Compose)**
   - **机制**：利用强类型语言的**编译器魔法**。编译期分析状态依赖，运行时精准定位局部重绘（Re-composition），直接与 GPU 交互，性能极高。

3. **HarmonyOS (ArkUI & ArkTS)**
   - **机制**：ArkTS 并非运行在浏览器里的标准 TS/JS，而是经过严格 AOT（提前编译）的静态类型语言，去除了所有动态特性。
   - **渲染刷新**：为了极致性能，ArkUI 默认只做**浅层地址比对**。这也是我们在 `CalculatorPanel` 和 `MarketChartPanel` 中遇到的最大挑战：修改对象的深层属性（如 `calcParams.spotPrice = x`）不会触发 `@State` 刷新，必须强制引入 `@Observed` + `@ObjectLink` 建立深度代理机制。这体现了华为在性能和灵活性之间选择了极致性能。

### 12.3 MVP 跨平台迁移与自动化转译构想
由于 ArkTS 强制要求了极度严格的静态类型，在这个环境下跑通的业务代码（特别是 Model 层和网络层），其健壮性极高，甚至成为了优秀的“基准代码”。这为跨平台迁移提供了几条清晰的路径：

#### 路径 A：基于 React Native / Expo 的跨平台重写
- **业务逻辑**：由于 ArkTS 本质上是 TS，`BSMModel.ets`、`ApiClient.ets` (含 LLM 交互逻辑)、核心数据结构等，可以**近乎 0 成本（一字不改）**复制到 React Native 运行。
- **混合图表**：`Highstock` 通过 HTML/Web 组件的交互逻辑，可以完美平移给 `react-native-webview`。
- **UI 适配**：需将 ArkUI 转换为 JSX 语法，将 `@State`/`@ObjectLink` 转换为 React Hooks (`useState`/`useMemo`)。

#### 路径 B：原生生态平移 (SwiftUI / Kotlin)
- 强类型的 ArkTS 可以很容易地建立与 Swift/Kotlin 的一对一映射。
- 状态管理机制高度同源（如 `@ObjectLink` 对应 SwiftUI 的 `@ObservedObject`）。

#### 远期规划：构建跨端转译 LLM Agent
基于上述高度一致的范式映射规则，下一步完全有可能编写一个**专用的 LLM Agent 或转译工具**：
1. **抽象语法树 (AST) 解析**：解析 ArkTS 的结构，分离 `Model`（数据/逻辑）、`View`（UI 结构）和 `ViewModel`（状态绑定）。
2. **LLM 语义翻译**：利用大型语言模型（如目前集成的 MiniMax 或其他 Coding LLM），将 ArkUI 的特定链式调用语法（`.width()`, `.padding()`）精确转译为 SwiftUI 的 ViewModifier 或 React 的 Style 对象。
3. **状态注入**：自动将 ArkTS 的装饰器（`@State`, `@Prop`）转换为目标平台的响应式状态原语。

可以说，**ArkTS 的严苛限制反而成了跨平台转译最好的“强类型模板”**。在这个平台上沉淀的“期权智能体”架构，完全具备成为多端同构标杆的潜力。