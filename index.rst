PAEDiff 交通信号优化文档
=========================

本文档用于记录当前交通信号优化框架的设计、数据、模型、训练评估和阶段性实验历史。

当前框架结合 SUMO 交通仿真、DQN 信号控制器、条件扩散模型、Pareto 平衡选择和代理模型后验筛选，目标是在降低整体排队的同时保持各方向交通压力均衡。

.. toctree::
   :maxdepth: 2
   :caption: 模型设计

   overview
   data
   architecture
   training
   implementation
   history
