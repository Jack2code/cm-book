# Ceilometer源码分析 - Central Agent

 - 列表项

标签（空格分隔）： Ceilometer 
---
[TOC]

## 概述

 - 列表项

Central Agent运行在控制节点上，它主要收集OpenStack其它服务(Glance，Swift，Cinder，Neutront等)的信息，通过调用这些服务的REST API去获取性能数据。

本文针对Ceilometer Havana版本。

## 入口文件
Central Agent通过OpenStack各个服务组件提供的API采集有用的信息，需要定期通过Pollsters轮询采集信息。
入口文件是：[setup.cfg](http://rnd-github.huawei.com/openstack/ceilometer/blob/havana-eol/setup.cfg)，找到配置项`console_scripts`：

```
console_scripts =
    ceilometer-api = ceilometer.cmd.api:main
    ceilometer-agent-central = ceilometer.cmd.agent_central:main
    ceilometer-agent-compute = ceilometer.cmd.agent_compute:main
    ceilometer-agent-notification = ceilometer.cmd.agent_notification:main
    ceilometer-agent-ipmi = ceilometer.cmd.agent_ipmi:main
    ceilometer-send-sample = ceilometer.cli:send_sample
    ceilometer-dbsync = ceilometer.cmd.storage:dbsync
    ceilometer-expirer = ceilometer.cmd.storage:expirer
    ceilometer-rootwrap = oslo.rootwrap.cmd:main
    ceilometer-collector = ceilometer.cmd.collector:main
    ceilometer-alarm-evaluator = ceilometer.cmd.alarm:evaluator
    ceilometer-alarm-notifier = ceilometer.cmd.alarm:notifier
```
由此可以看到central-agent对应的配置项名称是`ceilometer-agent-central`，而 `ceilometer-agent-central`对应的函数是`ceilometer.cmd.agent_central:main`。

## 入口函数
根据`ceilometer.cmd.agent_central:main`可知，Central Agent的入口函数在`ceilometer\cmd\agent_central.py`文件里的`main`函数。不同版本可能不同，具体在此不展开。

`main`函数代码如下所示，只有短短两行，

```python
def main():
    service.prepare_service()
    os_service.launch(manager.AgentManager()).wait()
```

其中，第一行主要是一些准备工作，依靠`prepare_service()`函数来完成，

```python
def prepare_service(argv=None):
    gettextutils.install('ceilometer')
    gettextutils.enable_lazy()
    log_levels = (cfg.CONF.default_log_levels +
                  ['stevedore=INFO', 'keystoneclient=INFO'])
    cfg.set_defaults(log.log_opts,
                     default_log_levels=log_levels)
    if argv is None:
        argv = sys.argv
    cfg.CONF(argv[1:], project='ceilometer')
    log.setup('ceilometer')
    messaging.setup()
```

`main`函数第二行才是主要的工作，`central agent`的`AgentManager`类继承自`agent.py`模块的`AgentManager`类，构造函数如下：
```python
def __init__(self, namespace, default_discovery=None, group_prefix=None):
    super(AgentManager, self).__init__()
    default_discovery = default_discovery or []
    self.default_discovery = default_discovery
    self.pollster_manager = self._extensions('poll', namespace)
    self.discovery_manager = self._extensions('discover')
    self.context = context.RequestContext('admin', 'admin', is_admin=True)
    self.partition_coordinator = coordination.PartitionCoordinator()
    self.group_prefix = ('%s-%s' % (namespace, group_prefix)
                    if group_prefix else namespace)
```
其中，有这么一句
```python
self.pollster_manager = self._extensions('poll', namespace)
```
这是一种动态加载模块的机制，它将载入所有它可以poll的模块。通过`_extensions`方法来组装namespace：
```python
def _extensions(category, agent_ns=None):
        namespace = ('ceilometer.%s.%s' % (category, agent_ns) if agent_ns
                     else 'ceilometer.%s' % category)
        return extension.ExtensionManager(
            namespace=namespace,
            invoke_on_load=True,
        )
```
而在central agent的构造函数中，
```python
        super(AgentManager, self).__init__(
            'central', group_prefix=cfg.CONF.central.partitioning_group_prefix)
```
由上可知，最后得到的namespace是`ceilometer.poll.central`。而从`setup.cfg`中可以得到`ceilometer.poll.central`相对应的配置项：
```python
ceilometer.poll.central =
    ip.floating = ceilometer.network.floatingip:FloatingIPPollster
    image = ceilometer.image.glance:ImagePollster
    image.size = ceilometer.image.glance:ImageSizePollster
    storage.containers.objects = ceilometer.objectstore.swift:ContainersObjectsPollster
    ...
```
这样central agent就知道它要调用哪些Pollster来获取信息。

然后通过调用ceilometer.openstack.common.service的`launch`函数来加载所有的pollster，`launch`函数在`ceilometer\openstack\common\service.py`文件里，

```python
def launch(service, workers=1):
    if workers is None or workers == 1:
        launcher = ServiceLauncher()
        launcher.launch_service(service)
    else:
        launcher = ProcessLauncher()
        launcher.launch_service(service, workers=workers)

    return launcher
```

由`launch`函数可以看到，默认工作进程`workers=1`，

```flow
st=>start: Start
cond=>condition: workers is None or workers == 1?
sl=>operation: ServiceLauncher()
pl=>operation: ProcessLauncher()
ls=>operation: launch_service
e=>end

st->cond
cond(yes)->sl->ls->e
cond(no)->pl->ls->e
```

其中，`ServiceLauncher`类和`ProcessLauncher`类均继承于`Launcher`类，`ServiceLauncher`类调用的是`Launcher`类的`launch_service(service)`函数，`ProcessLauncher`类调用的是自己重写父类的`launch_service(self, service, workers=1)`函数。

由代码
```python
os_service.launch(manager.AgentManager()).wait()
```
可知，加载所有的pollster后，将会调用`Launcher`（或子类）的`wait()`方法来启动pollsters。


## 数据存库
```python
ceilometer.metering.storage =
    log = ceilometer.storage.impl_log:Connection
    mongodb = ceilometer.storage.impl_mongodb:Connection
    mysql = ceilometer.storage.impl_sqlalchemy:Connection
    postgresql = ceilometer.storage.impl_sqlalchemy:Connection
    sqlite = ceilometer.storage.impl_sqlalchemy:Connection
    hbase = ceilometer.storage.impl_hbase:Connection
    db2 = ceilometer.storage.impl_db2:Connection
```
由`setup.cfg`里可知，`mongodb`对应的数据库连接函数是`mongodb = ceilometer.storage.impl_mongodb:Connection`，即`ceilometer\storage\impl_mongodb.py`里的`Connection`类。

其实，存库的函数就是`record_metering_data`，代码如下：
```python
def record_metering_data(self, data):
    """Write the data to the backend storage system.

    :param data: a dictionary such as returned by
                 ceilometer.meter.meter_message_from_counter
    """
    # Record the updated resource metadata - we use $setOnInsert to
    # unconditionally insert sample timestamps and resource metadata
    # (in the update case, this must be conditional on the sample not
    # being out-of-order)
    resource = self.db.resource.find_and_modify(
        {'_id': data['resource_id']},
        {'$set': {'project_id': data['project_id'],
                  'user_id': data['user_id'],
                  'source': data['source'],
                  },
         '$setOnInsert': {'metadata': data['resource_metadata'],
                          'first_sample_timestamp': data['timestamp'],
                          'last_sample_timestamp': data['timestamp'],
                          },
         '$addToSet': {'meter': {'counter_name': data['counter_name'],
                                 'counter_type': data['counter_type'],
                                 'counter_unit': data['counter_unit'],
                                 },
                       },
         },
        upsert=True,
        new=True,
    )

    # only update last sample timestamp if actually later (the usual
    # in-order case)
    last_sample_timestamp = resource.get('last_sample_timestamp')
    if (last_sample_timestamp is None or
            last_sample_timestamp <= data['timestamp']):
        self.db.resource.update(
            {'_id': data['resource_id']},
            {'$set': {'metadata': data['resource_metadata'],
                      'last_sample_timestamp': data['timestamp']}}
        )

    # only update first sample timestamp if actually earlier (the unusual
    # out-of-order case)
    # NOTE: a null first sample timestamp is not updated as this indicates
    # a pre-existing resource document dating from before we started
    # recording these timestamps in the resource collection
    first_sample_timestamp = resource.get('first_sample_timestamp')
    if (first_sample_timestamp is not None and
            first_sample_timestamp > data['timestamp']):
        self.db.resource.update(
            {'_id': data['resource_id']},
            {'$set': {'first_sample_timestamp': data['timestamp']}}
        )

    # Record the raw data for the meter. Use a copy so we do not
    # modify a data structure owned by our caller (the driver adds
    # a new key '_id').
    record = copy.copy(data)
    record['recorded_at'] = timeutils.utcnow()
    self.db.meter.insert(record)
```
由`record_metering_data`的代码可知，存库时涉及4张表，分别是`user`、`project`、`resource`、`meter`，其中，`user`、`project`、`resource`都只是做更新操作，`meter`做插入操作。操作通过调用monogodb的api进行操作。