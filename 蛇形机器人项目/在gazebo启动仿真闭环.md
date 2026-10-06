

请完全关闭旧 Gazebo，然后按以下方式重启。

终端 1：

```
cd ~/F407_active/joho_snake_robot
source /opt/ros/jazzy/setup.bash
source install/setup.bash
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6

ros2 launch ros_gz_sim gz_sim.launch.py gz_args:="-v 3 empty.sdf"
```

终端 2：

```
cd ~/F407_active/joho_snake_robot
source /opt/ros/jazzy/setup.bash
source install/setup.bash

ros2 launch snake_description view_cad_model.launch.py use_gui:=false use_rviz:=false
```

终端 3：

```
cd ~/F407_active/joho_snake_robot
source /opt/ros/jazzy/setup.bash
source install/setup.bash

ros2 run ros_gz_sim create \
  -world empty \
  -topic robot_description \
  -name cad_snake \
  -x 0 -y 0 -z 1.0
```