
按下面做：

1. 若用 CH343/沁恒转串口的 Type-C 口：重新插拔后执行：

```
ls -l /dev/ttyUSB* /dev/ttyACM*
dmesg | tail -50
```

必须出现类似 `/dev/ttyUSB0`。若依旧只有 `lsusb` 中的 `1a86:55d3`、没有 `ttyUSB`，就是 CH343 驱动未绑定；升级内核或安装匹配的 WCH CH343 Linux 驱动后再试。

2. 让 ESP32-P4 进入下载模式：按住 `BOOT`，短按一次 `RST/EN`，先松开 `RST/EN`，再松开 `BOOT`。然后立即烧录。
    
3. 用实际端口烧录，先用低波特率排除通信质量问题：