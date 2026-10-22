
这个工程用容器的核心目的，是把“能运行无人车软件的一整套环境”固定下来：Ubuntu、ROS 2 Lyrical、编译器、Gazebo、依赖库、各 ROS 包的已编译结果。这样每个人、CI 和车载工控机尽量运行相同的软件组合，而不用在宿主机上手动安装并维护大量依赖。

容器不是虚拟机。它与宿主机共用 Linux 内核，但有自己看到的文件系统、进程、环境变量和（通常）网络空间。可以把它理解为“受隔离、可复制的程序运行环境”。

这几个概念要分开：

```
Dockerfile ──构建──> 镜像 image ──创建/启动──> 容器 container
                   （只读模板）          （模板的一次运行实例）
```

- `Dockerfile`：做环境的配方。
- 镜像（image）：按配方打好的、可重复使用的软件包；本身不运行。
- 容器（container）：从镜像创建出来的实际进程。停止后仍可存在，删除容器后其未挂载的数据会消失。
- `compose.yaml`：描述一个或多个容器如何启动、网络如何接入、哪些文件共享、环境变量是什么。
器/镜像作用`amoyfs_simulation`Gazebo 仿真、RViz、赛道、传感器仿真`amoyfs_perception`LiDAR 驱动/仿真点云、地面去除、锥桶检测`amoyfs_planning`根据锥桶、任务状态生成规划结果`amoyfs_rt_control`接近实车的实时控制、车辆描述等`amoyfs_monitoring`Foxglove Bridge，把 ROS 数据提供给可视化客户端

仿真运行时，实际上不止一个容器：

```
simulation                 Gazebo、世界、传感器、RViz
racecar_description        发布赛车 URDF / TF
spawn_racecar              把赛车放入 Gazebo 世界
perception                 订阅仿真点云，输出锥桶检测
```

它们通过 ROS 2/DDS 交换话题。例如仿真容器发布 `/simulation/lidar/points`，感知容器订阅它并发布 `/cone_detection_node/detections`。

### “下载镜像”到底是什么？

`docker pull` 是从镜像仓库下载镜像到本机，类似下载一个版本化的软件运行环境。

项目里的：

```
cd workspace/amoyfs_simulation
./scripts/pull_image.sh
```

只会拉取仿真使用的基础镜像；它不会构建本项目代码，也不会启动任何容器。

下载后可以查看：

```
docker images
```

当前机器上已经有这些本地项目镜像：

```
amoyfs_simulation:dev   约 4.15 GB
amoyfs_perception:dev   约 6.39 GB
amoyfs_rt_control:dev   约 1.89 GB
amoyfs_monitoring:dev   约 1.58 GB
```

镜像很大主要是 ROS、Gazebo、图形/驱动库、编译工具和 ROS 工作空间，不是单纯项目源码很大。
### “下载镜像”到底是什么？

`docker pull` 是从镜像仓库下载镜像到本机，类似下载一个版本化的软件运行环境。

项目里的：

```
cd workspace/amoyfs_simulation
./scripts/pull_image.sh
```

只会拉取仿真使用的基础镜像；它不会构建本项目代码，也不会启动任何容器。

下载后可以查看：

```
docker images
```

当前机器上已经有这些本地项目镜像：

```
amoyfs_simulation:dev   约 4.15 GB
amoyfs_perception:dev   约 6.39 GB
amoyfs_rt_control:dev   约 1.89 GB
amoyfs_monitoring:dev   约 1.58 GB
```

镜像很大主要是 ROS、Gazebo、图形/驱动库、编译工具和 ROS 工作空间，不是单纯项目源码很大。

### “构建镜像”与“拉起容器”不是一回事

```
docker compose build
```

读取 `Dockerfile`，把当前源代码编译进镜像。例如生成 `amoyfs_simulation:dev`。

```
docker compose up
```

根据 `compose.yaml` 创建并启动容器。

```
docker compose up --build
```

先构建镜像，再创建/启动容器。开发时最常用。

仿真的推荐入口是：

```
cd workspace/amoyfs_simulation
./scripts/start_simulation.sh
```

它会根据显卡情况选择 NVIDIA、集显或软件渲染配置，然后组合多个 Compose 文件，启动仿真、车辆描述、生成车辆，并默认启动感知容器。

常用状态命令：

```
docker ps                 # 仅查看正在运行的容器
docker ps -a              # 连已停止的也查看
docker compose logs -f    # 持续看当前项目日志
docker compose stop       # 停止，但保留容器
docker compose down       # 停止并删除该 Compose 项目的容器
docker compose exec simulation bash  # 进入一个正在运行的仿真容器
```

`docker compose start` 只是重新启动已经存在且停止的容器；它不会重新构建镜像，也不会应用新的 Compose 配置。

### 本工程的几个关键 Compose 配置

`network_mode: host`

- 容器与宿主共享网络。
- ROS 2/DDS 能方便发现彼此，也便于对接 LiDAR、CAN、仿真和 Foxglove。
- 因此它不需要常见的 `ports: 8765:8765` 映射；例如 monitoring 的 Foxglove Bridge 直接使用宿主机端口。