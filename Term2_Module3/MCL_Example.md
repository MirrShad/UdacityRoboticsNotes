# 例子
简单分析了Python Robotics中提供的例程，用来分析EKF，[链接](https://github.com/AtsushiSakai/PythonRobotics/tree/master/Localization/particle_filter)
# 结果
其中红色的线是hxEst，就是估计出来的路线  
蓝色的线是hxTrue，是真实的位置  
黑色的线是hxDR，是仅仅根据陀螺仪数据推测出来的路线  
关于画图的解读可以参考底部链接  
# 分析
我们从main函数开始分析，撇开初始化和画图，主要就三行代码
```
u = calc_input()
xTrue, z, xDR, ud = observation(xTrue, xDR, u, RFID)
xEst, PEst, px, pw = pf_localization(px, pw, xEst, PEst, z, ud)
```
calc_input()是提供真实输入的，就是真实的速度和角速度  
observation()提供传感器数据  
pf_localization()判断目前位置  
## calc_input
这个函数就是返回目前真实速度1m/s 真实角速度1rad/s
## observation
首先我们理解函数输入输出  
```
def observation(xTrue, xd, u, RFID):
return xTrue, z, xd, ud
```
xTrue是期望位置
RFID是RFID传感器的位置
u是真实输入
xd是仅仅根据陀螺仪反馈推测出来的结果
z是添加了噪声的RFID信息
ud是添加了噪音的陀螺仪数据
## pf_localization
```
def pf_localization(px, pw, xEst, PEst, z, u): # z是RFID信息，u是陀螺仪反馈的角速度和速度
    """
    Localization with Particle filter
    """

    for ip in range(NP):                        # NP number of particle
        x = np.array([px[:, ip]]).T
        w = pw[0, ip]
        #  Predict with random input sampling
        ud1 = u[0, 0] + np.random.randn() * Rsim[0, 0]  # 这里将陀螺仪的数据又添加了一层噪音，尝试去掉过后，跟随效果明显变差
        ud2 = u[1, 0] + np.random.randn() * Rsim[1, 1]
        ud = np.array([[ud1, ud2]]).T
        x = motion_model(x, ud)                         # 让每个粒子做跟目标物体相同的动作

        #  Calc Importance Weight
        for i in range(len(z[:, 0])):                   # 把所有的RFID传感器都循环一遍
            dx = x[0, 0] - z[i, 1]                      # 到RFID的x轴的距离
            dy = x[1, 0] - z[i, 2]                      # 到RFID的y轴的距离
            prez = math.sqrt(dx**2 + dy**2)             # 到RFID的距离
            dz = prez - z[i, 0]                         # 目标物体到这个RFID的距离减去粒子到RFID的距离
            w = w * gauss_likelihood(dz, math.sqrt(Q[0, 0]))    # 计算高斯分布下的可能性作为权重

        px[:, ip] = x[:, 0]                             # 更新粒子的最新位置
        pw[0, ip] = w                                   # 更新粒子的权重

    pw = pw / pw.sum()  # normalize

    xEst = px.dot(pw.T)                                 # 把所有点的位置乘以权重相加就是估计位置 
    PEst = calc_covariance(xEst, px, pw)                # 协方差计算公式

    px, pw = resampling(px, pw)                         

    return xEst, PEst, px, pw
```
# 链接
关于python matplot画图的简介，比如.r就是画红色的点，^r就是画红色的三角形
https://blog.csdn.net/qiurisiyu2016/article/details/80187177  

