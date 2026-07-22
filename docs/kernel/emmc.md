# emmc

## 查看emmc信息

mmcinfo

## 压测emmc
开源工具：stress-ng
命令：stress-ng --hdd 10 --hdd-bytes 128M --hdd-opts fsync,sync --temp-path /userdata/stress --timeout 200h --metrics-brief


压测时间8天