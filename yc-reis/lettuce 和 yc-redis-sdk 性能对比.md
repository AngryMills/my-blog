# lettuce 和 yc-redis-sdk 性能对比

## 耗时对比

1.  1000 次 set 请求 耗时对比

   yc-redis-sdk:平均耗时:avg =6.217 ms

   lettuce:平均耗时: avg =7.306  ms

2.  1000次 get 请求 耗时对比

   yc-redis-sdk:平均耗时: avg =6.878 ms

   lettuce:平均耗时: avg =9.061  ms

3.  1000次 zsetadd 请求 耗时对比

   yc-redis-sdk:平均耗时: avg =6.93 ms

   lettuce:平均耗时: avg =8.171 ms

4.  1000次 zsetget 请求 耗时对比

   yc-redis-sdk:平均耗时: avg =32.877 ms

   lettuce:平均耗时:  avg =22.336 ms

## 并发情况下 性能对比

10 条线程/20次循环

QPS大概在50左右

lettuce:

cpu: 峰值15%（初始 0%）,内存223M（初始80M）

yc-redis-sdk:

cpu: 峰值12.4%,内存224M（初始80M）

## 总结

1.  次数较少时(<1000次),redis-sdk时间提升比较明显,次数较多时,差距不大。
2.  两种方式性能差别不大,redis-sdk略优。
3.  zset的get命令(执行1000次)，lettuce 耗时明显优于 redis-sdk。
4. 统一开发sdk版本,便于升级、维护和管理。