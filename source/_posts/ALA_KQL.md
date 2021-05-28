---
title: Azure Monitoring KQL Study
date: 2020-06-04 11:05:42
tags: azure
categories:
- Technical Notes
---
### azure-monitoring-KQL
Azure monitor—Log Analystics  
VM’Extensions— OmsAgentForLinux

### 扩展介绍


### 基础语法
语法  
CustomLog_CL | where RawData matches regex "System warning”
- 区分大小写
- 语法来源
  来自 Azure Data Explorer ，但又不完全相同 [找不同](https://docs.microsoft.com/en-us/azure/azure-monitor/log-query/data-explorer-difference)
  比如,不支持alis：
  ```
  alias database["wiki"] = cluster("https://somecluster.kusto.windows.net:443").database("somedatabase");
  database("wiki").PageViews | count
  ```
  不支持查询参数：
  ```
  declare query_parameters(UserName:string, Password:string);
  print n=UserName, p=hash(Password)
  ```
- 表结构说明
  azure提供了默认表结构字典,[看这里](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/azuremetrics)


### quick start
#### select
```
Event | search "error"
等于
search in (Event) "error"
```
获取某个表内容的任意10行
search in (Heartbeat) "Linux"
| take 10

注：每次查询后的行数限制在30,000行  
获取前10行 用 top

#### 排序用sort
SecurityEvent | sort by TimeGenerated desc


#### 也有order by
```
let t1 = datatable(key:long, value:string,order1:string)
[
5,"status is ok group1","a",
5,"status is ok","a",
5,"status is ok","b",
2,"status is ok","c",
3,"status is ok","d",
1,"status is ok","e",
3,"status is ok","d",
6,"status is ok from group3","f"
];
t1|order by key
```

#### 少不了时间

过去2分钟内：
```
Heartbeat
| where TimeGenerated > now() - 2m
```

某个时间内：
```
where TimeGenerated between(datetime("2020-05-15 22:46:42") .. now())
```

获取在value范围内，最接近value的，roundTo的整数倍值
```
bin(value, roundTo )
```

使用以下查询获取过去半小时内每 5 分钟发生的事件数：
每5m一组，统计每组event的总数 count()
```
Perf
| where TimeGenerated > ago(30m)
| summarize events_count=count() by bin(TimeGenerated, 5m)
```

#### 聚合summarize
Summarize by 类似 distinct
```
let t1 = datatable(key:long, value:string,order1:string)
[
5,"status is ok group1","a",
5,"status is ok","a",
5,"status is ok","b",
2,"status is ok","c",
3,"status is ok","d",
1,"status is ok","e",
3,"status is ok","d",
6,"status is ok from group3","f"
];
t1|summarize by key, value
```
Example：  
```
Heartbeat //表
| where OSType == 'Linux' //列
| summarize arg_max(TimeGenerated, *) by SourceComputerId
| top 500000 by Computer asc //Computer名字升序 最新500000条
| render table //用表格展示 // render columnchart
```

#### 类型强制转换
SecurityEvent | where toint(Level) >= 10  

Example:  
print toint("123")== 123s   结果为false  
print tobool("true") == true 结果为true  


#### join
去重：
![inner+unique example](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/azure_kql/kql-1
)
不去重：
![inner example](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/azure_kql/kql-2)
leftouter 左外部连接，同sql表查询  

leftanti  左反连接 ，仅列出在右表中没有列出的行  
leftsemi 结果中包含左表中具有右表匹配项的记录  
![leftsemi example](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/azure_kql/kql-3)

#### 参考链接：
[DemoLink](https://portal.azure.com/#blade/Microsoft_Azure_Monitoring_Logs/DemoLogsBlade)  
[KQL详细教程说明](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/)  
[和SQL的对比](https://docs.azure.cn/zh-cn/azure-monitor/log-query/sql-cheatsheet)  
[查询日志的方式](https://docs.azure.cn/zh-cn/azure-monitor/log-query/log-query-overview)
