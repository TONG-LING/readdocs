模型架构
========

整体架构
--------

.. mermaid::

   graph TD
      sumo["SUMO 交通仿真"]
      dqn["DQN 信号控制器"]
      data["真实评估样本"]
      diffusion["条件扩散模型"]
      candidates["候选 reward 序列"]
      proxy["代理模型 MLP"]
      posterior["后验筛选"]
      pareto["Pareto 平衡选择"]
      nextgen["下一代种群"]

      sumo --> dqn
      dqn --> data
      data --> diffusion
      diffusion --> candidates
      data --> proxy
      candidates --> proxy
      proxy --> posterior
      data --> pareto
      pareto --> posterior
      posterior --> nextgen
      nextgen --> sumo

后验筛选结构
------------

.. mermaid::

   graph TD
      prior["先验：扩散模型生成候选池"]
      score["代理模型预测 mean 和 uncertainty"]
      utility["计算 proxy_score"]
      rank["按 proxy_score 排序"]
      select["选择 Top-K 候选"]
      elite["注入真实 Pareto elite"]
      next["形成下一代种群"]

      prior --> score
      score --> utility
      utility --> rank
      rank --> select
      elite --> next
      select --> next

当前实现中，扩散模型负责提供候选样本池，代理模型负责预测候选样本的真实 SUMO 表现，后验筛选模块再根据代理分数选择下一代样本。

DQN 控制器
----------

DQN 用于交通信号控制，与 SUMO 环境交互并产生状态、动作和 reward 数据。

每个候选 reward 序列会构造对应经验，并用于评估该候选样本对交通控制策略的影响。

条件扩散模型
------------

条件扩散模型用于生成候选 reward 序列。

输入：

.. code-block:: text

   state_action_condition

输出：

.. code-block:: text

   reward_sequence

扩散模型在后验框架中承担先验作用，即提供候选样本池。

可以写成：

.. math::

   x_i \sim p_\theta(x \mid c)

其中 :math:`x_i` 是第 :math:`i` 个候选 reward 序列，:math:`c` 是当前状态-动作条件。

代理模型
--------

当前代理模型是基于统计池化特征的 MLP 回归模型。

输入：

.. code-block:: text

   reward_sequence + state_action_condition

输出：

.. code-block:: text

   predicted_balance_score

模型先将 reward 序列和 condition 拼接，然后对时间维度提取均值、标准差、最小值和最大值，再送入 MLP。

MLP 结构：

.. code-block:: text

   LayerNorm
   Linear
   GELU
   Dropout
   Linear
   GELU
   Dropout
   Linear

代理模型结构图：

.. mermaid::

   graph TD
      reward["reward_sequence x"]
      condition["condition c"]
      concat["拼接 concat(x,c)"]
      pool["统计池化 mean/std/min/max"]
      mlp["MLP 回归器"]
      pred["predicted_balance_score"]

      reward --> concat
      condition --> concat
      concat --> pool
      pool --> mlp
      mlp --> pred

代理模型可以表示为：

.. math::

   \hat{y}_i = f_\phi(x_i, c)

其中 :math:`\hat{y}_i` 是代理模型预测的 `balance_score`，:math:`y_i` 是 SUMO 仿真得到的真实 `balance_score`。

损失函数
--------

代理模型当前使用 MSE 回归损失。

.. code-block:: text

   loss = MSE(predicted_balance_score, true_balance_score)

训练时目标值使用标准化后的 `balance_score`。

数学形式为：

.. math::

   \mathcal{L}_{proxy}
   =
   \frac{1}{N}
   \sum_{i=1}^{N}
   \left(
   f_\phi(x_i,c_i) - \tilde{y}_i
   \right)^2

其中标准化标签为：

.. math::

   \tilde{y}_i
   =
   \frac{y_i-\mu_y}{\sigma_y}

训练后再反标准化得到真实尺度下的预测值。

后验筛选
--------

后验筛选不是直接替代 Pareto，而是在扩散候选池中进行低成本筛选。

当前后验打分：

.. code-block:: text

   proxy_score = - predicted_balance_mean + gamma * prediction_uncertainty

其中 `predicted_balance_mean` 越低越好，`prediction_uncertainty` 用于保留一定探索性。

最终按 `proxy_score` 从高到低选择候选样本。

数学形式为：

.. math::

   s_i
   =
   -\mu_i
   +
   \gamma \sigma_i

其中 :math:`\mu_i` 是代理模型集成预测均值，:math:`\sigma_i` 是集成预测不确定性，:math:`\gamma` 是探索权重。

最终选择：

.. math::

   X_{next}
   =
   \operatorname{TopK}_{x_i \in X_{pool}}(s_i)

这里的 Top-K 是根据代理模型分数进行的筛选，不等同于真实 SUMO 排序。真实效果仍需要后续 SUMO 仿真验证。

Pareto 平衡选择
---------------

Pareto 部分用于真实仿真后的多目标选择，和代理模型筛选是两个不同模块。

原始 Pareto 分数：

.. math::

   q_{raw}
   =
   \operatorname{ParetoRankCrowding}
   \left(
   t_{top},t_{right},t_{bottom},t_{left}
   \right)

平衡惩罚：

.. math::

   p_{balance}
   =
   \exp
   \left(
   -\alpha g_{worst}
   -
   \beta g_{spread}
   \right)

最终选择分数：

.. math::

   q
   =
   \operatorname{Normalize}
   \left(
   q_{raw} \cdot p_{balance}
   \right)

因此：

.. code-block:: text

   pareto_q_raw：未加入方向平衡惩罚前的原始 Pareto 选择分数
   q：加入最差方向和方向差异惩罚后的最终 Pareto 选择分数
