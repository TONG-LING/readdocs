任务概述
========

研究目标
--------

本项目面向多路口交通信号控制任务，使用 SUMO 构建交通仿真环境，通过强化学习和生成式搜索共同优化信号控制策略。

核心目标是降低交通拥堵，并避免只优化某一个方向而牺牲其他方向。

优化目标可以概括为：

.. code-block:: text

   整体队列更低
   最差方向队列更低
   四个方向更加均衡

整体流程
--------

当前框架包含五个核心部分：

.. code-block:: text

   SUMO 仿真环境
   DQN 交通信号控制器
   条件扩散模型
   Pareto 平衡选择
   代理模型后验筛选

整体思路是先使用扩散模型生成候选 reward 序列，再通过 SUMO 真实评估得到交通指标，并利用这些真实样本训练代理模型。代理模型用于低成本预测候选样本的真实表现，从而辅助后验筛选下一代样本。

总体闭环如下：

.. mermaid::

   graph LR
      condition["交通状态条件 c"]
      diffusion["扩散模型 p_theta(x|c)"]
      pool["候选样本池 {x_i}"]
      proxy["代理模型 f_phi(x_i,c)"]
      posterior["后验筛选"]
      sumo["SUMO 真实仿真"]
      dataset["真实样本数据集 D"]

      condition --> diffusion
      diffusion --> pool
      pool --> proxy
      proxy --> posterior
      posterior --> sumo
      sumo --> dataset
      dataset --> proxy
      dataset --> diffusion

其中：

.. math::

   x_i \sim p_\theta(x \mid c)

表示扩散模型在当前条件 :math:`c` 下生成第 :math:`i` 个候选 reward 序列。

.. math::

   y_i = \operatorname{SUMO}(x_i, c)

表示候选样本经过 SUMO 仿真后得到真实评价，当前主要使用 :math:`y_i=\text{balance\_score}_i`。

当前好样本定义
--------------

当前任务中的好样本不是指 reward 序列本身形状好看，而是指经过 SUMO 真实仿真后交通表现更好。

好样本通常满足：

.. code-block:: text

   balance_score 较低
   worst_dir_queue 较低
   dir_spread 较低
   四个方向 tail_top / tail_right / tail_bottom / tail_left 都不能极端拥堵
   q 较高

其中 `q` 是加入方向平衡惩罚后的最终 Pareto 选择分数。

平衡评价指标
------------

当前 `balance_score` 将四个方向的平均排队和方向差异合并为一个标量：

.. math::

   \text{balance\_score}
   =
   \operatorname{mean}(t_{top}, t_{right}, t_{bottom}, t_{left})
   +
   \lambda
   \left(
   \max(t)-\min(t)
   \right)

其中 :math:`t` 表示四个方向的尾部排队指标，:math:`\lambda` 是方向差异惩罚权重。

这个指标的含义是：

.. code-block:: text

   平均队列要低
   四个方向差异也要小

因此，一个方向特别堵的样本即使平均值不高，也会被惩罚。
