# SOUL

## What is the Soul?

This is an asynchronous, high-performance, cross-language, responsive API gateway.

## Features
 - Support various languages (http protocol), support dubbo, spring-cloud, Grpc, Motan, Sofa, Tars protocol.
 - Plugin design idea, plugin hot swap, easy to expand.
 - Flexible flow filtering to meet various flow control.
 - Built-in rich plugin support, authentication, limiting, fuse, firewall, etc.
 - Dynamic flow configuration, high performance.
 - Support cluster deployment, A/B Test, blue-green release.

### Architecture Diagram

![](img/diagrams/soul-framwork.png)


## Table of Contets
 * [What is the Soul](#what-is-the-soul)
 * [Features](#features)
 	* [Architecture](#architecture-diagram)
 * [Design](#design)
 	* [Database Design](#database-design)
 	* [Config](#configuration-flow-introduction)
 	* [DataSync]()
 	* [MetaData]()
 * [Admin]()

## Design

### Database Design

 - Plugin use database to store plugin, selector, rule configuration data and relationship.
 - The Database Table UML Diagram: 
 ![](img/diagrams/database_table.png)
 - Detailed design:
	* One plugin corresponds to multiple selectors,one selector corresponds to multiple rules.
  	* One selector corresponds to multiple match conditions,one rule corresponds to multiple match conditions.
  	* Each rule handles differently in corresponding plugin according to field handler,field handler is a kind of data of JSON string type.You can view detail during the use of admin.
  	* Plugin use database to store user name,role,resource data and relationship.
 - The Permission Table UML Diagram: 
 ![](img/diagrams/database_permissions.png)
 - Detailed design:
  	* one user corresponds to multiple role,one role corresponds to multiple resources.

### Configuration Flow Introduction

#### Description

This article introduces the flow of synchronizing to the gateway after the data operation of admin backend system.

#### Usage

 - User can arbitrary modify data in soul-admin backend and this will immediately synchronize to the jvm memory of the gateway.
 - Synchronize the plugin data of soul,selector,rule data, metadata, signature data, etc.
 - All the rules of plugin selectors are dynamically configured and take effect immediately without restarting the service.
 - Data Flow Chart: 
 ![](img/diagrams/data_flow.png)

#### Feature
 
 - All the configurations of user can be dynamically updated, there is no need to restart the service for any modification.
 - Local cache is used to provide efficient performance during high concurrency.


### Data Synchronization Design

#### Description

This article mainly explains three ways of database synchronization and their principles.

#### Preface

Gateway is the entrance of request and it is a very important part in micro service architecture, therefore the importance of gateway high availability is self-evident. When we use gateway, we have to change configuration such as flow rule, route rule for satisfying business requirement. Therefore, the dynamic configuration of the gateway is an important factor to ensure the high availability of the gateway. Then, how does `Soul` support dynamic configuration?

Anyone who has used `Soul` knows, `Soul` plugin are hot swap ,and the selector, rule of all plugins are dynamic configured, they take effect immediately without restarting service.But during using `Soul` gateway, users also report many problems.

 - Rely on `zookeeper`, this troubles users who use `etcd` `consul` and `nacos` registry
 - Rely on `redis`, `influxdb`, I have not used the limiting plugin, monitoring plugin, why do I need these

Therefore,we have done a partial reconstruction of `Soul`, after two months of version iteration, we released version `2.0`

 - Data Synchronization removes the strong dependence on `zookeeper`,and we add `http long polling` and `websocket`
 - Limiting plugin and monitoring plugin realize real dynamic configuration, we use `admin` backend for dynamic configuration instead of `yml` configuration before

*Q: Someone may ask me,why don’t you use configuration center for synchronization?*

First of all, it will add extra costs, not only for maintenance, but also make `Soul` heavy; In addition, using configuration center, data format is uncontrollable and it is not convenient for `soul-admin` to do configuration management.

*Q: Someone may also ask,dynamic configuration update?Every time I can get latest data from database or redis,why are you making it complicated?*

As a gateway, soul cached all the configuration in the `HashMap` of JVM in order to provide higher response speed and we use local cache for every request, It’s very fast. So this article can also be understood as three ways of memory synchronization in a distributed environment.