# OpenConfig：合作使可编程网络管理成为可能

标签（空格分隔）： OpenConfig

---
[TOC]

[原文链接][1]

## 管理大规模网络的挑战

- 20+网络设备角色
- 超过半打的厂商，多平台
- 400万行的配置文件
- 每月达到约3万的配置修改量
- 每5分钟超过800万OID被收集
- 第5分钟超过2万命令行命令下发及删除
- 更多的工具，以及多个版本的软件

> Opportunity for significant OPEX savings: reduced outage impact, simplification of management stack, automation / self-healing, better scaling...

## 管理平面的元素
- 通用的配置API和监控（多厂商设备）
- 传输和RPC协议是开放、流式、安全的
- 互操作性的整网视图

### 模型驱动的网络管理
- 配置Configuration -- 描述配置的数据结构和内容
- 遥测Telemetry -- 描述监控数据的结构和属性
- 拓扑Topology -- 描述网络结构

## 遥测框架要求
- 网络元素流式数据到收集器（推送模型）
- 数据填充基于厂商中立的模型
- 发布/订阅API来选择需要的数据
- 满足未来十年高频数据密度增长的规模
- 拥有活跃开发社区的现代传输机制
    - 例如gRPC(HTTP/2),Thrift,protobuf over UDP

## OpenConfig动机
- 管理接口是厂商无关，平台无关，版本具体的
    - NETCONF/RESTCONF，CIM，SNMP没有解决这个问题
    - 自动化框架（Puppet，Chef，Ansible等）没有解决这个问题
- 复杂性和成本转移给操作者
    - 对所有专有的变化必须有构建、生成和测试工具 
    - 配置和监控标准协议和服务没有不必要的差异
    - 需要专门的技能来处理专有的差异

## OpenConfig：用户定义API
- 网络运营商的正规行业协作
- 重点：根据实际操作定义厂商中立的配置和运行状态模型
- 主要输出是模型代码，通过公共GitHub库发布为开放源代码
- 与主要供应商建立伙伴关系，以推动本地实现
    - 完全支持和维护为一体的平台软件部分
    - 提供给所有的客户，没有“特价”
- 参与标准（IETF，ONF）和OSS项目（ODL，ONOS，NTT）

## 为什么需要行业合作
- 拓宽用例超越了任何一个运营商/客户
- 简化的销售商 - 巩固客户的要求
- 通过广泛的审查和开放的过程改进模型
- 集体的力量来驱动的发展模式
- 确保不同管理/ NMS方法是否切合实际

## OpenConfig参与者
广泛的使用案例，网络环境中，供应商的部署，服务和商业模式

Level(3)/BT/at&t/xfinity/cox/google/facebook/Microsoft/Apple/Yahoo

## OpenConfig治理
短版本：暂无

- 没有董事会，指导委员会章程，...
    - 避免法律协议，认证等。
    - 依靠良好的行为，透明度和共同的目标
- OpenConfig参与者加入每周一次的工作会议
    - “参加” == 工程师/设计师提交和评审标准守则
- 在github上或邮件列表提出问题/讨论模型
- 在Apache许可下发布模型的代码和工具

## “合作创新”
> “事实上，这些明显不同（而且往往具有竞争力） 服务供应商正在合作，是他们感到紧急的迹象”
> - LightReading，The New IP，2015年二月 

> “[OpenConfig]用来为制定扭结转动规范了官方财团之前提供一个试验场。”
> - siliconAngle，2015年6月

## OpenConfig开发过程
![OpenConfig开发过程][1]

## OpenConfig进展
### 数据模型（配置和运行状态）
- BGP和路由策略
    - 多个厂商正在实现
    - BGP模型被IETF接纳进行标准跟踪
- 本地路由（本地生成的静态路由，聚合等）
- MPLS/TE综合模型
    - RSVP/TE和分段路由作为初始焦点
- 设备型号 -- 通用结构构成的模型

### 设计模式和可用性方面的改进
- 设计模式的运行状态和模型组成
- 型号目录建议

### 模型目前正在审查
- 更新接口和系统模型
- RIB模型 -- 代表通用格式的路由表
- 光传输设备（传输SDN）

### 工具和API
- pyangbind -- 从YANG模型生成Python类
- 协议无关的规范的配置和遥测RPC的本机实现 -- BGP + 策略模型
    - 思科IOS-XR
    - 瞻博JUNOS
    - 其他正在实现的厂商

## 模型必须是有用的集合
- 模型构成的框架是现有的模式建设工作的关键缺少的部分
- 如何支持构建组合到建模语言

## 建模工作状态
### 运行状态数据的类型
- 派生，协商，由协议等设置（协商BGP保持时间）
- 运行状态数据计数器或统计信息（接口计数器）
- 运行状态数据代表的应用配置（配置实际对比）

### 使用YANG明显的好处是配置和运行状态在同一个数据模型建模
- 提供跨设备的通用结构的监测数据
- 轻松关联配置与相应的状态
- 但是......YANG重点主要是配置，NETCONF为中心，缺乏共同的约定

## 关于YANG/NETCONF的观察
- YANG和NETCONF应该解耦--每一个都是有单独作用的
- YANG需要在早期更快发展，随着实际使用增长保持稳定
- YANG需要审查以及从一个更大范围的用户的不同视角输入
- 当前YANG模型版本管理并没有什么用处--以软件产品的方式来对待模型，而不是注明日期的文档
- 当前“标准”模型应该开放出来重新审视和修订，避免更多的模型被部署到生产环境中使用前仓促标准化

这些观点OpenConfig还没达成一致

## OpenDaylight机会
- ODL网络管理系统
    - 网络管理系统作为ODL平台一流的用例
    - 鼓励聚焦于更多的管理和运营功能
- 支持OpenConfig
    - 使用OpenConfig发布的模型作为ODL能力的接口 
- YANG工具化和生态系统
    - 通用的YANG模型的工具链
    - 使模型语言功能的试验成为现实

有些已经正在发生：）
对其他SDN相关的OSS项目有相同的机会：例如OPNFV，ONOS

## OpenDaylight网络管理系统
- ODL已经支持一些管理/运营功能
    - 监控和路径管理：SNMP，BGP-LS/PCEP
    - 网络配置：NETCONF/RESTCONF，OVSDB
    - 数据管理和建模：YANG工具，时序数据仓库
- 潜在的附加功能
    - 发布/订阅模式的流遥测收集器
    - 支持额外数据传输和编码
    - 配置校验

## ODL中对OpenConfig模型的支持
配置和监控API基于OpenConfig模型。
- BGP和路由策略
    - ODL BGP接口实现
    - 一些正在进行的进展（例如，IETF93 hackthon）
- MPLS/TE
    - 与PCEP集成，分段路由

## 开发YANG生态系统
- 交叉验证YANG工具和模型
- 使YANG模型更易于可视化和试验
- 从YANG模型生成的代码构件的一致性
    - 例如，类绑定Java,Python,Go等
- 改进了YANG模型语言
    - 用户/执行者的角度来补充IETF标准
    - 地址重大缺陷（列表，版本控制，选择器，模型组成）
        - 可以看看Colin Dixon在ONS 2015关于YANG/ODL的演讲

## 总结
- 网络管理需要一个模型驱动的途径来进入SDN和可编程网络的时代
- OpenConfig是一种新型的行业合作
    - 网络运营商直接贡献开源数据模型，工具和设计模式
- 作为本机的实现变得可用，潜在显著改变网络监控和配置
- 为OpenDaylight和其他开源软件项目的重要作用，帮助实现愿景

      [1]: http://www.slideshare.net/aashaikh/openconfig-collaborating-to-enable-programmable-network-management

  [1]: http://static.zybuluo.com/Jack2code/ey96h0okc0kmhifn9ypjrmem/openconfig-collaborating-to-enable-programmable-network-management-10-638.jpg