
# List源码

## 1.1 ArrayList

## 1.2 LinkedList

## 1.3 CopyOnWriteArrayList

Redis4之后支持RDB-AOF混合持久化的方式

bgsave操作使用CopyOnWrite机制进行写时复制，是由一个子进程将内存中的最新数据遍历写入临时文件，此时父进程仍旧处理客户端的操作，当子进程操作完毕后再将该临时文件重命名为dump.rdb替换掉原来的dump.rdb文件，因此无论bgsave是否成功，dump.rdb都不会受到影响
