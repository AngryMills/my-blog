# kinaba图形化表达式

**Timelion Expression**

接口流量全项目三天对比

```protobuf
.es(index='accesslog-xxxx*').label('当前时间').lines(width=2).color(red),.es(index='accesslog-xxxx*',offset=-1d).label('昨天').lines(width=1).color(green),.es(index='accesslog-xxxx*',offset=-2d).label('前天').lines(width=1).color(blue)
```

平均响应时间三天对比

```protobuf
.es(index='accesslog-car-api*',metric='avg:elapsed_time').color(red).label('当前时间'),
.es(index='accesslog-car-api*',metric='avg:elapsed_time',offset=-1d).color(green).lines(1).label('前一天'),
.es(index='accesslog-car-api*',metric='avg:elapsed_time',offset=-2d).color(blue).lines(1).label('前两天')
```

999Line响应时间三天对比

```protobuf
.es(index='accesslog-car-api*',metric='percentiles:elapsed_time:99.9').color(red).label('当前时间'),
.es(index='accesslog-car-api*',metric='percentiles:elapsed_time:99.9',offset=-1d).color(green).lines(1).label('前一天'),
.es(index='accesslog-car-api*',metric='percentiles:elapsed_time:99.9',offset=-2d).color(blue).lines(1).label('前两天')
```

