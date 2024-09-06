---
title: 'pytorch坑：torch.index_select()导致结果无法复现'
date: 2024-01-29
excerpt: ""
permalink: /posts/2024/01/indexselect/
tags:
  - pytorch
  - torch.index_select()
  - random seed
---

做深度学习方面的研究，很重要的一点就是希望自己的网络结果可以复现，即在输入和各项参数相同的情况下，重复运行代码，得到的结果也是相同的。但是由于代码中各种各样的不确定性因素，常常会导致结果无法复现的情况。

绝大部分的不确定性因素（不考虑软硬件环境、库版本的问题）出现在random、numpy等一些库的随机性上面，这些情况只要确定好随机种子，并在代码中进行如下设置通常就可以完美的解决：

```
random.seed(seed)
os.environ['PYTHONHASHSEED'] = str(seed)
np.random.seed(seed)
torch.manual_seed(seed)
torch.cuda.manual_seed(seed)
torch.cuda.manual_seed_all(seed)
torch.backends.cudnn.deterministic = True
```

还有少数情况是由于函数接口的算法没有确定性的实现导致的，这类问题往往是隐式的，很难发现。

正如本人的代码，设置好了随机种子还是一直没法复现，经过一天半时间的排查和检索资料，终于找到了罪魁祸首，即：torch.index_select()这个函数。我希望通过这个函数来对特征进行一些筛选，但是因为该函数的算法没有确定性的实现，因此在每次运行代码时都会导致随机性，所以每次的结果不同。

此时，你可以torch.index_select()替换成自己的实现，例如下面的函数就可以替代其功能，且可以保证复现：

```
def deterministic_index_select(input_tensor, dim, indices):
    """
    input_tensor: Tensor
    dim: dim 
    indices: 1D tensor
    """
    tensor_transpose = torch.transpose(x, 0, dim)
    return tensor_transpose[indices].transpose(dim, 0)
```

具体我是怎么发现这个函数有问题的：这就要借助到一个很好的工具：

```
torch.use_deterministic_algorithms(True)
```

将此行代码加入到你的代码中后，会自动帮助你查找你的代码中的不确定性算法。如果存在不确定性算法，代码就会终止运行并抛出一个error，如：

```
RuntimeError: index_add_cuda_ does not have a deterministic implementation, but you set 'torch.use_deterministic_algorithms(True)'. You can turn off determinism just for this operation if that's acceptable for your application. You can also file an issue at https://github.com/pytorch/pytorch/issues to help us prioritize adding deterministic support for this operation.
```

这个error还会告诉你具体的那个不确定性算法是什么，通常根据该error信息去官方文档中进行查阅就可以发现有问题的函数，例如抛出了index_add_cuda_这个error一般就是由于使用了torch.index_select()所导致的。

ps1：其实在排查的一开始我就在代码里加了torch.use_deterministic_algorithms(True)，但当时不知道该代码的具体作用，一运行还报错了，所以就把它删掉了...

ps2：本人pytorch版本为1.8.1，其他版本下torch.index_select()是否有问题不清楚。

ps3：相关参考资料：

[PyTorch可复现/重复实验的相关设置](https://zhuanlan.zhihu.com/p/448284000)

[pytorch官方文档](https://pytorch.org/docs/stable/generated/torch.use_deterministic_algorithms.html)




------
