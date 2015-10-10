# NaaS-Controller性能数据存取接口说明文档

标签（空格分隔）： Ceilometer

---

[TOC]

## 背景
OpenStack Ceilometer V2 Web API提供了一系列API，但却没有批量存储外部性能数据和多指标查询性能数据的API，因为OpenStack原本只是采集所管理的虚机性能及OpenStack其他组件的性能数据，而不会采集网络设备的性能数据。虽然OpenStack打算支持服务器的性能数据采集，但在NaaS-Controller中使用的Havana版本OpenStack却不支持。
因此，有两种方案：一是开发一个支持在网络设备上部署的agent，二是修改Ceilometer源码，新增两个Web API接口。
本文是方案二。

Controller业务层接口代码位置：
```
EBG_Ctrller_branch_code_SVN\Code\NaaS-Controller\source\naas-controller\performancemgr\pfmmgr-service\src\main\java\com\huawei\controller\pfmmgr\service\ceilometer\PfmDataService.java
```

Controller南向接口代码位置：
```
EBG_Ctrller_branch_code_SVN\Code\NaaS-Controller\source\naas-controller\southbound\adapter\ops-ceilometer-adapter\ops-ceilometer-adapter-service\src\main\java\com\huawei\controller\ops\ceilometer\adapter\impl\CeilometerMeterServiceImpl.java
```

## 保存性能数据到Ceilometer
    naas-controller目前使用MongoDB，本接口也仅支持MongoDB。
    
### 接口描述
Controller业务层接口：
```java
public RpcResult<AddSampleOutput> insertSamples(List<Samples> samples)
```

Controller南向接口：
```java
public Future<RpcResult<AddSampleOutput>> addSample(AddSampleInput input)
```

Ceilometer REST API：
```
POST /v2/meters                       --Post a list of new Samples to Telemetry.
Parameters:	samples (list(Sample))    –-a list of samples within the request body.
Return type: list(Sample)             --a list of samples
```    
### 接口样例


## 多指标性能查询
    naas-controller目前使用MongoDB，本接口也仅支持MongoDB。
    
### 接口描述

Controller业务层接口：
```java
public RpcResult<ComplexQuerySampleOutput> querySamples(ComplexQuery query)
```

Controller南向接口：
```java
public Future<RpcResult<ComplexQuerySampleOutput>> complexQuerySample(
            ComplexQuerySampleInput input)
```

Ceilometer REST API接口：
```
POST /v2/query/samples          --Return a list of samples of particular meters, based on the data recorded so far.
Parameters: q (ComplexQuery)    –-Filter rules for the meters to be returned.
Return type: list(Sample)       --a list of samples
```
### 接口详解
```Python
class ComplexQuery(_Base):
    """Holds a sample query encoded in json."""

    filter = wtypes.text
    "The filter expression encoded in json."

    orderby = wtypes.text
    "List of single-element dicts for specifing the ordering of the results."

    limit = int
    "The maximum number of results to be returned."
    
    ...
```
由`ComplexQuery`类定义可知，`ComplexQuery`包含三个成员，分别是`filter`、`orderby`、`limit`，类型对应如下表格所示：
| 属性 | rest类型 | Java类型 |
|:---:|:---:|:---:|
|filter|wsme.types.text|String|
|orderby|wsme.types.text|String|
|limit|int|Long|

看一下OpenStack对该接口的官方说明：

    As the current implementation accepts only string values as query filter and order by definitions, the above defined expressions have to be converted to string values. By adding a limit criteria to the request, which maximizes the number of returned samples

由此可知，查询过滤条件`filter`当前仅支持`string`，官方例子如下所示：
```
{
"filter" : "{\"and\":[{\"and\": [{\"=\": {\"counter_name\": \"cpu_util\"}}, {\">\": {\"counter_volume\": 0.23}}, {\"<\": {\"counter_volume\": 0.26}}, {\"not\": {\"=\": {\"counter_volume\": 0.2512}}}]}, {\"or\": [{\"and\": [{\">\": {\"timestamp\": \"2013-12-01T18:00:00\"}}, {\"<\": {\"timestamp\": \"2013-12-01T18:15:00\"}}]}, {\"and\": [{\">\": {\"timestamp\": \"2013-12-01T18:30:00\"}}, {\"<\": {\"timestamp\": \"2013-12-01T18:45:00\"}}]}]}]}",
"orderby" : "[{\"counter_volume\": \"ASC\"}, {\"timestamp\": \"DESC\"}]",
"limit" : 4
}
```
调整一下格式：
```
{
"filter" : 
"{
    \"and\":
    [
        {
            \"and\": 
            [
                {
                    \"=\": {\"counter_name\": \"cpu_util\"}
                }, 
                {
                    \">\": {\"counter_volume\": 0.23}
                }, 
                {
                    \"<\": {\"counter_volume\": 0.26}
                }, 
                {
                    \"not\": 
                    {
                        \"=\": {\"counter_volume\": 0.2512}
                    }
                }
            ]
        }, 
        {
            \"or\": 
            [
                {
                    \"and\": 
                    [
                        {
                            \">\": {\"timestamp\": \"2013-12-01T18:00:00\"}
                        }, 
                        {
                            \"<\": {\"timestamp\": \"2013-12-01T18:15:00\"}
                        }
                    ]
                }, 
                {
                    \"and\": 
                    [
                        {
                            \">\": {\"timestamp\": \"2013-12-01T18:30:00\"}
                        }, 
                        {
                            \"<\": {\"timestamp\": \"2013-12-01T18:45:00\"}
                        }
                    ]
                }
            ]
        }
    ]
}",
"orderby" : "[{\"counter_volume\": \"ASC\"}, {\"timestamp\": \"DESC\"}]",
"limit" : 4
}
```

可支持多条件多层嵌套，但需要注意对`"`做转义，毕竟整个`filter`是一个字符串。

### 接口样例
1. 查询指标项`cpu`的性能数据，按`cpu`数值大小升序、时间戳降序排列，返回第一条数据：
```
URL:    http://10.64.93.67:8777/v2/query/samples
方法：  POST
HEADER：User-Agent:python-ceilometerclient
BODY:   {
            "filter": "{\"=\": {\"counter_name\": \"cpu\"}}", 
            "orderby": "[{\"counter_volume\": \"ASC\"}, {\"timestamp\": \"DESC\"}]", 
            "limit": 1
        }
结果：  [
            {
                "counter_name": "cpu", 
                "user_id": "9b0aa021c55345d3925d9fc9d9886990", 
                "resource_id": "4bbf8f4d-2508-4782-ab6a-90e19f3227f7", 
                "timestamp": "2014-11-28T09:10:57+00:00", 
                "resource_metadata": 
                {
                    "ramdisk_id": "None", 
                    "flavor.vcpus": "1", 
                    "flavor.ephemeral": "0", 
                    "display_name": "testVM", 
                    "flavor.ram": "512", 
                    "OS-EXT-AZ:availability_zone": "nova", 
                    "ephemeral_gb": "0", 
                    "flavor.name": "m1.tiny", 
                    "disk_gb": "1", 
                    "kernel_id": "None", 
                    "image.id": "9870b5a8-663b-493d-b4b8-680cf9a944b7", 
                    "flavor.id": "1", 
                    "host": "74b35cc12ed701930bfa7ae9a6e8128b4d4529280344c1cac9be1ab4", 
                    "image.name": "CirrOS 0.3.1", 
                    "image_ref_url": "http://controller:8774/abf7d168ee54403cbd9c26f1be1645be/images/9870b5a8-663b-493d-b4b8-680cf9a944b7", 
                    "cpu_number": "1", 
                    "flavor.disk": "1", 
                    "root_gb": "1", 
                    "name": "instance-00000011", 
                    "memory_mb": "512", 
                    "instance_type": "1", 
                    "vcpus": "1", 
                    "image_ref": "9870b5a8-663b-493d-b4b8-680cf9a944b7"
                }, 
                "source": "6d478c042ac8440cb1aad7e4b81dba2f:openstack", 
                "counter_unit": "ns", 
                "counter_volume": 1353290000000.0, 
                "project_id": "6d478c042ac8440cb1aad7e4b81dba2f", 
                "message_id": "529d313a-7e22-11e4-afc2-286ed4894417", 
                "counter_type": "cumulative"
            }
        ]
```

    
    
    





