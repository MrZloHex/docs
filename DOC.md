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
 	* [Database Design](#databse-design)
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


