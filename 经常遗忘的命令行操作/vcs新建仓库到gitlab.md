

cd src/<pkg>

git remote add origin git@gitlab.xmutros2snake.com:xmut/ros_packages/<name>.git

git fetch origin # 先看远端有什么，别急着 branch -M

git checkout -b feature/arti origin/main # 或 origin/master，接住远端的初始 commit

git add -A && git commit -m "..."

git push -u origin feature/arti # 新分支，无需 -f




## 1. `vcs` 是什么

在 ROS 2 开发中，`vcs` 通常指 **vcstool** 提供的命令行工具，而不是单独的 Git 命令。

它的作用是：**用一个 YAML 清单文件统一记录、下载、更新、检查多个代码仓库。**

一个 ROS 2 工作空间往往包含多个仓库，例如主项目、机器人描述、驱动、消息包、第三方依赖。逐个 `git clone` 容易遗漏分支、版本和目录；`vcs` 用一个 `.repos` 文件描述这些仓库，别人或 CI 可据此复现相同的源码树。

`vcstool` 支持 Git、Mercurial、Subversion 和 Bazaar；ROS 2 工程中几乎总是使用 Git。它递归扫描指定目录下的仓库，并调用对应原生 VCS 客户端执行操作。


## 2. 它与 Git、rosdep、colcon 的分工

|工具|解决的问题|典型命令|
|---|---|---|
|Git|管理**一个**仓库的提交、分支和合并|`git status`、`git commit`、`git pull`|
|`vcs` / vcstool|批量管理工作空间中的**多个**仓库|`vcs import`、`vcs pull`|
|rosdep|安装系统级/ROS 依赖|`rosdep install ...`|
|colcon|构建 ROS 2 包|`colcon build`|

一套典型流程是：

```
vcs import  →  rosdep install  →  colcon build  →  source install/setup.bash
下载源码        安装依赖          编译工作空间       载入环境
```

## 4. `.repos` 文件是什么

`.repos` 是 YAML 格式的多仓库清单。它记录每个仓库的：

- 本地相对路径；
- 版本控制类型；
- 远程地址；
- 目标分支、标签或提交哈希。

示例：`my_robot.repos`

```
repositories:
  src/my_robot_bringup:
    type: git
    url: https://github.com/example/my_robot_bringup.git
    version: main

  src/my_robot_description:
    type: git
    url: git@github.com:example/my_robot_description.git
    version: jazzy

  src/third_party/sensor_driver:
    type: git
    url: https://github.com/example/sensor_driver.git
    version: 8f4a1c2d7e9b
```

目录键的含义：

```
工作空间根目录
└── src/
    ├── my_robot_bringup/         ← 第一个仓库
    ├── my_robot_description/     ← 第二个仓库
    └── third_party/sensor_driver/← 第三个仓库
```

`version` 可以是分支名（如 `main`、`jazzy`）、标签或提交哈希。使用提交哈希最可复现；使用分支名则以后再次导入时可能得到更新后的代码。


## 5. 最常用：导入一个 ROS 2 工作空间

先创建工作空间和 `src`：

```
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
```

从清单导入所有仓库：

```
vcs import src < my_robot.repos
```

含义：

- `vcs import`：读取仓库清单并克隆；
- `src`：将清单中相对路径放在该目录下；
- `< my_robot.repos`：把 YAML 文件通过标准输入交给 `vcs`。
导入后继续执行：

```
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

`vcs import` 通过标准输入读取 YAML 并克隆所有仓库，也兼容较早的 rosinstall 清单格式。

### 已存在仓库时

如果 `src` 已经有部分仓库，常用：

```
vcs import src --skip-existing < my_robot.repos
```

这会跳过已存在的仓库，避免重复克隆；但不会把它们自动切换到 `.repos` 中指定的版本，版本仍需自行检查。



## 工作空间日常维护命令

以下命令通常在工作空间根目录执行：

```
cd ~/ros2_ws
```

|目的|命令|说明|
|---|---|---|
|查看所有仓库状态|`vcs status src`|批量执行 Git status|
|查看所有未提交改动|`vcs diff src`|批量显示差异|
|拉取所有仓库更新|`vcs pull src`|对各仓库执行 pull|
|查看分支/远端信息|`vcs custom src --args branch -vv`|将 Git 参数批量传入|
|查看当前提交|`vcs custom src --args rev-parse HEAD`|输出每个仓库的 commit|
|执行任意 Git 子命令|`vcs custom src --args <git参数>`|批量操作入口|