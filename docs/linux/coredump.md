# coredump

#ulimit -c unlimited
#echo 1 > /proc/sys/kernel/core_uses_pid
#/usr/sbin/haveged -w 1024 -v 0
ulimit -c 204800  # 200MB = 200*1024KB = 204800KB
# ulimit -c unlimited
echo "/backup/core-%e-%p-%t" > /proc/sys/kernel/core_pattern