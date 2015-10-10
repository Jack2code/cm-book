# Ceilometer源码分析 - Compute Agent

标签（空格分隔）： Ceilometer

---
[TOC]

## 概述
Ceilometer通过Agent模块去polling虚拟机或者OpenStack中需要的信息，然后将它传送至Ceilometer Event Bus中去。对于虚拟机的具体信息（CPU,Memory,Disk I/O,Network I/O）需要去虚拟机所运行的节点上获取（其实使用libvirt可以直接tcp到那个节点上），所以需要将Agent模块丢到计算机点上，然后通过Libvirt获取虚拟机状态，这就是Compute Agent的功能。
本文针对`Havana`版本。

## 入口文件
Compute Agent以OpenStack Service（OpenStack服务分为Service和WSGI Service）的形式运行在计算节点上，它由一个可执行文件开启，这个可执行文件安装时生成。
入口文件是[setup.cfg](http://rnd-github.huawei.com/openstack/ceilometer/blob/havana-eol/setup.cfg)，找到配置项`console_scripts`：

```
console_scripts = 
	ceilometer-api = ceilometer.api.app:start
	ceilometer-agent-central = ceilometer.central.manager:agent_central
	ceilometer-agent-compute = ceilometer.compute.manager:agent_compute
	ceilometer-dbsync = ceilometer.storage:dbsync
	ceilometer-expirer = ceilometer.storage:expirer
	ceilometer-collector = ceilometer.collector.service:collector
	ceilometer-collector-udp = ceilometer.collector.service:udp_collector
	ceilometer-alarm-evaluator = ceilometer.alarm.service:alarm_evaluator
	ceilometer-alarm-notifier = ceilometer.alarm.service:alarm_notifier
ceilometer.dispatcher = 
	database = ceilometer.collector.dispatcher.database:DatabaseDispatcher
	file = ceilometer.collector.dispatcher.file:FileDispatcher
```
这里表明`ceilometer-agent-compute`从`ceilometer.compute.manager:agent_compute`运行。

## 入口函数
根据`ceilometer.compute.manager:agent_compute`可知，入口函数为`ceilometer\compute\manager.py`文件里的`agent_compute`函数：

```python
def agent_compute():
    service.prepare_service()
    os_service.launch(compute_manager.AgentManager()).wait()
```
`agent_compute`函数包含两行代码，其中第一行代码主要是准备工作，调用`prepare_service()`函数：

```python
def prepare_service(argv=None):
    gettextutils.install('ceilometer', lazy=True)
    gettextutils.enable_lazy()
    rpc.set_defaults(control_exchange='ceilometer')
    cfg.set_defaults(log.log_opts,
                     default_log_levels=['amqplib=WARN',
                                         'qpid.messaging=INFO',
                                         'sqlalchemy=WARN',
                                         'keystoneclient=INFO',
                                         'stevedore=INFO',
                                         'eventlet.wsgi.server=WARN',
                                         'iso8601=WARN'
                                         ])
    if argv is None:
        argv = sys.argv
    cfg.CONF(argv[1:], project='ceilometer')
    log.setup('ceilometer')
```

真正实现功能的是最后一行

```python
os_service.launch(compute_manager.AgentManager()).wait()
```

再把`launch`函数展开，

```python
from ceilometer.compute import manager as compute_manager
from ceilometer.openstack.common import service as os_service

ceilometer.openstack.common.service.launch(
    ceilometer.compute.manager.AgentManager()
).wait()
```

`ceilometer.compute.manager`（继承自`ceilometer.agent.AgentManager`，文件路径：[ceilometer\agent.py](http://rnd-github.huawei.com/openstack/ceilometer/blob/havana-eol/ceilometer/agent.py)）执行`AgentManager()`

```python
from ceilometer import agent

class AgentManager(agent.AgentManager):

    def __init__(self):
        super(AgentManager, self).__init__('compute', ['local_instances'])
        self._inspector = virt_inspector.get_hypervisor_inspector()

    @property
    def inspector(self):
        return self._inspector
```

`launch`方法包含两个非常重要的类，`ServiceLauncher`和`ProcessLauncher`

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

