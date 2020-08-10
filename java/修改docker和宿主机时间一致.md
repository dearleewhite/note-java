在查看容器的日志的，发现时间有和宿主主机时间相差有8个小时，而且宿主主机使用的是CST时间，容器容器使用的是UTC时间

![img](https://img2018.cnblogs.com/blog/1216966/201901/1216966-20190131083814668-1176123645.png)

主机时间

![img](https://img2018.cnblogs.com/blog/1216966/201901/1216966-20190131083855413-2045070492.png)

DOCKER容器的时间

世界协调时间(Universal Time Coordinated,UTC) 

CST China Standard Time UTC+8:00 中国沿海时间(北京时间)

 在容器中修改下/etc/localtime文件的名称，避免冲突。

root@ddbfb445e9ca:# cd /etc/ 

root@ddbfb445e9ca:/etc# mv localtime localtime_bak

 

root@ddbfb445e9ca:/etc# cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

 然后在查看时间

![img](https://img2018.cnblogs.com/blog/1216966/201901/1216966-20190131084856337-1360965974.png)

docker中的时间

 

![img](https://img2018.cnblogs.com/blog/1216966/201901/1216966-20190131084907645-1749637661.png)

宿主主机的时间。
 
宿主主机和容器时间一致。