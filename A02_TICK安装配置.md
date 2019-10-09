## 安装配置

*参考：http://gitlab.jiagouyun.com/DataProduct/CloudCareMonitor/tree/master/scripts/bash_scripts*



## 配置influxdb

```shell
influx -execute "create database telegraf"  
# 创建用户
influx -database 'telegraf' -execute "create user "xxx" with password 'xxx' with all privileges" 
influx -database 'telegraf' -execute "ALTER RETENTION POLICY autogen ON telegraf DURATION 30d DEFAULT"
sed -i '/auth-enabled = false/a\\  auth-enabled = true' /etc/influxdb/influxdb.conf
systemctl restart influxdb



```

## 配置telegraf

```shell
[root@Oracle01 install]# rpm -ql telegraf
/etc/logrotate.d/telegraf
/etc/telegraf
/etc/telegraf/telegraf.conf
/etc/telegraf/telegraf.d
/usr/bin/telegraf
/usr/lib/telegraf/scripts/init.sh
/usr/lib/telegraf/scripts/telegraf.service
/var/log/telegraf

[root@Oracle01 telegraf]# cat telegraf.conf 
[global_tags] #所有的measurements中都会有
  instanceId = oracle_ocp01
  product = idc 
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  hostname = ""
  omit_hostname = false
  logfile = "/var/log/telegraf/telegraf.log"
[[outputs.influxdb]]                                    # 采集的数据填写到influxdb
  urls = ["http://127.0.0.1:8086"]
  database = "telegraf"
  skip_database_creation = true
  username = "xxx"
  password = "xxx"
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]
[[inputs.diskio]]
[[inputs.kernel]]
[[inputs.mem]]
[[inputs.processes]]
[[inputs.swap]]
[[inputs.system]]

#配置mysql 插件 
vim /etc/telegraf/telegraf.d/mysql.conf
[[inputs.mysql]]
  interval = "300s"
  servers = ["xxx:xxx@tcp(xxx:3306)/?tls=false"]
  metric_version = 2
  perf_events_statements_digest_text_limit  = 120
  perf_events_statements_limit              = 250
  perf_events_statements_time_limit         = 86400
  table_schema_databases                    = []
  gather_table_schema                       = false
  gather_process_list                       = true
  gather_user_statistics                    = true
  gather_info_schema_auto_inc               = false
  gather_innodb_metrics                     = true
  gather_slave_status                       = true
  gather_binary_logs                        = false
  gather_table_io_waits                     = false
  gather_table_lock_waits                   = false
  gather_index_io_waits                     = false
  gather_event_waits                        = false
  gather_file_events_stats                  = false
  gather_perf_events_statements             = false
  interval_slow                   = "30m"
  [inputs.mysql.tags] #告警吐出该台mysql的信息
    host='rds描述信息'
    instanceId='rds1008id'
    product='rds'
    
# 启动telegraf
systemctl start telegraf
    
```

### 调试telegraf

```shell
[root@Oracle01 telegraf]# ps -ef | grep telegraf
telegraf  14222      1  0 11:40 ?        00:00:28 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d
root      42854  37007  0 15:26 pts/2    00:00:00 grep --color=auto telegraf

[root@Oracle01 telegraf]# /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d --test
```

###  查看采集数据

```shell
[root@Oracle01 telegraf.d]# influx -username xxx -password xxx -precision 'rfc3339'
Connected to http://localhost:8086 version 1.7.8
InfluxDB shell version: 1.7.8
> show databases
name: databases
name
----
telegraf
_internal
chronograf
> use telegraf
Using database telegraf
> show measurements
name: measurements
name
----
cpu
disk
diskio
kernel
mem
mysql
mysql_innodb
mysql_process_list
mysql_users
mysql_variables
processes
swap
system

> select * from cpu order by time desc limit 1;
name: cpu
time                 cpu  host     usage_guest usage_guest_nice usage_idle        usage_iowait usage_irq usage_nice usage_softirq usage_steal usage_system        usage_user
----                 ---  ----     ----------- ---------------- ----------        ------------ --------- ---------- ------------- ----------- ------------        ----------
2019-10-09T07:46:23Z cpu0 Oracle01 0           0                98.58585858586467 0            0         0          0             0           0.40404040404026426 1.0101010101010912
> 

> show tag keys from cpu
name: cpu
tagKey
------
cpu
host

> show series from cpu  #数据会放到时间线当中
key
---
cpu,cpu=cpu-total,host=Oracle01
cpu,cpu=cpu-total,host=Oracle01,instanceId=oracle_ocp01,product=idc
cpu,cpu=cpu0,host=Oracle01
cpu,cpu=cpu0,host=Oracle01,instanceId=oracle_ocp01,product=idc

> show field keys from cpu
name: cpu
fieldKey         fieldType
--------         ---------
usage_guest      float
usage_guest_nice float
usage_idle       float
usage_iowait     float
usage_irq        float
usage_nice       float
usage_softirq    float
usage_steal      float
usage_system     float
usage_user       float

> show retention policies on "telegraf"  # 查看保存策略
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 720h0m0s 168h0m0s           1        true

duration是数据保留时间
shardGroupDuration是每个shard上的数据的时间跨度
replicaN是数据的副本数量
default是指这个retention policy是否是指定数据库的默认数据保留策略

## 备份
[root@Oracle01 telegraf.d]# influxd backup  -portable -database telegraf /tmp/
2019/10/09 16:13:42 backing up metastore to /tmp/meta.00
2019/10/09 16:13:42 backing up db=telegraf
2019/10/09 16:13:42 backing up db=telegraf rp=autogen shard=2 to /tmp/telegraf.autogen.00002.00 since 0001-01-01T00:00:00Z
2019/10/09 16:13:42 backup complete:
2019/10/09 16:13:42 	/tmp/20191009T081342Z.meta
2019/10/09 16:13:42 	/tmp/20191009T081342Z.s2.tar.gz
2019/10/09 16:13:42 	/tmp/20191009T081342Z.manifest

备份的目录必须存在

## 还原
[root@Oracle01 2]# influxd restore -portable -db telegraf -newdb telegraf_bak /tmp/
2019/10/09 17:15:49 Restoring shard 2 live from backup 20191009T081342Z.s2.tar.gz

USE telegraf_bak
SELECT * INTO telegraf..:MEASUREMENT FROM /.*/ GROUP BY *
DROP DATABASE telegraf_bak

两份备份会取离当前时间较近的
不能在同一个目录下指定要还原的备份
无法还原到已经存在的数据库中
可以通过还原到临时数据库，再从临时数据库select * into数据后，删除临时数据库

> drop database telegraf_bac  # 删库操作


```

## 配置chronograf

```shell
启动
[root@Oracle01 ~]# systemctl start chronograf
```

![](D:\智能团队相关资料\08_学习内容\09_TICK\pic\1570524248768.png)

![](D:\智能团队相关资料\08_学习内容\09_TICK\pic\1570524296750.png)

![](D:\智能团队相关资料\08_学习内容\09_TICK\pic\1570524365847.png)

![](D:\智能团队相关资料\08_学习内容\09_TICK\pic\1570524416354.png)

![](D:\智能团队相关资料\08_学习内容\09_TICK\pic\1570613932937.png)

![](D:\智能团队相关资料\08_学习内容\09_TICK\pic\1570613998554.png)

## 配置kapacitor

- kapacitor常用命令

```shell
kapacitor define cpu_idle_task -tick cpu_idle_task.tick # 定义任务名
kapacitor list tasks # 查看任务列表
kapacitor enable 任务名 # 开启任务
kapacitor show 任务名 # 查看任务信息
kepacitor watch 任务名 # 查看详情日志

#相关目录
[root@Oracle01 task]# rpm -ql kapacitor
/etc/kapacitor
/etc/kapacitor/kapacitor.conf
/etc/logrotate.d/kapacitor
/usr/bin/kapacitor
/usr/bin/kapacitord
/usr/bin/tickfmt
/usr/lib/kapacitor
/usr/lib/kapacitor/scripts
/usr/lib/kapacitor/scripts/init.sh
/usr/lib/kapacitor/scripts/kapacitor.service
/usr/share/bash-completion/completions/kapacitor
/var/lib/kapacitor
/var/log/kapacitor

```

- TICKscripts

```shell
cd /var/lib/kapacitor/task
[root@Oracle01 task]# ll
total 4
-rw-r--r--. 1 kapacitor kapacitor 1646 Oct  9 11:57 cpu_idle_task.tick

[root@Oracle01 task]# cat cpu_idle_task.tick  #tick文件更改后，需要重新define
dbrp "telegraf"."autogen"
var db = 'telegraf'

var rp = 'autogen'

var measurement = 'cpu'

var groupBy = ['host', 'instanceId']

var whereFilter = lambda: ("cpu" == 'cpu-total')

var period = 1m

var every = 1m

var name = 'qinxi_alert_test'

var idVar = name + '-{{.Group}}'

var message = '实例为 {{ index .Tags "instanceId" }} 的CPU空闲异常,当前空闲值为 {{ index .Fields "usage_idle_mean" }}'

var idTag = 'alertID'

var levelTag = 'level'

var messageField = 'message'

var durationField = 'duration'

var outputDB = 'chronograf'

var outputRP = 'autogen'

var outputMeasurement = 'alerts'

var triggerType = 'threshold'

var crit = 99

var data = stream
    |from()
        .database(db)
        .retentionPolicy(rp)
        .measurement(measurement)
        .groupBy(groupBy)
        .where(whereFilter)
    |window()
        .period(period)
        .every(every)
        .align()
    |mean('usage_idle')
        .as('usage_idle_mean')

var trigger = data
    |log()
    |alert()
        .crit(lambda: "usage_idle_mean" < crit)
        .message(message)
        .id(idVar)
        .idTag(idTag)
        .levelTag(levelTag)
        .messageField(messageField)
        .durationField(durationField)
        .log('/tmp/qinxialert.log')

trigger
    |eval(lambda: float("usage_idle_mean"))
        .as('usage_idle_mean')
        .keep()
    |influxDBOut()
        .create()
        .database(outputDB)
        .retentionPolicy(outputRP)
        .measurement(outputMeasurement)
        .tag('alertName', name)
        .tag('triggerType', triggerType)

trigger
    |httpOut('output')


[root@Oracle01 task]# kapacitor watch cpu_idle_task
ts=2019-10-09T18:01:10.004+08:00 lvl=info msg=point service=kapacitor task_master=main task=cpu_idle_task node=influxdb_out7 prefix= name=cpu db= rp= group=host=Oracle01,instanceId=oracle_ocp01 dimension_0=host dimension_1=instanceId tag_instanceId=oracle_ocp01 tag_host=Oracle01 field_usage_idle_mean=97.14545517664192 time=2019-10-09T10:01:00Z
ts=2019-10-09T18:01:10.004+08:00 lvl=debug msg="alert triggered" service=kapacitor task_master=main task=cpu_idle_task node=influxdb_out7 level=CRITICAL id=qinxi_alert_test-host=Oracle01,instanceId=oracle_ocp01 event_message="实例为 oracle_ocp01 的CPU空闲异常,当前空闲值为 97.14545517664192" data="&{cpu map[host:Oracle01 instanceId:oracle_ocp01] [time usage_idle_mean] [[2019-10-09 10:01:00 +0000 UTC 97.14545517664192]]}"

```

