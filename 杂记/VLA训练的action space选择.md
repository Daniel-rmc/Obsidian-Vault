Action space
可以选择为：
1. detal joint space
2. absolute joint space
3. detal eep
4. absolute eep

同时action chunk 中的设置会有关系，action chunk 如何设置绝对和相对
相对的话，**根据action chunk的输入观测，相对应该是在chunk内部做相对；**
而不是帧间做相对。

数据采集记录保留
state 记录 关节角，和末端位置。
action：末端位置（复制的state） 或者 joint 状态。
action 本质上应该由数据采集的 state来构造action，跟robot controller 做对齐。 而数据采集是真机上采集的，只要能在数据集中出现，就是robot controller能做出来的动作。

训练方案：
1. Action：joint space 变化连续，有界，网络易学习。但跨具身泛化性差（但是数据多了，以及指定型号后，也能识别选择domain空间） 还是需要优先能在单机上出现结果，再考虑泛化。
2. Action：EE-pose： 分为平移：dx,dy,dz；姿态变化：四元数表示，欧拉角表示等。


敲定一个训练方案：
1. Action：joint  + observation.state：joint 
2. **Action: eep 欧拉角 增量表示（对应osc controller）， state：joint  （libero benchmark数据集是使用的这种方式）**
3. state：eep （绝对位置 + 欧拉角？这里又有state 应该使用哪些空间表示方法的因素）+ action:  joint / eep。


派生 LeRobot 数据集：

- state 只保留左臂 7 个 joint
- action 只保留左臂的 dx, dy, dz, droll, dpitch, dyaw（均为增量，欧拉角由四元数转，且 action[t]=action[t+1]-action[t]，严格对齐“当前位置预测下一帧目标”）


帮我再写一个转换数据脚本，保留action 和现在的增量位移和欧拉角， 但是state 保留 左臂的