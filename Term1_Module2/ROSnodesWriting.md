# 本章目的
1.写一个simple mover控制simple angle  
2.写一个提供safe move服务的arm mover  
3.写一个订阅camera数据的look away节点
# 定义publisher
```
pub1 = rospy.Publisher("/topic_name", message_type, queue_size=size)
```
publisher可以同步发布(Synchronous publishing )或者异步发布(Asynchronous publishing)  
区别在于如果选择了同步发布，前面如果有一个publisher在发布，会卡在这里直到前一个全部发布完了，然后自己发布然后再进行下面的程序  
异步发布如果前面有一个publisher在发布，会先将这个message放到一个bufffer里面，如果buffer满了会把最旧的message给推掉，然后自己就去做别的事了，然后message会等到能发送的时候再发送。  
可以看到同步异步的区别在于有没有一个buffer来存暂时发不出去的message，所以区别在于是否定义了queue_size，如果queue_size为None或者不定义，就是同步发布，如果定义了就是buffer的大小
```
pub1.publish(message)   //发布message
```
# 定义subscribers
```
sub1 = rospy.Subscriber("/topic_name", message_type, callback_function)
```
在实际运用中会遇到程序运行先后的问题，所以一般在挂载subscriber之前会等待我们需要订阅的messages初始化完成
```
rospy.wait_for_message('/simple_arm/joint_states', JointState)
self.sub1 = rospy.Subscriber('/simple_arm/joint_states', 
                                 JointState, self.joint_states_callback)
```
# 定义service
```
//创建service
service = rospy.Service('service_name', serviceClassName, handler)
//调用service
service_proxy = rospy.ServiceProxy('service_name', serviceClassName)
try:
    rospy.wait_for_service('service_name')//虽然调用service使用上下两句就可以了，但是一般会调用这个方法来保证这个service已经被注册
    response = service_proxy(msg)
except rospy.ServiceException, e:
    rospy.logwarn("Service call failed: %s", e)
```
serviceClassName是service定义的文件的名字
# 流程
1. 定义好service会用到的srv文件，并且挂载到编译系统里面
首先在package中创建srv文件夹，然后在srv里面创建service名字的srv文件，定义好输入和输出
```
float64 joint_1
float64 joint_2
---
duration time_elapsed
```
修改package目录下的CMakeLists.txt
```
find_package(catkin REQUIRED COMPONENTS
        std_msgs
        message_generation
)
add_service_files(
   FILES
   GoToPosition.srv
)
generate_messages(
   DEPENDENCIES
   std_msgs  # Or other packages containing msgs
)
```
修改package.xml

2. 编写nodes的行为
```
cd ~/catkin_ws/src/simple_arm/
mkdir scripts
cd scripts
/* 编写nodes的行为
echo '#!/bin/bash' >> hello
echo 'echo Hello World' >> hello
*/
chmod u+x hello
cd ~/catkin_ws
catkin_make
source devel/setup.bash
rosrun simple_arm hello
```
## 例子
```python
#!/usr/bin/env python

import math
import rospy
from std_msgs.msg import Float64
from sensor_msgs.msg import JointState
from simple_arm.srv import *

def at_goal(pos_j1, goal_j1, pos_j2, goal_j2):
    tolerance = .05
    result = abs(pos_j1 - goal_j1) <= abs(tolerance)
    result = result and abs(pos_j2 - goal_j2) <= abs(tolerance)
    return result

def clamp_at_boundaries(requested_j1, requested_j2):
    clamped_j1 = requested_j1
    clamped_j2 = requested_j2

    min_j1 = rospy.get_param('~min_joint_1_angle', 0)           # 这些参数能在launch文件中找到，像是roboshop中的mutable
    max_j1 = rospy.get_param('~max_joint_1_angle', 2*math.pi)
    min_j2 = rospy.get_param('~min_joint_2_angle', 0)
    max_j2 = rospy.get_param('~max_joint_2_angle', 2*math.pi)

    if not min_j1 <= requested_j1 <= max_j1:
        clamped_j1 = min(max(requested_j1, min_j1), max_j1)
        rospy.logwarn('j1 is out of bounds, valid range (%s,%s), clamping to: %s',
                      min_j1, max_j1, clamped_j1)

    if not min_j2 <= requested_j2 <= max_j2:
        clamped_j2 = min(max(requested_j2, min_j2), max_j2)
        rospy.logwarn('j2 is out of bounds, valid range (%s,%s), clamping to: %s',
                      min_j2, max_j2, clamped_j2)

    return clamped_j1, clamped_j2

def move_arm(pos_j1, pos_j2):
    time_elapsed = rospy.Time.now()
    j1_publisher.publish(pos_j1)
    j2_publisher.publish(pos_j2)

    while True:
        joint_state = rospy.wait_for_message('/simple_arm/joint_states', JointState)
        if at_goal(joint_state.position[0], pos_j1, joint_state.position[1], pos_j2):
            time_elapsed = joint_state.header.stamp - time_elapsed
            break

    return time_elapsed

def handle_safe_move_request(req):
    rospy.loginfo('GoToPositionRequest Received - j1:%s, j2:%s',
                   req.joint_1, req.joint_2)
    clamp_j1, clamp_j2 = clamp_at_boundaries(req.joint_1, req.joint_2)
    time_elapsed = move_arm(clamp_j1, clamp_j2)

    return GoToPositionResponse(time_elapsed)   #返回值为GoToPosition(srv文件名)Response(time_elapsed(返回值))

def mover_service():
    rospy.init_node('arm_mover')
    service = rospy.Service('~safe_move', GoToPosition, handle_safe_move_request)   #GoToPosition中定义了req的输入和输出是上面流程章节的示例
    rospy.spin()    #spin一般用于设置好回调并且不需要后台程序继续维护，比如定时更新的情况下

if __name__ == '__main__':
    j1_publisher = rospy.Publisher('/simple_arm/joint_1_position_controller/command',
                                   Float64, queue_size=10)
    j2_publisher = rospy.Publisher('/simple_arm/joint_2_position_controller/command',
                                   Float64, queue_size=10)

    try:
        mover_service()
    except rospy.ROSInterruptException:
        pass
```
这个例子在主函数中演示了如何发布topic并且注册service，这个service是service_name是safe_move，实际调用的是handle_safe_move_request，这个回调函数会接收一个变量，在clamp_at_boundaries判断是否在合理范围内，然后在move_arm中发布命令topic持续等待JointState发布的topic直到Joint执行到命令的角度
3. 把nodes放在launch中让nodes在ROS启动的时候启动
在launch文件夹里面~/catkin_ws/src/simple_arm/launch写好
```
  <!-- The arm mover node -->
  <node name="arm_mover" type="arm_mover" pkg="simple_arm">
    <rosparam>
      min_joint_1_angle: 0
    </rosparam>
  </node>
```


# TIPS快速调用service的方法
```
rosservice call /arm_mover/safe_move "joint_1: 1.57
joint_2: 1.57"
```
