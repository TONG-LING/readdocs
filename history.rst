历史
====

.. mermaid::

   graph TD
      base["基础框架"]
      problem["问题发现"]
      proxy["当前方案"]
      diagnose["当前诊断"]
      next["后续改进方向"]

      rank["排序学习"]
      multi["多目标拆分预测"]
      sample["样本优化"]
      separate["单独验证"]

      base --> problem
      problem --> proxy
      proxy --> diagnose
      diagnose --> next
      next --> rank
      next --> multi
      next --> sample
      next --> separate

图中节点说明：

.. code-block:: text

   基础框架：DQN + SUMO + 条件扩散模型 + Pareto 平衡选择
   问题发现：原始 -MSE 不能作为仿真后的真实似然
   当前方案：代理模型预测真实 SUMO 后的 balance_score
   当前诊断：MAE 接近标签标准差，Spearman 接近 0
   后续改进方向：排序学习、多目标拆分预测、样本优化、单独验证

2026-06-13：清理原始后验设计
----------------------------

主要内容：

.. code-block:: text

   移除旧的 -MSE 后验逻辑
   明确似然应来自真实 SUMO 评价或代理模型对真实评价的估计
   保留扩散模型作为候选样本先验

当前理解：

.. code-block:: text

   先验：扩散模型生成候选 reward 序列
   真实评价：SUMO 仿真得到 balance_score
   代理似然：代理模型预测真实 balance_score
   后验筛选：根据 proxy_score 选择下一代候选

2026-06-17：加入代理模型诊断
----------------------------

主要内容：

.. code-block:: text

   增加代理模型训练样本保存
   增加代理模型训练指标记录
   增加代理模型质量检测脚本
   增加后验效果检测脚本

代理模型当前结果：

.. code-block:: text

   代理模型训练样本记录数：3072
   最后一次训练样本数：512
   最后一次验证样本数：128
   验证 MAE：8.7898
   真实标签标准差：8.8851
   MAE / 标签标准差：0.9893
   预测值与真实值 Spearman 相关：0.0508
   proxy_score 与真实值 Spearman 相关：-0.0424

当前判断：

.. code-block:: text

   代理模型已经完成训练流程，但预测误差较大，排序相关性较弱。
   当前代理模型还不能稳定支撑后验筛选。

2026-06-17：后验筛选效果评估
----------------------------

主要内容：

.. code-block:: text

   检查代理后验筛选后的真实 SUMO 表现是否变好
   对比筛选前后 balance_score 的 mean 和 best
   检查代理预测排序与真实排序是否一致

当前结果：

.. code-block:: text

   可评估后验轮数：22
   平均表现变好的轮数：11 / 22
   最优样本变好的轮数：7 / 22
   平均表现变差的轮数：11 / 22

当前判断：

.. code-block:: text

   后验流程已经跑通，但效果不稳定。
   当前问题更可能来自代理模型预测和排序能力不足。
   也需要进一步检查扩散模型候选池本身是否包含足够好的样本。

2026-06-25：整体种群生成趋势评估
--------------------------------

本次实验目的：

.. code-block:: text

   老师要求观察训练过程整体是否越来越好。
   因此本次不把单个 round 作为主要展示对象，而是按 epoch 汇总种群质量。
   评价指标仍然使用真实 SUMO 仿真后的 balance_score，数值越低表示效果越好。

整体 fitness 趋势：

.. image:: images/01_epoch_fitness_trend.png
   :alt: epoch 级种群 fitness 趋势
   :align: center
   :width: 90%

实验结果：

.. code-block:: text

   epoch 1：mean = 37.9372，median = 34.5929，best = 33.2110
   epoch 2：mean = 37.9439，median = 35.1833，best = 33.5855
   epoch 3：mean = 38.3249，median = 34.4186，best = 30.3600
   epoch 4：mean = 38.7608，median = 34.8174，best = 31.0725

趋势判断：

.. code-block:: text

   从 mean 看，整体种群平均 fitness 没有稳定下降，反而从 37.94 上升到 38.76。
   从 median 看，中位数在 34.4 到 35.2 附近波动，也没有稳定变好。
   从 best 看，epoch 3 出现过更优个体，best 从 33.21 降到 30.36，但 epoch 4 又回升到 31.07。

   因此当前实验只能说明模型有时能够生成更好的单个样本，
   但还不能证明整个种群生成质量随着训练稳定提升。

tau 分布变化：

.. image:: images/03_epoch_tau_stats.png
   :alt: epoch 级 tau 统计趋势
   :align: center
   :width: 90%

当前观察：

.. code-block:: text

   tau_mean 和 tau_std 随 epoch 有明显变化。
   这说明扩散模型生成的 reward 序列分布确实在变动。
   但是 tau 的变化本身不能直接证明效果变好，最终仍然要看真实 SUMO fitness。

epoch 内部进化变化：

.. image:: images/04_epoch_evolution_change.png
   :alt: epoch 内部进化变化
   :align: center
   :width: 90%

当前观察：

.. code-block:: text

   mean delta 大部分为正，表示平均 fitness 在内部进化后经常变差。
   best delta 在 epoch 2 和 epoch 3 为负，说明最优个体有时会被改进。
   improved ratio 没有稳定超过 0.5，说明大部分样本并没有稳定改善。

本次结论：

.. code-block:: text

   当前框架已经可以记录并展示 epoch 级别的种群变化。
   从现有结果看，扩散模型生成分布在变化，也能偶尔产生更优个体。
   但是整体种群质量没有随 epoch 稳定提升。

   下一步不应只看是否出现最优样本，而应重点提高整体种群分布质量：
   1. 让训练数据中优秀样本比例更高；
   2. 改进代理模型或筛选方式，避免平均种群质量被拉差；
   3. 继续用真实 SUMO 的 balance_score 作为最终判断依据。

后续计划
--------

.. mermaid::

   graph TD
      trend["当前现象：best 偶尔变好，但 mean 没有稳定下降"]
      mean_plan["目标一：提高整体种群平均质量"]
      best_plan["目标二：保留并扩展优秀个体附近区域"]
      tau_plan["目标三：约束 tau 分布漂移方向"]
      proxy_plan["目标四：让代理模型服务于种群筛选"]
      verify["真实 SUMO 验证 epoch 趋势"]

      trend --> mean_plan
      trend --> best_plan
      trend --> tau_plan
      mean_plan --> proxy_plan
      best_plan --> proxy_plan
      tau_plan --> proxy_plan
      proxy_plan --> verify

改进方向说明：

.. code-block:: text

   1. 提高整体种群平均质量
      当前 mean 没有随 epoch 下降，说明不是整个种群都在变好。
      后续不能只关注 best，而要记录每个 epoch 的 mean、median、best、std。
      如果 mean 不下降，即使偶尔出现好样本，也不能说明生成分布整体有效。

   2. 保留并扩展优秀个体附近区域
      epoch 3 出现 best = 30.36，说明候选空间中确实可能存在好样本。
      后续应把这类优秀样本作为局部中心，在其附近生成更多候选。
      目标不是只保存一个最优样本，而是让更多样本靠近优秀区域，从而拉低 mean。

   3. 约束 tau 分布漂移方向
      tau_mean 和 tau_std 变化明显，但 fitness 没有同步稳定变好。
      这说明 tau 分布在动，但移动方向不一定对应更优交通控制效果。
      后续需要比较优秀样本 tau 与普通样本 tau 的差异，避免扩散模型盲目漂移。

   4. 让代理模型服务于整体种群筛选
      代理模型不应只追求预测分数准确，而要能把会拉低 mean 的候选提前筛出来。
      后续可以优先做排序学习，让代理模型判断两个候选谁更好。
      同时保留一部分代理模型不确定的样本送入真实 SUMO，避免错过潜在好样本。

   5. 用真实 SUMO 验证 epoch 趋势
      每次修改后都继续画 epoch 级趋势图。
      只有当 mean、median、best 至少两个指标稳定下降时，才认为整体种群生成质量变好。

后续重点：

.. code-block:: text

   当前最关键的问题不是能不能生成一个好样本，
   而是能不能让整个生成种群的分布逐步向好样本区域移动。

   因此后续实验判断标准调整为：
   1. mean 是否随 epoch 下降；
   2. median 是否随 epoch 下降；
   3. best 是否稳定保持或继续下降；
   4. std 是否变小，说明种群不再大范围波动；
   5. tau 分布变化是否能对应 fitness 改善。

当前核心公式
------------

扩散先验：

.. math::

   x_i \sim p_\theta(x \mid c)

SUMO 真实评价：

.. math::

   y_i = \operatorname{SUMO}(x_i,c)

代理模型：

.. math::

   \hat{y}_i = f_\phi(x_i,c)

后验筛选分数：

.. math::

   s_i = -\mu_i + \gamma\sigma_i

下一代选择：

.. math::

   X_{next} = \operatorname{TopK}(s_i)

当前需要解决的问题是让 :math:`s_i` 对真实 SUMO 表现具有稳定排序能力。
