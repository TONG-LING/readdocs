训练与评估
==========

训练流程
--------

当前训练流程如下：

.. mermaid::

   graph TD
      start["开始当前 epoch / round"]
      condition["构造 state_action_condition"]
      generate["扩散模型生成初始种群"]
      eval["SUMO 真实评估每个样本"]
      save["保存样本与真实标签"]
      pareto["计算 Pareto q 与平衡惩罚"]
      best["选择当前最优和历史最优"]
      finetune["使用最优样本微调扩散模型"]
      proxytrain["训练代理模型"]
      posterior["代理后验筛选下一代"]
      loop["进入下一轮迭代"]

      start --> condition
      condition --> generate
      generate --> eval
      eval --> save
      eval --> pareto
      pareto --> best
      best --> finetune
      save --> proxytrain
      finetune --> posterior
      proxytrain --> posterior
      posterior --> loop

训练数据流
----------

.. mermaid::

   sequenceDiagram
      participant Diff as 扩散模型
      participant Proxy as 代理模型
      participant SUMO as SUMO 仿真
      participant Pareto as Pareto 选择
      participant Data as 样本库

      Diff->>SUMO: 生成候选 reward 序列并送入真实评估
      SUMO->>Data: 保存 balance_score 和方向队列指标
      Data->>Proxy: 使用真实样本训练代理模型
      Proxy->>Diff: 对扩散候选池进行低成本评分
      Proxy->>Pareto: 提供后验筛选候选
      Pareto->>Diff: 保留真实 elite 并指导下一代

SUMO 评估
---------

每个候选样本都会被送入 SUMO 真实仿真，得到 `balance_score`、四方向尾部队列、最差方向队列和方向差异等指标。

真实评估结果保存时按 `sample_id` 排序，保证 CSV 行与当前种群数组位置对应。

代理模型训练
------------

代理模型使用已经经过 SUMO 仿真的样本进行训练。

训练数据：

.. code-block:: text

   reward_sequence
   state_action_condition
   balance_score

训练目标：

.. code-block:: text

   预测真实 SUMO 后的 balance_score

评估指标：

.. code-block:: text

   MAE
   RMSE
   预测排序与真实排序的 Spearman 相关

误差指标定义
------------

MAE 表示预测值平均错多少：

.. math::

   \operatorname{MAE}
   =
   \frac{1}{N}
   \sum_{i=1}^{N}
   |\hat{y}_i-y_i|

RMSE 会更重视较大的预测错误：

.. math::

   \operatorname{RMSE}
   =
   \sqrt{
   \frac{1}{N}
   \sum_{i=1}^{N}
   (\hat{y}_i-y_i)^2
   }

标签标准差表示真实 `balance_score` 自身的波动幅度：

.. math::

   \sigma_y
   =
   \sqrt{
   \frac{1}{N}
   \sum_{i=1}^{N}
   (y_i-\bar{y})^2
   }

当 :math:`\operatorname{MAE}/\sigma_y` 接近 1 时，表示模型平均预测误差已经接近真实标签自身的波动幅度。

排序相关性
----------

后验筛选更关心候选样本之间的相对好坏，因此需要检查预测排序和真实排序是否一致。

Spearman 相关可以写成：

.. math::

   \rho
   =
   \operatorname{corr}
   \left(
   \operatorname{rank}(\hat{y}),
   \operatorname{rank}(y)
   \right)

含义：

.. code-block:: text

   rho 接近 1：预测排序和真实排序基本一致
   rho 接近 0：预测排序和真实排序基本无关
   rho 接近 -1：预测排序和真实排序基本相反

当前 Spearman 接近 0，说明代理模型输出的分数暂时不能可靠支撑后验排序。

当前诊断结果
------------

代理模型训练样本记录数：3072

最后一次训练样本数：512

最后一次验证样本数：128

验证 MAE：8.7898

真实标签标准差：8.8851

MAE / 标签标准差：0.9893

预测值与真实值 Spearman 相关：0.0508

proxy_score 与真实值 Spearman 相关：-0.0424

当前判断
--------

当前代理模型流程已经跑通，但预测误差较大，排序相关性较弱。

这说明代理模型输出的分数暂时还不能稳定支撑后验筛选。

后验效果诊断
------------

可评估后验轮数：22

平均表现变好的轮数：11 / 22

最优样本变好的轮数：7 / 22

平均表现变差的轮数：11 / 22

当前判断：后验流程已经建立，但当前代理模型还没有提供稳定有效的筛选依据。

后验有效性的判断方式
--------------------

一次后验筛选是否有效，主要看筛选后的真实 SUMO 结果：

.. math::

   \Delta_{mean}
   =
   \operatorname{mean}(y_{after})
   -
   \operatorname{mean}(y_{before})

如果 :math:`\Delta_{mean}<0`，说明筛选后种群平均真实表现变好。

最优样本变化：

.. math::

   \Delta_{best}
   =
   \min(y_{after})
   -
   \min(y_{before})

如果 :math:`\Delta_{best}<0`，说明筛选后真实最优样本变好。

当前 22 轮实验中，平均表现变好和变差各占一半，因此后验效果还不稳定。
