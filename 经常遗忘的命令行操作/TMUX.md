

# tmux 常见命令总结

## 一、核心概念

|概念|说明|
|---|---|
|**Session（会话）**|最外层，一个项目/任务对应一个会话|
|**Window（窗口）**|会话内的标签页，类似浏览器 Tab|
|**Pane（窗格）**|窗口内的分屏，可水平/垂直切分|

层级关系：`Session > Window > Pane`


## 二、会话管理（终端中执行）

bash

tmux                    # 新建匿名会话
tmux new -s work        # 新建名为 work 的会话
tmux ls                 # 列出所有会话
tmux a                  # 连接最近的会话
tmux a -t work          # 连接名为 work 的会话
tmux kill-session -t work   # 杀死指定会话
tmux kill-server        # 杀死所有会话
tmux rename-session -t old new  # 重命名会话

> 快捷键前缀默认是 `Ctrl+b`，下文用 `<prefix>` 表示。按下前缀后松开，再按后续键。

## 三、会话内操作（快捷键）

| 操作         | 快捷键          |
| ---------- | ------------ |
| 分离会话（后台运行） | `<prefix> d` |
| 列出会话并切换    | `<prefix> s` |
| 重命名会话      | `<prefix> $` |
| 显示所有快捷键    | `<prefix> ?` |
