---
title: '让 OpenWRT 下的空闲硬盘自动停转'
date: 2012-04-14

tags: [Linux]
---

前一段时间在一个跑着 OpenWRT 的路由器上加了个移动硬盘，但是过了段时间发现硬盘一直不停的转，不管有没有存取操作。这不行啊，既费电也减少硬盘寿命。后来发现 OpenWRT 里可以安装一个可以对 SCSI 驱动器进行操作的软件，sdparm。
关于 sdparm 的详细解释可以看[官方文档](http://sg.danny.cz/sg/sdparm.html)。这里只用到一条简单的命令，

```bash
sdparm -C stop /dev/$DISKNAME
```

作用就是使指定的硬盘停转。接下来的问题是怎样判断硬盘是否空闲。可以根据/proc/diskstats 里的状态信息来判断。这个文件里的一条典型记录长这个样：

```bash
8       8 sda8 11831 25104 1228898 1598268 4290 4388 249536 3469116 0 83176 5067356
```

前三列分别是主设备号，次设备号和设备名称。而倒数第三列是当前对硬盘的 IO 操作数，正是需要的信息。可以看出来 sda8 分区现在就是空闲的。

有了这些准备就可一写出一个简单的脚本了：

```bash
#!/bin/bash
DISKNAME='sda1'
a=0
for i in `seq 0 10`
do
    b=`cat /proc/diskstats | grep $DISKNAME | awk '{print $(NF-2)}'`
    a=`expr $a + $b`
    sleep 1
done
echo $a
if [ $a == 0 ]
then
    echo "No Activity"
    sdparm -C stop /dev/$DISKNAME
else
    echo "Disk Active"
fi
exit 0
```

for 循环是判断在 10s 内对硬盘是否有存取操作，如果没有就认为空闲。其中的`awk '{print $(NF-2)}'`是打印出倒数第三列，NF=number of fields。
接下来还需要建立一个周期任务来执行这个脚本。OpenWRT 下 root 用户的 crontab 是/etc/crontabs/root 这个文件。加入如下这行：

```bash
*/10 * * * * sh /usr/bin/spindown >> /var/log/spindown.log
```

每十分钟判断一下硬盘是否空闲。

参考:

1. [The sdparm utility](http://sg.danny.cz/sg/sdparm.html)
2. [I/O statistics fields](http://www.kernel.org/doc/Documentation/iostats.txt)
