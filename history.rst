历史
====

.. mermaid::

   graph TD
      root["PAEDiff 交通信号优化"]

      base["基础框架"]
      posterior["后验设计"]
      proxy["代理模型"]
      assess["效果评估"]
      next["后续改进"]

      root --> base
      root --> posterior
      root --> proxy
      root --> assess
      root --> next

      base --> base1["DQN + SUMO"]
      base --> base2["条件扩散模型生成 reward 序列"]
      base --> base3["Pareto 平衡选择"]

      posterior --> p1["原始尝试：-MSE 作为似然"]
      posterior --> p2["问题：-MSE 不是仿真后的真实似然"]
      posterior --> p3["调整：扩散模型作为先验"]
      posterior --> p4["调整：代理模型提供低成本似然/效用"]

      proxy --> pr1["输入：reward_sequence + condition"]
      proxy --> pr2["输出：predicted_balance_score"]
      proxy --> pr3["结构：统计池化 + MLP"]
      proxy --> pr4["损失：MSE 回归真实 balance_score"]
      proxy --> pr5["样本：proxy_surrogate_train.csv"]

      assess --> e1["代理诊断：MAE 接近标签标准差"]
      assess --> e2["排序诊断：Spearman 接近 0"]
      assess --> e3["后验诊断：22 轮中 11 轮变好，11 轮变差"]

      next --> n1["排序学习"]
      next --> n2["多目标拆分预测"]
      next --> n3["优秀区域局部采样"]
      next --> n4["保留不确定样本送 SUMO"]
      next --> n5["单独拆出代理模型训练"]
      next --> n6["检查扩散候选池质量"]
      next --> n7["构造好/中/差样本"]

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

后续计划
--------

.. mermaid::

   graph TD
      improve["代理模型后续改进"]

      rank["1. 排序学习"]
      pair["构造样本对：判断 A 是否优于 B"]
      rankloss["目标从精确估分转为排对顺序"]

      multi["2. 多目标拆分预测"]
      q["预测综合队列 q"]
      worst["预测最差方向 worst_dir_queue"]
      spread["预测方向差异 dir_spread"]
      penalty["预测平衡惩罚 q_balance_penalty"]
      combine["最后再合成最终评分"]

      sample["3. 优化样本采样"]
      local["在优秀样本附近局部采样"]
      uncertain["保留不确定性高的样本送 SUMO"]
      avoid["避免过早过滤潜在优秀样本"]

      separate["4. 单独拆出代理模型训练"]
      fixeddiff["先固定训练好的扩散模型"]
      collect["配合 SUMO 单独收集代理训练数据"]
      observe["输出更多 CSV 观察代理模型效果"]

      dataissue["5. 检查样本与评价定义"]
      agg["检查四方向聚合方式是否压缩标签波动"]
      manual["尝试构造好/中/差样本"]
      pool["检查扩散候选池是否包含高质量样本"]

      verify["真实 SUMO 验证"]

      improve --> rank
      rank --> pair
      rank --> rankloss
      rank --> verify

      improve --> multi
      multi --> q
      multi --> worst
      multi --> spread
      multi --> penalty
      multi --> combine
      combine --> verify

      improve --> sample
      sample --> local
      sample --> uncertain
      sample --> avoid
      sample --> verify

      improve --> separate
      separate --> fixeddiff
      separate --> collect
      separate --> observe
      separate --> verify

      improve --> dataissue
      dataissue --> agg
      dataissue --> manual
      dataissue --> pool
      dataissue --> verify

后续重点：

.. code-block:: text

   1. 不只看预测数值误差，还要看排序是否正确。
   2. 不只判断代理模型，还要判断扩散候选池质量。
   3. 不只预测单一综合指标，也尝试拆成多个子目标预测。
   4. 不完全相信代理模型，对不确定样本保留一部分真实 SUMO 验证。
   5. 单独拆出代理模型训练流程，降低调试成本。

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
