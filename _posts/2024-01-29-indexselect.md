---
title: 'pytorch坑：torch.index_select()导致结果无法复现'
date: 2024-01-29
permalink: /posts/2024/01/indexselect/
tags:
  - pytorch
  - torch.index_select()
  - random seed
---

[原文链接](https://ieeexplore.ieee.org/document/9458932)

看了论文标题就差不多知道，这篇文章要做的，就是设计一种算法来衡量两条轨迹之间的相似度，并且这些轨迹数据是有定位误差和零星采样问题的。



------
