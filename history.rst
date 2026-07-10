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

   本次实验关注训练过程中生成种群的整体质量变化。
   因此不把单个样本或单个 round 作为主要结论，而是按 epoch 汇总种群质量。
   评价指标仍然使用真实 SUMO 仿真后的 balance_score，数值越低表示效果越好。

整体 fitness 趋势：

.. code-block:: text

   mean：当前 epoch 内所有生成样本 balance_score 的平均值。
   median：当前 epoch 内所有生成样本 balance_score 的中位数。
   best：当前 epoch 内 balance_score 最小的样本，即该 epoch 的最优个体。

   由于 balance_score 越低越好：
   mean 下降表示整体种群平均质量变好；
   median 下降表示大多数样本的质量变好；
   best 下降表示当前 epoch 能生成更优的单个个体。

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

   mean 从 37.94 缓慢上升到 38.76，没有下降趋势。
   这说明整体种群平均质量没有变好，反而略微变差。

   median 在 34.4 到 35.2 附近上下波动，没有持续下降。
   这说明大多数样本的质量没有稳定改善。

   best 在 epoch 3 明显下降到 30.36，说明模型可以偶尔生成更优个体。
   但 epoch 4 又回升到 31.07，说明最优个体的改善还不稳定。

   因此当前实验只能说明模型有时能够生成更好的单个样本，
   但还不能证明整个种群生成质量随着训练稳定提升。

tau 分布变化：

.. code-block:: text

   tau 是扩散模型生成的 reward 序列。
   tau_mean 表示当前 epoch 生成 reward 序列的整体平均值。
   tau_std 表示当前 epoch 生成 reward 序列的离散程度。

   代码中的 reward = -queue，因此 reward 越大表示排队惩罚越小。
   tau_mean 上升说明扩散模型生成的 reward 序列整体往更大的奖励值方向移动。
   tau_std 下降说明生成 reward 的波动范围变小，样本分布更集中。

.. image:: images/03_epoch_tau_stats.png
   :alt: epoch 级 tau 统计趋势
   :align: center
   :width: 90%

当前观察：

.. code-block:: text

   tau_mean 整体从较低位置向上移动，说明扩散模型生成的 reward 序列确实在往奖励值更大的方向变化。
   结合 reward = -queue，这个方向从定义上看更接近低排队惩罚。

   tau_std 相比初始 epoch 明显下降，说明生成 reward 的分布变得更集中。
   这代表模型生成的样本不再像初始阶段那样分散，探索范围有所收缩。

   但是 tau_mean 上升和 tau_std 下降没有稳定带来 balance_score 的下降。
   因此当前问题不是扩散模型没有变化，而是 reward 分布变化还没有稳定转化成真实 SUMO 效果提升。

epoch 内部进化变化：

.. code-block:: text

   mean delta = 进化后 mean balance_score - 进化前 mean balance_score。
   best delta = 进化后 best balance_score - 进化前 best balance_score。
   improved ratio = 进化后 balance_score 下降的样本比例。

   由于 balance_score 越低越好：
   delta < 0 表示变好；
   delta > 0 表示变差；
   improved ratio 越高，说明变好的样本比例越大。

.. image:: images/04_epoch_evolution_change.png
   :alt: epoch 内部进化变化
   :align: center
   :width: 90%

当前观察：

.. code-block:: text

   mean delta 大部分位于 0 以上，说明内部进化后平均 balance_score 经常升高。
   这代表平均种群质量在进化后没有稳定变好，部分 epoch 反而变差。

   best delta 在 epoch 2 和 epoch 3 为负，说明最优个体在这些 epoch 中确实被进一步改进。
   但是 best delta 不是每个 epoch 都为负，因此最优样本的改进也不稳定。

   improved ratio 没有稳定超过 0.5，说明超过一半样本变好的情况并不稳定。
   这进一步说明当前进化过程更像是偶尔找到好个体，而不是稳定推动整个种群变好。

高维样本结构可视化：

.. code-block:: text

   前面的折线图主要观察整体结果，例如 mean、median、best 和 delta。
   这些图可以反映一批样本整体有没有变好，
   但是不能直接展示样本内部哪些位置发生了变化。

   因此这里把每轮保存下来的 tau 和 rho 画成热力图。
   图中的前三个子图分别表示 round1、round2、round3。
   右下角是 round3 - round1，用来观察三轮之后哪些位置增大，哪些位置减小。

tau 高维结构：

.. code-block:: text

   tau 表示扩散模型生成的 reward 序列。
   原始 tau 是 sample_count × time_step × direction。
   因为一个 round 中包含多个样本，
   所以这里先对 sample_count 求平均，
   这样就能得到一张 time_step × direction 的图。

   横轴是 direction_index，对应 reward 序列中的不同方向或动作维度。
   纵轴是 time_step，对应时间步。
   颜色表示这个位置上的 tau 平均值。

.. image:: images/tau.png
   :alt: tau 高维结构变化
   :align: center
   :width: 90%

tau 图观察：

.. code-block:: text

   从图上看，round1、round2、round3 的颜色分布并不完全相同。
   这说明扩散模型每一轮生成出来的 tau 存在变化，
   不是三轮都在生成完全相同的 reward 序列。

   右下角的差值图里有红色也有蓝色。
   红色表示 round3 这个位置的 tau 比 round1 更大；
   蓝色表示 round3 这个位置的 tau 比 round1 更小。

   因此目前可以看到，tau 并不是只发生整体平移，
   而是在不同 time_step 和 direction 上都有一些局部变化。
   不过这个图只能说明 tau 本身发生了变化，
   还不能直接说明这些变化一定能让 SUMO 结果变好，
   这个还要和真实仿真后的 balance_score 放在一起看。

rho 高维结构：

.. code-block:: text

   rho 表示状态-动作条件信息。
   当前 rho 的形状为 time_step × feature_index。

   横轴是 feature_index，对应状态-动作条件里的不同特征。
   纵轴是 time_step。
   颜色表示这个时间步下某个特征的 rho 数值。

.. image:: images/rho.png
   :alt: rho 高维结构变化
   :align: center
   :width: 90%

rho 图观察：

.. code-block:: text

   rho 这张图和 tau 不太一样。
   它的横轴只有 24 个特征，所以图上会出现比较明显的竖向特征带。
   这些特征带说明 rho 的变化主要集中在部分 feature 上，
   并不是所有 feature 都有相同程度的变化。

   从 round1、round2、round3 看，主要高值区域大体集中在相近的 feature 上。
   这说明 rho 的主要结构保持相对稳定。

   但是右下角的差值图里也能看到一些红色和蓝色区域，
   说明部分 feature 在某些时间步上仍然发生了变化。

   所以这张图主要说明：
   rho 的整体结构比较稳定；
   但不同轮次之间并不是完全不变，局部特征仍然有调整。

本次结论：

.. code-block:: text

   当前框架已经可以记录并展示 epoch 级别的种群变化。
   从现有结果看，扩散模型生成分布在变化，也能偶尔产生更优个体。
   但是整体种群质量没有随 epoch 稳定提升。

   下一步重点不是继续增加很多新模块，
   而是围绕当前结果中最明显的两个问题进行改进：
   1. 整体 mean 没有下降；
   2. best 偶尔变好，但没有带动整个种群一起变好。

后续计划
--------

.. mermaid::

   graph TD
      trend["当前现象：best 偶尔变好，但 mean 没有稳定下降"]
      mean_plan["改进一：让整体种群 mean 下降"]
      best_plan["改进二：围绕优秀个体进行局部扩展"]
      verify["真实 SUMO 验证 epoch 趋势"]

      trend --> mean_plan
      trend --> best_plan
      mean_plan --> verify
      best_plan --> verify

改进方向说明：

.. code-block:: text

   1. 让整体种群 mean 下降
      这次实验中，best 曾经变好，但 mean 没有下降，说明问题不在于完全找不到好样本，
      而在于大部分生成样本仍然没有被推向更好的区域。

      因此后续要把评价重点从“有没有一个最优样本”改成“整批样本的平均质量有没有提升”。
      具体做法是继续用真实 SUMO 的 balance_score 统计每个 epoch 的 mean、median、best 和 std。
      如果修改后只有 best 下降，而 mean 和 median 不下降，就说明改进仍然只是偶然找到好个体；
      只有 mean 或 median 也开始下降，才能说明生成种群整体质量真的在提高。

      这样改的原因是：最终训练需要的是一批稳定可用的候选种群，
      不是只依赖某一次随机生成出来的单个好样本。

   2. 围绕优秀个体进行局部扩展
      epoch 3 中 best 下降到 30.36，说明当前扩散模型生成空间里确实存在更好的个体。
      但是这个好个体没有带动 mean 下降，说明优秀区域没有被稳定利用起来。

      因此后续可以把已经找到的优秀样本作为局部中心，
      在它附近继续生成或筛选更多候选样本，而不是每次都大范围随机探索。
      目标是让更多样本靠近优秀区域，使 best 的优势逐渐扩散到 median 和 mean。

      这样改的原因是：当前结果已经证明模型偶尔能找到好点，
      下一步更合理的是利用这些好点扩大优秀样本比例，而不是只继续盲目探索。

后续重点：

.. code-block:: text

   当前最关键的问题不是能不能生成一个好样本，
   而是能不能让整个生成种群的分布逐步向好样本区域移动。

   因此后续实验判断标准调整为：
   1. mean 是否随 epoch 下降；
   2. median 是否随 epoch 下降；
   3. best 是否稳定保持或继续下降；
   4. std 是否变小，说明种群不再大范围波动。

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

2026-07-03：进化停滞原因诊断
----------------------------

想回答的问题：

.. code-block:: text

   有时候能出一个更好的样本，但整个种群就是没有一直往好了走。
   这到底是扩散模型的问题，还是代理模型的问题，还是继承机制的问题？

所以我看了下面的四个采样数据并进行了分析：

.. code-block:: text

   1. iteration_stats.csv
      整体种群最后有没有变好？

   2. proxy_surrogate_metrics.csv
      代理模型本身训练得怎么样？

   3. proxy_candidate_selection_*.csv
      代理筛选时有没有真的把候选区分开？

   4. sample_eval_proxy_posterior_*.csv
      被筛出来的样本，真实跑 SUMO 后到底有没有更好？

怎么判断真实后验结果：

.. code-block:: text

   best delta = 最后一代 best balance_score - 第一代 best balance_score
   mean delta = 最后一代 mean balance_score - 第一代 mean balance_score

   balance_score 越低越好：
   delta < 0 说明变好了
   delta = 0 说明基本没动
   delta > 0 说明变差了

.. list-table:: 真实 SUMO 后验结果（epochs 17-19，round 1-3）
   :header-rows: 1

   * - epoch-round
     - best delta
     - mean delta
     - 现象
   * - 17-1
     - 0.0000
     - 0.0000
     - 完全停住
   * - 17-2
     - 0.0000
     - 0.0000
     - 完全停住
   * - 17-3
     - 0.0000
     - -0.0020
     - 只有整体均值轻微变好
   * - 18-1
     - 0.0000
     - +0.2021
     - 最优没变，整体均值变差
   * - 18-2
     - +0.7012
     - -4.4013
     - 整体回缩了，但原本最优个体没有保住
   * - 18-3
     - 0.0000
     - -0.1890
     - 均值略有改善，但最好样本没继续下降
   * - 19-1
     - -0.1657
     - +0.5515
     - 最优个体变好，但整体种群反而变差
   * - 19-2
     - 0.0000
     - 0.0000
     - 完全停住
   * - 19-3
     - 0.0000
     - 0.0000
     - 完全停住

把上面这些整理一下：

.. list-table::
   :header-rows: 1

   * - 指标
     - 数值
     - 说明
   * - 可分析 round 数
     - 9
     - 只统计真正进了进化流程的 round 1-3
   * - best 和 mean 同时变好
     - 0 / 9
     - 没有一次同时带动最优值和整体均值下降
   * - 只有 best 变好
     - 1 / 9
     - 典型是 19-1
   * - 只有 mean 变好
     - 3 / 9
     - 更像是整体回缩，不是稳定保住最好个体
   * - 两者都没变好
     - 5 / 9
     - 经常等于没动或者原地波动

从这些数据来看：

.. code-block:: text

   19-1 里 best 确实下降过，说明系统偶尔能碰到更好的点。
   但好样本出来一次之后，下一轮大部分又回去了，没有带着整批一起变好。

代理筛选为什么没把候选样本拉开：

.. code-block:: text

   每次代理筛选时，先随机生成 192 个候选样本，
   然后根据 predicted_balance_score 和 proxy_score 选出 95 个，
   加上 1 个真实 elite，组成下一代 96 个样本。

.. list-table:: 代理筛选分数离散度（epochs 17-19，round 1-3）
   :header-rows: 1

   * - 指标
     - 数值
     - 说明
   * - 代理筛选文件数
     - 83
     - 一共看了 83 次 posterior candidate selection
   * - predicted span < 1e-3
     - 74 / 83
     - 大多数轮次里，192 个候选分数几乎一样
   * - predicted span = 0
     - 30 / 83
     - 代理直接给所有候选打了一模一样的分数
   * - target_std < 1e-3
     - 21 / 83
     - 训练代理时，真实标签本身波动就很小
   * - 每次保留的真实 elite
     - 1
     - 剩下 95 个主要靠代理筛选

.. list-table:: 各 round 的代理预测离散程度
   :header-rows: 1

   * - epoch-round
     - 平均 predicted span
     - 现象
   * - 17-1
     - 0.000000
     - 完全常数打分
   * - 17-2
     - 0.000025
     - 几乎常数打分
   * - 17-3
     - 0.000001
     - 几乎常数打分
   * - 18-1
     - 0.000053
     - 几乎常数打分
   * - 18-2
     - 0.092432
     - 少数能真正拉开差距的轮次
   * - 18-3
     - 0.000025
     - 几乎常数打分
   * - 19-1
     - 0.000200
     - 仍然很小，区分度很弱
   * - 19-2
     - 0.000000
     - 完全常数打分
   * - 19-3
     - 0.000001
     - 几乎常数打分

从上面数据看：

.. code-block:: text

   代理模型在很多轮次里基本选不出来，192 个候选分数差不多，硬选了 95 个。
   不是"有一点误差"的程度，是大部分情况下代理根本没起筛选作用。

看代码能发现几个原因：

1. 真实 elite 继承太弱

   - 现在的做法是：每次从扩散模型重新生成一整池候选，再从里面筛下一代。
   - 实际上每轮只把 1 个真实 elite 强行放回去，另外 95 个都是重新筛的。
   - 所以好样本的记忆很弱，某一代偶尔变出一个好点，下一代大部分样本又回去了。

2. fine-tune 被重置了

   - 回合内每次迭代都会调 diffusion model 的 fine-tune。
   - 但是 train_diffusion_model 开头会重新加载 BEST_DDP_MODEL_PATH。
   - 这样看起来是连续微调，实际每次从同一个起点重新学，之前的改进没有积累下来。

3. elite 选择目标和最终评价目标不太一样

   - 当前代码里，current_best_sample 是按 q 或者 Pareto 平衡挑的。
   - 但后面真实仿真真正关心的是 balance_score。
   - 这两个目标不完全一样，所以可能出现：
     某个样本在 Pareto 平衡上被认为是当前最好的，
     但它不一定就是 balance_score 最低的那个。
   - 这也会让优秀个体的继承更难稳住。

4. reward 离散化有风险

   - traffic_simulator.py 里，reward 在几处直接转成了 int。
   - 如果不同样本的差异主要体现在小数部分，
     那这些差异在后面学习和回填的时候就被抹掉了。

为什么我觉得问题主要在继承机制，不只是扩散模型的问题：

.. code-block:: text

   这次数据里能看出两件事。

   第一，扩散模型不是完全没变化，前面的 tau 统计和热力图都能看出来。
   第二，系统不是完全找不到好样本，19-1 里 best 确实下降过。

   所以更像是：好样本偶尔出现一次，但没被保留住，也没吸引更多样本往它那边靠。
   问题不是找不到好点，是好点出现了但传不下去。

本次诊断的结论：

.. code-block:: text

   当前流程大致是：

   1. 扩散模型重新抽一大池候选；
   2. 代理模型在很多轮次里打不出差别；
   3. 只保留 1 个真实 elite；
   4. 下一代大部分样本又是重新来的。

   所以 best 偶尔能降一点，但 mean 很难持续下降。
   整体就是"有时能出一个好样本，但种群整体没有稳定变好"。

已经改了对应的代码，正在跑实验，等结果出来再看看。

.. list-table::
   :header-rows: 1

   * - 改了什么
     - 具体内容
   * - 多 elite 继承
     - 用多个真实 elite 做局部变异，不再只保留 1 个
   * - fine-tune 重置
     - 去掉每次从 BEST_DDP_MODEL_PATH 重新加载，让回合内微调可以累积
   * - reward 离散化
     - 去掉 reward 转 int 的操作，保持浮点精度

这次发现的问题：

.. code-block:: text

   不是没有优化信号，是好结果传不下去。

   下一步最要紧的不是加新模块，
   而是先把多 elite 继承、局部变异、连续微调这几个问题解决掉。

2026-07-06：0703 改动后的中途结果
----------------------------------

这次我又看了一次更新后的结果。
原来只看到 epoch 1 第二个 round 的第 4 次进化，
这次解压出来的文件里，round0、round1、round2 都已经有完整的 10 次进化结果。
这里还是要注意，输出文件里的 round 是从 0 开始数的，
所以文件里的 ``round2`` 对应实际说的第三个 round。

这次我主要看了这些文件：

.. code-block:: text

   data/iteration_stats.csv
   data/proxy_surrogate_metrics.csv
   data/samples/sample_eval_ep1_round0.csv
   data/samples/sample_eval_ep1_round1.csv
   data/samples/sample_eval_ep1_round2.csv
   data/samples/proxy_candidate_selection_ep1_round0_iter*.csv
   data/samples/proxy_candidate_selection_ep1_round1_iter*.csv
   data/samples/proxy_candidate_selection_ep1_round2_iter*.csv
   data/samples/sample_eval_proxy_posterior_ep1_round0_iter*.csv
   data/samples/sample_eval_proxy_posterior_ep1_round1_iter*.csv
   data/samples/sample_eval_proxy_posterior_ep1_round2_iter*.csv

另外，``iteration_stats.csv`` 和 ``proxy_surrogate_metrics.csv`` 里已经能看到 round3 的部分记录，
但是这次打包进来的详细 ``sample_eval_proxy_posterior`` 文件只到 round2，
所以我下面主要还是按 round0 到 round2 来分析。

我先看样本来源。
从 ``proxy_candidate_selection`` 文件可以看到，每一轮候选池里还是固定有：

.. code-block:: text

   current_true_balance_elite：8
   diffusion_candidate：96
   elite_local_mutation：96

这说明 0703 加的多 elite 保留和局部变异一直都在跑，
不是只在前几次进化里出现。

把每个 round 的 10 次进化合在一起看，进入下一代的样本来源大概是这样：

.. list-table::
   :header-rows: 1

   * - round
     - 平均每次保留真实好样本
     - 平均每次选中局部变异样本
     - 平均每次选中普通扩散样本
   * - round0
     - 8.0
     - 46.0
     - 42.0
   * - round1
     - 8.0
     - 39.8
     - 48.2
   * - round2
     - 8.0
     - 40.6
     - 47.4

从这里看，真实好样本每次都稳定保留了 8 个。
这个比之前只保留 1 个要稳很多，至少不会很容易把已经找到的好样本丢掉。
局部变异样本也不是偶尔才出现，三个 round 里平均每次大概能选中 40 个左右。
所以现在可以说，局部变异确实参与到了下一代生成里。

然后我看真实 SUMO 的 ``balance_score``。
这个指标越低越好。
三个 round 从 iter1 到 iter10 的变化如下：

.. list-table::
   :header-rows: 1

   * - round
     - mean 变化
     - median 变化
     - best 变化
     - std 变化
   * - round0
     - 37.6485 -> 36.4466，下降 1.2019
     - 34.5529 -> 34.5197，略有下降
     - 30.8307 -> 30.8307，保持不变
     - 6.9491 -> 6.9795，基本不变
   * - round1
     - 39.9893 -> 37.4683，下降 2.5210
     - 34.8044 -> 34.3893，下降 0.4151
     - 33.9550 -> 29.9937，下降 3.9612
     - 9.4545 -> 8.2482，下降 1.2064
   * - round2
     - 39.0773 -> 37.1740，下降 1.9033
     - 34.4379 -> 34.9295，上升 0.4916
     - 33.7640 -> 27.0010，下降 6.7630
     - 9.4953 -> 5.8946，下降 3.6007

这组结果比我之前只看到 round1 iter4 的时候更清楚。
三个 round 的 mean 都下降了，说明整体平均表现确实有往好的方向走。
round1 和 round2 的 best 也明显下降，说明新的流程不只是保住原来的好样本，
也能继续找到更好的单个样本。

不过这里也不是所有指标都一致变好。
round2 的 median 反而从 34.4379 上升到 34.9295，
说明虽然 mean 和 best 都变好了，但中间水平的样本没有一起变好。
所以我现在还不能说整个种群已经稳定整体变好，
更准确地说，是平均值和最优样本有明显改善，但种群中部还不够稳定。

最终每个 round 保存下来的种群结果是：

.. list-table::
   :header-rows: 1

   * - round
     - balance mean
     - balance median
     - balance best
     - balance std
     - worst_dir mean
     - dir_spread mean
   * - round0
     - 36.4466
     - 34.5197
     - 30.8307
     - 6.9795
     - 36.8985
     - 4.0020
   * - round1
     - 37.4683
     - 34.3893
     - 29.9937
     - 8.2482
     - 38.4021
     - 4.8654
   * - round2
     - 37.1740
     - 34.9295
     - 27.0010
     - 5.8946
     - 37.6886
     - 5.3933

从最终结果看，round2 的 best 已经到了 27.0010，
这是目前这几轮里最好的一个值。
同时 round2 的 std 也降到了 5.8946，说明这一轮的样本波动比前面小。
但是 round2 的 ``dir_spread mean`` 比 round0、round1 都高，
说明方向均衡这块还没有同步变好。
这一点后面还需要继续盯着看，不能只看 ``balance_score`` 的最优值。

代理模型这边，我觉得问题还是在。
如果看代理模型给候选样本打出的分数差距，三个 round 的平均差距是：

.. list-table::
   :header-rows: 1

   * - round
     - predicted_balance_score 平均差值
     - predicted_balance_score 最大差值
     - proxy_score 平均差值
   * - round0
     - 0.003152
     - 0.020969
     - 0.002945
   * - round1
     - 0.000915
     - 0.002403
     - 0.000926
   * - round2
     - 0.000439
     - 0.001667
     - 0.000454

这些数还是比较小。
尤其 round1 和 round2，大多数时候代理模型给不同候选样本的分数差距仍然很小。
所以我认为目前的改善主要还是来自两个方面：
一是 8 个真实好样本被稳定保留，二是在好样本附近生成了一批局部变异样本。
代理模型现在可能参与了筛选，但还不能说它已经很会区分候选好坏。

代理训练本身没有明显异常。
每个 round 最后一次的 ``target_std`` 和误差大概是：

.. list-table::
   :header-rows: 1

   * - round
     - target_std
     - val_mae
     - val_rmse
   * - round0 iter10
     - 6.7916
     - 4.4868
     - 6.7936
   * - round1 iter10
     - 8.8545
     - 6.4731
     - 9.0934
   * - round2 iter10
     - 8.5938
     - 6.0344
     - 8.5081
   * - round3 iter9
     - 7.5873
     - 5.1287
     - 7.4903

这里的 ``target_std`` 没有塌到接近 0，说明标签本身还有波动。
但是 ``val_mae`` 仍然不低，所以代理模型目前还是只能作为辅助，
不能把它当成很可靠的排序器。

我目前的判断是：

.. code-block:: text

   0703 的改动已经不是只在机制上生效，真实结果里也能看到一些改善。
   round0、round1、round2 的 balance mean 都下降了。
   round1 和 round2 的 best 也明显变好，说明好样本不仅被保住了，还能继续往下找。

   但是这个改善还不完全稳定。
   round2 的 median 没有下降，dir_spread mean 也偏高，说明种群中部和方向均衡还没有一起变好。

   代理模型的分数差距仍然很小。
   目前更像是多 elite 保留和局部变异先起到了作用，代理排序能力还没有根本变强。

所以我现在会把这次结果理解为：
0703 的改动确实缓解了“好样本传不下去”的问题，
并且已经开始让 mean 和 best 往好的方向走；
但如果要说明整个后验筛选稳定有效，
还需要继续看后面 round 和后面 epoch 里 median、dir_spread、worst_dir_queue 能不能一起改善。

后面我会继续重点看：

.. code-block:: text

   1. 后续 round 里 balance mean 是否还能继续下降；
   2. median 是否也能跟着下降，而不是只有 best 变好；
   3. dir_spread mean 和 worst_dir mean 是否能同步下降；
   4. elite_local_mutation 被选中的数量是否继续稳定在较高比例；
   5. 代理模型给候选样本打出的分数差距是否能变大。

2026-07-07：下一步实验方向
--------------------------

后面我准备补一个人为控制变异方向的实验。
这个实验我想看的不是再去调某个参数大一点还是小一点，
而是直接看变异出来的 reward 序列本身：
如果我手动把它往一个更像好样本的方向改，
真实 SUMO 结果会不会也跟着变好。

我一开始想到的是按时间分块来改。
因为当前 reward 序列是 ``1000 x 4`` 的时间序列，
不同时间段的交通状态不一样。
前期可能还没有堵起来，后期车辆积累以后 reward 往往会变差。
所以我最开始的想法是把 1000 个 time step 分成几个时间段，
分别比较好样本和差样本在每个时间段里的 reward 差异，
然后按这些差异去人为改 reward。

大致想法是：

.. code-block:: text

   1. 先用真实 SUMO 的 balance_score 选出好样本和差样本；
   2. 分别看每个时间段里，好样本和差样本的 reward 差在哪里；
   3. 把这个差异当成人为变异方向；
   4. 让普通样本沿这个方向变一下，再送回 SUMO 验证。

我觉得这个思路的优点是比较直观。
它不是随便加随机噪声，而是根据真实跑出来的好样本和差样本，
手动构造一个“从差样本往好样本靠”的方向。

但是后面我又想到一个问题：
reward 序列本身是连续变化的，如果按时间块硬切开，
每一段都用不同的改法，段和段之间可能会突然跳一下。
比如前 100 个时间步用一种方向，后 100 个时间步突然换成另一种方向，
这样 reward 曲线就可能不连续。

我觉得这个问题对当前任务可能会有影响。
因为后面 DQN 会把 reward 序列转成经验来学习。
如果人为制造了明显的时间跳变，
那最后 SUMO 结果变好或者变差，就不一定能说明变异方向本身有效，
也可能只是 reward 序列被改得不自然了。

所以后面我更倾向于不做硬分块，而是做一个整体连续的变异。
具体来说，还是先比较好样本和差样本，
但不是每个时间块单独换一个方向，
而是得到整条时间序列上的差异以后，再沿时间方向做平滑。
这样变异方向还是随时间变化的，但变化会比较连续。

可以写成：

.. code-block:: text

   raw_direction[t, d]
   =
   mean(good_tau[:, t, d]) - mean(bad_tau[:, t, d])

   smooth_direction[t, d]
   =
   temporal_smooth(raw_direction[t, d])

   tau_mut[t, d]
   =
   tau_parent[t, d] + alpha * w(t) * smooth_direction[t, d]

这里的 ``w(t)`` 可以理解成一个随时间慢慢变化的权重。
这样仍然能考虑前期和后期交通状态不一样，
但不会像硬分块那样在边界处突然改变 reward。

我目前对这两个想法的理解是：

.. code-block:: text

   分块变异比较容易解释不同时间段发生了什么，
   但可能让 reward 序列在时间上断开；

   连续变异不那么容易出现突变，
   更适合当前这种随时间变化的 reward_sequence。

所以我后续优先考虑连续的 good-bad 方向变异：

.. code-block:: text

   1. 当前种群真实跑 SUMO，得到 balance_score；
   2. 选出 balance_score 最低的一批样本作为好样本；
   3. 选出 balance_score 最高的一批样本作为差样本；
   4. 计算好样本和差样本在整条 reward 序列上的差异；
   5. 对这个差异沿时间方向做平滑，避免 reward 曲线突然断开；
   6. 用平滑后的方向生成手工变异样本；
   7. 再用 SUMO 看 balance_score 的平均值、中位数和最优值有没有下降。

为了确认这个方向不是碰巧有效，我还可以做一个反向对照：

.. code-block:: text

   正向：tau_pos = tau_parent + alpha * smooth_direction
   反向：tau_neg = tau_parent - alpha * smooth_direction

如果正向扰动后 ``balance_score`` 下降，而反向扰动后变差，
就能更清楚地说明这个手工构造的变异方向确实有用。

所以这一步我最想看清楚的是：
reward 序列的变异方向能不能被人为控制，
以及这种人为控制能不能真正反映到 SUMO 的交通指标上。

2026-07-10：人工扰动方向实验
----------------------------

这次把前面设计的人为扰动方向实验实际运行了一次，
而且分成两步来看：
先看离线方向是否合理，
再看这个方向能不能传到真实 SUMO 指标上。

这次使用的是 ``epoch 1`` 的 ``round3``。
这里文件里的 ``round3`` 是从 0 开始计数的，
对应第 4 个 round。

先用 ``sample_eval_ep1_round3.csv`` 按真实 ``balance_score`` 做了一次排序：

.. code-block:: text

   前 5 个作为好样本
   中间 5 个作为被扰动样本
   后 5 个作为差样本参考

这次正向扰动做得比较直接，没有再按时间分块去修改。
具体做法是：

.. code-block:: text

   1. 先按真实 balance_score 排序；
   2. 取前 5 个作为好样本；
   3. 取中间 5 个作为原样本；
   4. 对每个原样本，在前 5 个好样本里找 reward 序列距离最近的那个；
   5. 让原样本朝这个最近的好样本靠一点。

也就是说，这次不是让所有样本都朝同一个固定模板去靠，
而是每个原样本先找一个离自己最近的好样本，
然后再沿着这个局部方向做扰动。

然后先做了最简单的一版人工扰动：

.. code-block:: text

   tau_pos = tau_parent + 0.1 * (tau_good - tau_parent)
   tau_neg = tau_parent - 0.1 * (tau_good - tau_parent)

也就是先不做时间分块，
而是让一个普通样本先朝更好的样本靠一点，
再做一个反方向的对照。

这里的 ``0.1`` 是手动设定的步长，
可以直接理解成：
先让原样本朝更好的样本靠 10%，
再看这个小步扰动本身是不是会把结果往好的方向推。

先看了离线结果。
这一步主要不是直接看 SUMO，
而是看扰动后的 reward 序列是不是整体更靠近好样本区域，
以及序列本身有没有被改得过于生硬。

离线这一步的结果大概是：

.. code-block:: text

   正向扰动后，更靠近好样本中心：5 / 5
   反向扰动后，更远离好样本中心：5 / 5
   正向扰动后，更远离差样本中心：4 / 5
   正向扰动后，到好样本中心的平均距离下降比例：0.0544
   正向扰动后，远离差样本中心的平均比例：0.0502

同时也检查了时间连续性：

.. code-block:: text

   正向扰动后，一阶差分均值变化：-0.1529

这一步可以理解为：
人工扰动至少已经不是随意改动了，
而是真的能把样本往好样本群体那边推一点，
而且暂时没有看到特别明显的 reward 时间断裂。

不过只做离线还不够，
所以后面又补了一个很小的在线验证。
这一步没有直接改主训练流程，
而是单独做了一次小实验：
先重新采一条共享轨迹，
然后在同一个初始网络状态下，
分别评估”原样本 / 正向扰动 / 反向扰动”三组样本的真实 SUMO 结果。

在线小实验的结果是：

.. code-block:: text

   正向扰动后，balance_score 比原样本更好：3 / 5
   反向扰动后，balance_score 比原样本更差：1 / 5
   正向扰动后，worst_dir_queue 比原样本更好：3 / 5
   反向扰动后，worst_dir_queue 比原样本更差：1 / 5
   正向扰动后，dir_spread 比原样本更好：3 / 5
   反向扰动后，dir_spread 比原样本更差：1 / 5

三组样本的均值是：

.. code-block:: text

   原样本平均 balance_score: 44.0033
   正向扰动平均 balance_score: 34.4701
   反向扰动平均 balance_score: 40.8895

这里有两点比较关键。

第一个点是，
人工方向的变化确实已经能传到真实 SUMO 指标上，
不是只在离线距离上看起来变了，
放到真实 SUMO 里也不是完全没有反应。

第二个点是，
这个方向现在还不算特别稳。
有些 pair 里正向扰动确实比原样本好，
但也有一些 pair 里反向扰动反而更好，
说明当前”朝单个最好样本靠近”的方向设计还比较粗。

另外还注意到，
``pair 2`` 和 ``pair 4`` 里，
原样本自己就很差，
而正向扰动和反向扰动都比原样本好很多。
这提示另一种可能：
有一部分当前样本本身就落在一个很差而且不稳的位置，
这时候不管朝哪个方向稍微离开一点，
都有可能先变好。

所以对这次实验的理解是：

.. code-block:: text

   1. 奖励序列空间里确实存在一个可以人为控制的改进方向；
   2. 这个方向不只是离线有意义，已经能反映到真实 SUMO 指标上；
   3. 但当前”朝单个最好样本靠近”的人工方向还不够稳；
   4. 这也说明原本的自动变异还没有稳定利用这个局部好方向。

换句话说，
不会把这次结果理解成”整个框架已经完全没问题”，
而是会理解成：
整体框架的大方向没有偏，
但当前自动变异的方向性还不够强，
它还没有稳定地把样本往这种局部好方向推过去。

为了继续把这个问题看清楚，
在训练代码中补了一个单独的实验入口，
不改默认训练流程，
专门拿同一条共享轨迹去比较：

.. code-block:: text

   原样本
   人工正向扰动
   当前自动变异

后续的目标是：
如果人工方向已经能在一些样本上带来改进，
那当前自动变异在同样条件下为什么没有稳定做到这一点。
