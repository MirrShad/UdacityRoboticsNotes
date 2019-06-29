# 例子
简单分析了Python Robotics中提供的例程，用来分析EKF，[链接](https://github.com/AtsushiSakai/PythonRobotics/tree/master/Localization/extended_kalman_filter)
# 分析
我们从main函数开始分析，撇开初始化和画图，主要就三行代码
```
u = calc_input()
xTrue, z, xDR, ud = observation(xTrue, xDR, u)
xEst, PEst = ekf_estimation(xEst, PEst, z, ud)
```
calc_input()是获得真实的速度和角速度，这两个都放在u里面  
observation()提供传感器数据  
observation的定义如下
```
observation(xTrue, xd, u)
return xTrue, z, xd, ud
```
u是真实的速度和角速度  
ud是参杂了噪音的陀螺仪数据，包含了模拟的陀螺仪反馈的速度和角速度  
xd是根据模拟陀螺仪的反馈，推算出来的位置  
z是模拟的GNSS反馈的位置  
xTrue是真实位置  
所以
```
xTrue, z, xDR, ud = observation(xTrue, xDR, u)
```
xTrue是真实位置  
xDR是假设仅仅只有陀螺仪估计出来的位置  
u是真实的速度和角速度  
ud是参杂了噪音的陀螺仪数据  
z是模拟的GNSS反馈的位置  
其实这里不应该叫observation，因为这里连真实的位置都在这里面计算掉了  
EKF是在这一步做掉的
```
xEst, PEst = ekf_estimation(xEst, PEst, z, ud)
```
z是模拟的GNSS反馈的位置  
ud是模拟的陀螺仪反馈  
xEst是先验概率  
PEst是后验概率  
所以我们主要分析这个代码就可以了  
```
def ekf_estimation(xEst, PEst, z, u):

    #  Predict
    xPred = motion_model(xEst, u)   # 根据先验概率和陀螺仪反馈计算估计这时候的位置
    jF = jacobF(xPred, u)           # 计算motion_model的雅可比方程
    PPred = jF@PEst@jF.T + Q        # @在这里是矩阵的乘法的意思，应该是重载过了

    #  Update
    jH = jacobH(xPred)                      # 计算observation_model的雅可比方程
    zPred = observation_model(xPred)        # 主要就是参数转换，这里只有x和y
    y = z - zPred                           # 后面就是标准的EKF了
    S = jH@PPred@jH.T + R
    K = PPred@jH.T@np.linalg.inv(S)
    xEst = xPred + K@y
    PEst = (np.eye(len(xEst)) - K@jH)@PPred #

    return xEst, PEst
```
