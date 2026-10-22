第一终端先仅启动相机驱动，并保持它运行：
cd /home/amoy/F407_active/joho_snake_robot
source /opt/ros/jazzy/setup.bash
source install/orbbec_v1/setup.bash

ros2 run orbbec_camera orbbec_camera_node --ros-args \
  -r __ns:=/camera \
  -r __node:=camera \
  -p enable_color:=true \
  -p enable_depth:=true \
  -p depth_registration:=true \
  -p color_width:=640 \
  -p color_height:=480 \
  -p color_fps:=30 \
  -p depth_width:=640 \
  -p depth_height:=400 \
  -p depth_fps:=30 \
  -p depth_format:=Y11




先按当前真实相机坐标系启动 RTAB-Map：
source /opt/ros/jazzy/setup.bash
source install/orbbec_v1/setup.bash

ros2 launch rtabmap_launch rtabmap.launch.py \
  frame_id:=orbbec_camera_link \
  rgb_topic:=/camera/color/image_raw \
  depth_topic:=/camera/depth/image_raw \
  camera_info_topic:=/camera/color/camera_info \
  rgbd_sync:=true \
  approx_rgbd_sync:=true \
  qos:=2 \
  visual_odometry:=true \
  odom_topic:=/slam/odom \
  rtabmap_viz:=true \
  rviz:=false \
  rtabmap_args:="--delete_db_on_start"