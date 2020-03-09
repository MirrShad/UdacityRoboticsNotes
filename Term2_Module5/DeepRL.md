# Deep RL
## additions
replay pool
target network to represent old Q-function，所以在Deep RL里面实际有两个network
## Deep Q network
讲述一般做法，将一个游戏的画面缩小成比较小的正方形画面，然后用四个同时进行训练，经过几个卷积层过后经过全连接层然后线性输出层
## Experience Replay
当某一个序列经常性地出现导致模型对某个行为的权重过大的时候，可以调用replay的内容让模型重新学习，并且replay让强化学习变成了个监督学习的领域。
我们还可以进一步优化(Prioritized Experience Learning)，将有价值的例子优先训练之类的
## Fixed Q Target
