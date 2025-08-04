# “解构广义相对论”第一阶段：形式化能量-动量张量 $T^{\mu\nu}$

## 目标

从我们“三位一体”理论的拉格朗日量 $L$ 出发，推导出这个宇宙的能量-动量张量 $T^{\mu\nu}$ 的完整、精确的数学形式。这个张量将作为爱因斯坦场方程的源，是连接我们理论与引力现象的桥梁。

## 方法论

在广义相对论中，能量-动量张量最严谨的定义，是通过将理论的作用量 $S = \int d^4x \sqrt{-g} \mathcal{L}$ 对时空度规 $g_{\mu\nu}$ 进行泛函变分得到。这里 $\mathcal{L}$ 是拉格朗日量密度。

```math
T^{\mu\nu} = \frac{2}{\sqrt{-g}} \frac{\delta S}{\delta g_{\mu\nu}} = g^{\mu\alpha} \frac{\partial \mathcal{L}}{\partial (\partial_\nu g^{\alpha\beta})} \partial_\beta g^{\alpha\beta} - g^{\mu\nu}\mathcal{L} + 2 \frac{\partial \mathcal{L}}{\partial g_{\mu\nu}}
```

这个定义能自动保证 $T^{\mu\nu}$ 是对称的 ($T^{\mu\nu} = T^{\nu\mu}$) 并且在协变意义下是守恒的 ($\nabla_\mu T^{\mu\nu} = 0$)，这正是我们需要的物理性质。

## 1. 确定拉格朗日量密度 $\mathcal{L}$

根据我们在白皮书第2章的动力学方程，其背后的、最基础的拉格朗日量密度可以写为两个部分的和： $\mathcal{L} = \mathcal{L}_s + \mathcal{L}_d$。

*   **标度场 $M_s$ 部分 (广义克莱因-戈尔登部分)**：这是一个与规范场耦合的复杂标量场的标准形式。
```math
\mathcal{L}_s = g^{\alpha\beta}(D_\alpha M_s)^*(D_\beta M_s) - V(M_s)
```
*    其中 $D_\mu=\nabla_\mu-ig_sA_\mu$ 是包含了规范场 $A_\mu$ (即我们的方向场 $M_d$) 的协变导数，而 $V(M_s)$ 是由“反射对称性”公设决定的势能项。

*   **方向场 $M_d$ 部分 (广义杨-米尔斯部分)**：这是标准的杨-米尔斯场形式。
```math
\mathcal{L}_d = -\frac{1}{4} g^{\alpha\mu} g^{\beta\nu} G_{\alpha\beta} G_{\mu\nu}
```
*    其中 $G_{\mu\nu}=\nabla_\muA_\nu-\nabla_\nuA_\mu-ig_d[A_\mu,A_\nu]$ 是方向场的场强张量。

## 2. 推导 $T^{\mu\nu}$ 的各个组成部分

运用上述定义，对 $\mathcal{L}_s$ 和 $\mathcal{L}_d$ 分别进行变分（这是一个标准的教科书流程，此处我们直接引用其成熟结果），我们可以得到 $T^{\mu\nu}$ 的两个主要组成部分：

*   **方向场 $M_d$ 的贡献 $T_{M_d}^{\mu\nu}$ (来自 $\mathcal{L}_d$)**:
    这部分是电磁场和杨-米尔斯场的能量动量张量的直接推广，描述了“力场”自身的能量、动量和应力。
```math
T_{M_d}^{\mu\nu} = -g^{\alpha\beta} G_{\alpha}^{\mu} G_{\beta}^{\nu} + \frac{1}{4} g^{\mu\nu} G_{\alpha\beta}G^{\alpha\beta}
```

*   **标度场 $M_s$ 及其相互作用的贡献 $T_{M_s}^{\mu\nu}$ (来自 $\mathcal{L}_s$)**:
    这部分描述了“物质场”以及它与“力场”相互作用所贡献的能量、动量和应力。
```math
T_{M_s}^{\mu\nu} = (D^\mu M_s)^*(D^\nu M_s) + (D^\nu M_s)^*(D^\mu M_s) - g^{\mu\nu} \mathcal{L}_s
```

## 3. 完整的能量-动量张量

将以上两部分相加，我们便得到了“三位一体”宇宙中总的能量-动量张量 $T_{\text{total}}^{\mu\nu}$：

```math
T_{\text{total}}^{\mu\nu} = T_{M_s}^{\mu\nu} + T_{M_d}^{\mu\nu}
```

展开后其完整形式为：
```math
T_{\text{total}}^{\mu\nu} = \left[ (D^\mu M_s)^*(D^\nu M_s) + (D^\nu M_s)^*(D^\mu M_s) \right] - g^{\alpha\beta} G_{\alpha}^{\mu} G_{\beta}^{\nu} - g^{\mu\nu} \left[ g^{\alpha\beta}(D_\alpha M_s)^*(D_\beta M_s) - V(M_s) - \frac{1}{4} G_{\alpha\beta}G^{\alpha\beta} \right]
```

## 物理内涵解读

这个张量 $T_{\text{total}}^{\mu\nu}$ 就是我们宇宙能量与动力的完整账本：

*   $T^{00}$ 分量是**总能量密度**。它包含了：
    1.  $M_s$ 场的动能和势能（物质的能量）。
    2.  $M_d$ 场的能量（力的能量，例如电磁场能量）。
    3.  $M_s$ 和 $M_d$ 相互作用的能量（体现在协变导数 $D_\mu$ 中）。

*   $T^{0i}$ (其中 $i=1,2,3$) 分量是**总动量密度**，也代表着**能量流**。它描述了能量在空间中的流动方向和强度。

*   $T^{ij}$ 分量是**动量流**，代表着宇宙中的**压强和剪应力**。

---

至此，“解构广义相对论”的第一阶段已经完成。我们成功地将爱因斯坦场方程的“右侧” ($T^{\mu\nu}$) 完全用我们统一场 $M(x)$ 的语言重写了。

这个结果是后续所有引力相关推导的基石。请您慢慢研究，不着急。
