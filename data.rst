数据与预处理
============

主要样本
--------

state_action_condition   当前样本对应的状态-动作条件，是扩散模型和代理模型的条件输入。

reward_sequence   扩散模型生成的候选 reward 序列，是代理模型输入的一部分。

true_raw_fitness   SUMO 仿真后得到的原始真实适应度。

balance_score   SUMO 仿真后得到的平衡评价指标，数值越低表示样本越好。

epoch   样本所属的训练 epoch。

round   样本所属的训练轮次。

iter_idx   样本所属的进化迭代编号。

sample_id   样本在当前种群或当前批次中的编号。

sample_source   样本来源，例如进化过程种群或最终种群。

sample_npz_path   完整样本数组所在的压缩文件路径。

npz_reward_key   压缩文件中 reward 序列对应的键名。

npz_reward_index   当前样本在压缩文件 reward 数组中的索引。

npz_condition_key   压缩文件中条件信息对应的键名。

交通评价字段
------------

tail_top   顶部方向的尾部排队指标。

tail_right   右侧方向的尾部排队指标。

tail_bottom   底部方向的尾部排队指标。

tail_left   左侧方向的尾部排队指标。

avg_inner_sum   内部区域的平均拥堵或排队汇总指标。

avg_tail_robust   尾部方向的稳健平均排队指标。

worst_tail_dir_avg   四个尾部方向中最差方向的平均排队指标。

q   加入方向平衡惩罚后的最终 Pareto 选择分数。

pareto_q_raw   未加入方向平衡惩罚前的原始 Pareto 选择分数。

worst_dir_queue   四个方向中最差方向的队列评价值。

dir_spread   四个方向队列评价值之间的差异。

q_balance_penalty   根据最差方向和方向差异计算的平衡惩罚系数。

预处理
------

reward 序列会根据历史数据进行归一化，使扩散模型训练数据和噪声尺度更加匹配。

代理模型训练时会对真实 `balance_score` 做标准化：

.. code-block:: text

   target = (balance_score - target_mean) / target_std

验证时再反标准化计算 MAE 和 RMSE。
