[[蛇形机器人]]

# 启动路径：
![[Pasted image 20261010143652.png]]



另外，Snap 已删除，但原先运行中的 Snap 进程还留在内存里。执行：

```
pkill -f '/snap/foxglove-studio'
/opt/Foxglove/foxglove-studio --ignore-gpu-blocklist
```



先开一个终端，启动 bridge，且不要关闭该终端：

```
source /opt/ros/jazzy/setup.bash
export ROS_DOMAIN_ID=0

ros2 launch foxglove_bridge foxglove_bridge_launch.xml \
  address:=127.0.0.1 \
  port:=8766
```