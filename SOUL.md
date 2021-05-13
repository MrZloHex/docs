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
 	* [DataSync](c)
 	* [MetaData](#metadata-concept-design)
 * [Admin](#admin)
 	* [Dict Management](#dict-management)
 	* [Plugin Handle Explanation](#plugin-handle-explanation)
 	* [Selector and Rule](#selector-detailed-explanation)
 * [Register Center](#register-center)
 	* [Register Center Design](#register-center-design)
 	* [Access](#register-center-access)
 * [User Guide](#user-guide)
 	* [Set up](#environment-setup)
 	* [Use Data Sync](#use-different-data-sync-strategy)	
 * [Developer Guide](#developer-guide)
 	* [Custom Filter](#filter-extension)
 	* [File and Image](#file-uploading-and-downloading)
 	* [Custom parsing IP and Host](#fetching-correct-ip-address-and-host)
 	* [Custom Sign Algorithm](#custom-sign-algorithm)
 	* [Developer Soul Client](#a-multilingual-http-client)
 	* [Tread](#thread-model)
 	* [Soul Optimize](#soul-optimize)

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

#### Principle Analysis

This is a HD uncoded image, it shows the flow of `Soul` data synchronization, when `Soul` gateway starts, it will synchronize configuration data from the configuration service and support push-pull mode to obtain configuration change information, and update the local cache.When administrator changes user,rule,plugin,flow configuration in the backend, modified information will synchronize to the `Soul` gateway through the push-pull mode,whether it is the push mode or the pull mode depends on the configuration.About the configuration synchronization module, it is actually a simplified configuration center. 

![](img/diagrams/soul_admin_config.png)

At version `1.x` ,configuration service depends on `zookeeper`,management backend `push` the modified information to gateway.But version `2.x` supports `webosocket`,`http`,`zookeeper`,it can specify the corresponding synchronization strategy through `soul.sync.strategy` and use `webosocket` synchronization strategy by default which can achieve second-level data synchronization.But,note that `soul-web` and `soul-admin` must use the same synchronization mechanism.

As showing picture below,`soul-admin` will issue a configuration change notification through `EventPublisher` after users change configuration,`EventDispatcher` will handle this modification and send configuration to corresponding event handler according to configured synchronization strategy(http,websocket,zookeeper)

 - If it is a `websocket` synchronization strategy, it will push modified data to `soul-web`,and corresponding `WebsocketCacheHandler` handler will handle `admin` data push at the gateway layer
 - If it is a `zookeeper` synchronization strategy, it will push modified data to `zookeeper`,and the `ZookeeperSyncCache` will monitor the data changes of `zookeeper` and process them
 - If it is a `http` synchronization strategy,`soul-web` proactively initiates long polling requests,90 seconds timeout by default,if there is no modified data in `soul-admin`,http request will be blocked,if there is a data change, it will respond to the changed data information,if there is no data change after 60 seconds,then respond with empty data,gateway continue to make http request after getting response,this kind of request will repeat 

![](img/diagrams/synchonization_type.png)

#### Zookeeper Synchronization

The zookeeper-based synchronization principle is very simple,it mainly depends on `zookeeper` watch mechanism,`soul-web` will monitor the configured node,when 	`soul-admin` starts, all the data will be written to `zookeeper`,it will incrementally update the nodes of `zookeeper` when data changes,at the same time, `soul-web` will monitor the node for configuration information, and update the local cache once the information changes.

![](img/diagrams/zookepear_sync.png)

`soul` writes the configuration information to the zookeeper node,and it is meticulously designed.

#### WebSocket 

The mechanism of `websocket` and `zookeeper` is similar,when the gateway and the `admin` establish a `websocket` connection,`admin` will push all data at once,it will automatically push incremental data to `soul-web` through `websocket` when configured data changes

When we use websocket synchronization,pay attention to reconnect after disconnection, which also called keep heartbeat.`Soul` uses `java-websocket` ,a third-party library,to connect to `websocket`.

```java
public class WebsocketSyncCache extends WebsocketCacheHandler {
    /**
     * The Client.
     */
    private WebSocketClient client;

    public WebsocketSyncCache(final SoulConfig.WebsocketConfig websocketConfig) {
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(1,
                SoulThreadFactory.create("websocket-connect", true));
         client = new WebSocketClient(new URI(websocketConfig.getUrl())) {
                @Override
                public void onOpen(final ServerHandshake serverHandshake) {
                  //....
                }
                @Override
                public void onMessage(final String result) {
                  //....
                }    
            };
        //connect
        client.connectBlocking();
        //reconnect after disconnection,using scheduling thread pool,execute every 30 seconds
        executor.scheduleAtFixedRate(() -> {
            if (client != null && client.isClosed()) {
                    client.reconnectBlocking();
            }
        }, 10, 30, TimeUnit.SECONDS);
 	}
```

#### Http Long Polling

The mechanism of zookeeper and websocket data synchronization is relatively simple,but http synchronization will be relatively complicated.Soul borrows the design ideas of `Apollo` and `Nacos` and realizes `http` long polling data synchronization using their advantages.Note that this is not traditional ajax long polling.

![http long polling mechanism as above,soul-web gateway requests admin configuration services,timeout is 90 seconds,it means gateway layer request configuration service will wait at most 90 seconds,this is convenient for admin configuration service to respond modified data in time,and therefore we realize near real-time push.](img/diagrams/http_sync.png)

After the http request reaches soul-admin, it does not respond immediately,but uses the asynchronous mechanism of Servlet3.0 to asynchronously respond to the data.First of all,put long polling request task `LongPollingClient` into `BlocingQueue`,and then start scheduling task,execute after 60 seconds,this aims to remove the long polling request from the queue after 60 seconds,even there is no configured data change.Because even if there is no configuration change,gateway also need to know,otherwise it will wait,and there is a 90 seconds timeout when the gateway requests configuration services.

```java
public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {
    // since soul-web may not receive notification of a configuration change, MD5 value may be different,so respond immediately
    List<ConfigGroupEnum> changedGroup = compareMD5(request);
    String clientIp = getRemoteIp(request);
    if (CollectionUtils.isNotEmpty(changedGroup)) {
        this.generateResponse(response, changedGroup);
        return;
    }

    // Servlet3.0 asynchronously responds to http request
    final AsyncContext asyncContext = request.startAsync();
    asyncContext.setTimeout(0L);
    scheduler.execute(new LongPollingClient(asyncContext, clientIp, 60));
}
    
class LongPollingClient implements Runnable {
    LongPollingClient(final AsyncContext ac, final String ip, final long timeoutTime) {
        // omit......
    }
    @Override
    public void run() {
        // join a scheduled task, if there is no configuration change within 60 seconds, it will be executed after 60 seconds and respond to http requests
        this.asyncTimeoutFuture = scheduler.schedule(() -> {
            // clients are blocked queue,saved the request from soul-web
            clients.remove(LongPollingClient.this);
            List<ConfigGroupEnum> changedGroups = HttpLongPollingDataChangedListener.compareMD5((HttpServletRequest) asyncContext.getRequest());
            sendResponse(changedGroups);
        }, timeoutTime, TimeUnit.MILLISECONDS);
        // 
        clients.add(this);
    }
}
```

If the administrator changes the configuration data during this period,the long polling requests in the queue will be removed one by one, and respond which group’s data has changed(we distribute plugins, rules, flow configuration , user configuration data into different groups).After gateway receives response,it only knows which Group has changed its configuration,it need to request again to get group configuration data.Someone may ask,why don’t you write out the changed data directly?We also discussed this issue deeply during development, because the http long polling mechanism can only guarantee quasi real-time,if gateway layer does not handle it in time,or administrator updates configuration frequently,we probably missed some configuration change push.For security, we only inform that a certain Group information has changed.

```java
// soul-admin configuration changed,remove the requests from the queue one by one and respond to them
class DataChangeTask implements Runnable {
    DataChangeTask(final ConfigGroupEnum groupKey) {
        this.groupKey = groupKey;
    }
    @Override
    public void run() {
        try {
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext(); ) {
                LongPollingClient client = iter.next();
                iter.remove();
                client.sendResponse(Collections.singletonList(groupKey));
            }
        } catch (Throwable e) {
            LOGGER.error("data change error.", e);
        }
    }
}
```

When `soul-web` gateway layer receives the http response information,pull modified information(if exists),and then request `soul-admin` configuration service again,this will repeatedly execute.

#### At Last

This article introduces that,in order to optimize the response speed, `soul` as a highly available micro service gateway, its three ways to cache the configuration rule selector data locally.After learning this article,I believe you have a certain understanding of the popular configuration center,it may be easier to learn their codes,I believe you can also write a distributed configuration center.Version 3.0 is already under planning,and I believe it will definitely surprise you.

### MetaData Concept Design

#### Description

This article mainly explains the concept,design of metadata and how to connect in the soul gateway.

#### Technical Solutions

 - Add a new table in the database,and data can synchronize to the JVM memory of gateway according to the data synchronization scheme.
 - Table Structure:

```sql
CREATE TABLE  IF NOT EXISTS `meta_data` (
`id` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'id',
`app_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'application name',
`path` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'path,not repeatable',
`path_desc` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'path description',
`rpc_type` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'rpc type',
`service_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT 'service name',
`method_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT 'method name',
`parameter_types` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT 'multiple parameter types, split by comma',
`rpc_ext` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT 'rpc extension information,json format',
`date_created` datetime(0) NOT NULL COMMENT 'create date',
`date_updated` datetime(0) NOT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT 'update date',
`enabled` tinyint(4) NOT NULL DEFAULT 0 COMMENT 'enable status',
PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci ROW_FORMAT = Dynamic;
```

 - Metadata design as below,the most important is using it in dubbo’s generalization call.
 - Pay attention to the field `path`,we will match specific data according to your field path during requesting gateway,and then carry out the follow-up process.
 - Pay attention to the field `rpc_ext`,if it is a dubbo service interface and service interface has group and version field,this field exists.
 - dubbo field structure as below,then we store json format string.

```java
public static class RpcExt {

    private String group;
    private String version;
    private String loadbalance;
    private Integer retries;
    private Integer timeout;
}
```

#### MetaDown Storage

 - A dubbo interface corresponds to a meta data.
 - SpringCloud protocol, only store one record, path: `/contextPath/**`.
 - Http service, no data.

## Admin

### Dict Management

#### Explanation

Dictionary management is primarily used to maintain and manage common data dictionaries.

#### Table design

 - sql

```sql
CREATE TABLE IF NOT EXISTS `soul_dict` (
   `id` varchar(128) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'primary key id',
   `type` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'type',
   `dict_code` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'dictionary encoding',
   `dict_name` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'dictionary name',
   `dict_value` varchar(100) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT 'dictionary value',
   `desc` varchar(255) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT 'dictionary description or remarks',
   `sort` int(4) NOT NULL COMMENT 'sort',
   `enabled` tinyint(4) DEFAULT NULL COMMENT 'whether it is enabled',
   `date_created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'create time',
   `date_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'update time',
   PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

 - The current usage scenario is when the plugin `handle` configuring the `data_type=3` `(select box)`

eg. `degradeRuleGrade` is one of fields of sentinel’s `handle` json

When it adds rules, it automatically looks up all the general dictionaries of `type='degradeRuleGrade'` in the `soul_dict` table as a select-box when you edit the General rules field

### Plugin Handle Explanation

#### Explanation

 - In our Soul-Admin background, each plugin uses the Handle field to represent a different processing, and plugin processing is used to manage and edit custom processing fields in JSON.
 - This feature is mainly used to support the plug-in handling template configuration

#### Table Design

```sql
CREATE TABLE IF NOT EXISTS `plugin_handle` (
`id` varchar(128) NOT NULL,
`plugin_id` varchar(128) NOT NULL COMMENT 'plugin id',
`field` varchar(100) NOT NULL COMMENT 'field',
`label` varchar(100) DEFAULT NULL COMMENT 'label',
`data_type` smallint(6) NOT NULL DEFAULT '1' COMMENT 'data type 1 number 2 string 3 select box',
`type` smallint(6) NULL COMMENT 'type, 1 means selector, 2 means rule',
`sort` int(4)  NULL COMMENT 'sort',
`ext_obj` varchar(1024) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT 'extra configuration (json format data)',
`date_created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'create time',
`date_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'update time',
PRIMARY KEY (`id`),
UNIQUE KEY `plugin_id_field_type` (`plugin_id`,`field`,`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE = utf8mb4_unicode_ci ROW_FORMAT = Dynamic;
```

#### Tutorial

eg. When we developed the `springCloud` plugin, the rule table needed to store some configuration into the handle field, Configure the corresponding entity class as follows:

```java
public class SpringCloudRuleHandle implements RuleHandle {
    
    /**
     * this remote uri path.
     */
    private String path;
    
    /**
     * timeout is required.
     */
    private long timeout = Constants.TIME_OUT;    
}
```

**step1**. We can go directly to the plug-in management link `http://localhost:9095/#/system/plugin` Click Edit Plugin for processing 

![](img/tutorials/admin/step_1.png)

**step2**. Add a string type field path and a numeric type TIMEOUT

![](img/tutorials/admin/step_2.png)

**step3**. Finally, you can enter path, TIMEOUT and commit to the handle field when you add a rule in the plugin rule configuration page

![](img/tutorials/admin/step_3.png)

*Note: If data_type is configured to be `3` `selection box`, the input field drop-down selection on the new rule page is displayed by going to the `soul_dict` table to find all the options available*

### Selector Detailed Explanation

#### Features

 - Selector and rule are the key point of soul gateway, you can manage any request with it.
 - This chapter is mainly focus on the concepts of selector and rule and how to use it.

#### Overview

 - One plugin has many selector and a selector has many rules, selector is the first filter of request, and the rule is the final filter.
 - Please consider this, it would be perfect when the plugin executes the request until it reached the config value.
 - Selector and rule are designed to execute only when the request meet the specific condition.
 - Please refer to the previous data structure [database design](#database-design).

#### Selector

![](img/examples/admin/selector.png)

 - selector detailed explanation：

   * name：create your selector with a distinguish name.
   * type：custom flow is customized request, full flow is full request. customized request will be handled by the conditions as below, while full request won’t.
   * match method: you can combine these conditions with ‘and’ , ‘or’ operators.
   * condition：
     - uri: filter request with uri method and support fuzzy matching (/\*\*).
     - header: filter request with request header.
     - query: filter request with query string.
     - ip: filter request with your real ip.
     - host: filter request with your real host.
     - post: not recommend to use.
     - condition match:
       * match : fuzzy string matching，recommend to combine with uri，support restful matching.（/test/\*\*).
       * = : if the values are the same, then they match.
       * regEx : regex matching，match characters in regex expression.
       * like : string fuzzy matching.
   * open option：only work with enabled.
   * print log：it will print the matching log with the open option enabled.
   * execution order：the smaller will have high priorty to execute among multi-selectors.
 - the above picture means: when the prefix of the request uri is `/test` and the value of `module` in `header` `istest`, it will redirect to this service `1.1.1.1:8080`.
 - selector advice : combine `uri` conditon and `match` prefix（/contextPath）as the first request filter.

#### Rule

![](img/examples/admin/rule.png)

 - when the request was passed by the seletor, then it will be processed by the rule, the final filter.
 - rule is the final confirmation about how to execute request logically.
 - rule detailed explanation：

   * name：create your rule with a distinguish name.
   * match method: you can combine these conditions with ‘and’ , ‘or’ operators.
   * condition：
     - uri: filter request with uri method and support fuzzy matching (/\*\*).
     - header: filter request with request header.
     - query: filter request with query string.
     - ip: filter request with your real ip.
     - host: filter request with your real host.
     - post: not recommend to use.
     - condition match:
       * match : fuzzy string matching，recommend to combine with uri，support restful matching.（/test/\*\*）
       * = : if the values are the same, then they match.
       * regEx : regex matching，match characters in regex expression.
       * like : string fuzzy matching.
   * open option：only work with enabled.
   * print log：it will print the matching log with the open option enabled.
   * execution order：the smaller will have high priorty to execute among multi-rules.
   * handle: different plugin has different execution method, pls refer to the specific one.
 - above picture means: when the request `uri` equals to `/http/order/save`, it will execute based on this rule，load strategy is `random`.
 - combine selector means ：when the request `uri` is `/http/order/save`, it will be redicted to `1.1.1.1:8080` by `random` method.
 - rule advice: combine `uri` condition with `match` the real `uri path` condition as the final filter.

#### Condition Explanation

 - uri matching （recommend）

 	* uri matching is based on your request uri, the frontend won’t change anything before accessing the gateway.
 	* the `match` filter method is the same with `springmvc` fuzzy matching.
 	* in selector，we recommend to match with the prefix of uri, and use the specific path in rule.
 	* when changing the match method, the matching field name can be filled randomly, but make sure the match value must be correct.
 - header matching

 	* header matches with your `http` request header value.
 - query matching

 	* it matches the query string in your uri, such as: `/test?a=1&&b=2`.
 	* so you can add a new condition, choose query method: `a = 1`.
 - ip matching

 	* it matches the ip of the http caller.
 	* especially in the waf plugin, if you find some ip is unsafe, you can add a match condition with this ip, then it can’t access any more.
 	* if you use nginx proxy before soul, you can get the right ip with refering to [Custom parsing IP and Host](#fetching-correct-ip-address-and-host)
 - host matching

 	* it matches the host of http caller.
 	* especially in waf plugin, if you find some host is unsafe, you can add a match condition with this host, then it can’t access any more.
 	* if you use nginx proxy before soul, you can get the right ip with refering to [Custom parsing IP and Host](#fetching-correct-ip-address-and-host)
 - post matching

 	* not recommend to use.

## Register Center

### Register Center Design

#### Description

This article mainly explains three ways of register center and their principles.

#### Client

![](img/diagrams/design_register_client.png)

When client server start, the register center client will be loaded by `spi`. Put data to `Disruptor` when spring bean load. `Soul` register client get data from `Disruptor`, and it will send request to register server. `Disruptor` can decouple data from operation and facilitate expansion.

#### Server

![](img/diagrams/design_register_server.png)

When `Soul-Admin` server start, register center server will be loaded by `spi`. Meanwile `Disruptor` will be inited too. `Soul` register server get data from register client, and then put then to `Disruptor`. `Soul-Admin` `Disruptor` consumer get data from register server by `Disruptor` queue, then save them to database and publish data synchronize event. `Disruptor` can decouple data from operation and buffering.

#### Http Registry

Principle of `http` register center is simple. Call interface of register server when `Soul-Client` start. `Soul-Admin` accept request, then save to database and publish data synchronize event.

#### Zookeeper Registry

Zookeeper storage struct is:

```sh
soul
   ├──regsiter
   ├    ├──metadata
   ├    ├     ├──${rpcType}
   ├    ├     ├      ├────${contextPath}
   ├    ├     ├               ├──${ruleName} : save metadata data of MetaDataRegisterDTO
   ├    ├──uri
   ├    ├     ├──${rpcType}
   ├    ├     ├      ├────${contextPath}
   ├    ├     ├               ├──${ip:prot} : save uri data of URIRegisterDTO
   ├    ├     ├               ├──${ip:prot}
```

`Zookeeper` register client will save data to `zookeeper` when `soul` client is started. `Zookeeper` register server will keep watching the change of data node. Trigger selector and rule data update and event will be published, when metadata data node update. Trigger selector and upstream update and event will be published, when `uri` data node update.

#### Etcd Registry

Etcd storage struct is:

```sh
soul
   ├──regsiter
   ├    ├──metadata
   ├    ├     ├──${rpcType}
   ├    ├     ├      ├────${contextPath}
   ├    ├     ├               ├──${ruleName} : save metadata data of MetaDataRegisterDTO
   ├    ├──uri
   ├    ├     ├──${rpcType}
   ├    ├     ├      ├────${contextPath}
   ├    ├     ├               ├──${ip:prot} : save uri data of URIRegisterDTO
   ├    ├     ├               ├──${ip:prot}
```

Etcd register client will save data to etcd when soul client is started. Etcd register server will keep watching the change of data node. Trigger selector and rule data update and event will be published, when metadata data node update. Trigger selector and upstream update and event will be published, when uri data node update.

#### Consul Registry

Consul register client will save `URIRegisterDTO` to service instance metadata, and `URIRegisterDTO` will disappear with service unregister.

![](img/examples/registry/consul.png)

And Consul register client will save MetaDataRegisterDTO to Key/Value store, storage struct is:

```sh
soul
   ├──regsiter
   ├    ├──metadata
   ├    ├     ├──${rpcType}
   ├    ├     ├      ├────${contextPath}
   ├    ├     ├               ├──${ruleName} : save metadata data of MetaDataRegisterDTO
```

Consul register client will save data to consul when soul client is started. Consul register server will keep watching the change of data node. Trigger selector and rule data update and event will be published, when metadata data node update. Trigger selector and upstream update and event will be published, when uri data node update.

#### Nacos Register

Nacos register have two parts：`URI` and `Metadata`
`URI` is instance register. URI instance node will be deleted when server is down.

`URI` service’s instance name will be named like below. Every `URI` instance has `ip`, `port` and `contextPath` as `identifiers`.

```sh
soul.register.service.${rpcType}
```

When `URI` instance up, it will publish metadata config. It’s name like below.

```sh
soul.register.service.${rpcType}.${contextPath}
```

Trigger selector and upstream update and event will be published, when URI service up or down.
Trigger selector and rule data update and event will be published, when metadata config update.


#### SPI

| *SPI Name*                            | *Description*                         |
| ------------------------------------- | ------------------------------------- |
| SoulClientRegisterRepository			|	Soul client register SPI 			|


| *Implementation Class*			| *Description*							 |
| --------------------------------- | -------------------------------------- |
| HttpClientRegisterRepository  	| 	Http client register repository 	 |
| ZookeeperClientRegisterRepository | 	Zookeeper client register repository |
| EtcdClientRegisterRepository  	| 	Etcd client register repository 	 |
| ConsulClientRegisterRepository 	| 	Consul client register repository 	 |
| NacosClientRegisterRepository 	| 	Nacos client register repository 	 |

| *SPI Name*                            | *Description*                         |
| ------------------------------------- | ------------------------------------- |
| SoulServerRegisterRepository			|	Soul server register SPI 			|

| *Implementation Class*			| *Description*							 |
| --------------------------------- | -------------------------------------- |
| SoulHttpRegistryController  		| 	Http server register repository 	 |
| ZookeeperServerRegisterRepository | 	Zookeeper server register repository |
| EtcdServerRegisterRepository  	| 	Etcd server register repository 	 |
| ConsulServerRegisterRepository 	| 	Consul server register repository 	 |
| NacosServerRegisterRepository 	| 	Nacos server register repository 	 |


### Register Center Access

#### Explain

Explain register center access config

#### HTTP Registry

**Soul-Admin**

Set the config in application.yml

```yml
soul:
  register:
    registerType: http
    props:
      checked: true  # is checked
      zombieCheckTimes: 5 # How many times does it fail to detect the service
      scheduledTime: 10 # Timed detection interval time
```

**Soul-Client**

Set the config in application.yml

```yml
soul:
  client:
    registerType: http
    serverLists: http://localhost:9095
    props:
      contextPath: /http
      appName: http
      port: 8188
      isFull: false
# registerType : register type, set http
# serverList: when register type is http，set Soul-Admin address list，pls note 'http://' is necessary.
# port: your project port number; apply to springmvc/tars/grpc
# contextPath: your project's route prefix through soul gateway, such as /order ，/product etc，gateway will route based on it.
# appName：your project name,the default value is`spring.application.name`.
# isFull: set true means providing proxy for your entire service, or only a few controller. apply to springmvc/springcloud
```

#### Zookeeper Registry

**Soul-Admin**

Add dependency in pom.xml (Default has been added):

```xml
<dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-register-client-zookeeper</artifactId>
            <version>${project.version}</version>
</dependency>
```

Set the config in application.yml

```yml
soul:
  client:
    registerType: zookeeper
    serverLists: localhost:2181
    props:
      contextPath: /http
      appName: http
      port: 8188  
      isFull: false
# registerType : register type, set zookeeper
# serverList: when register type is zookeeper，set zookeeper address list
# port: your project port number; apply to springmvc/tars/grpc
# contextPath: your project's route prefix through soul gateway, such as /order ，/product etc，gateway will route based on it.
# appName：your project name,the default value is`spring.application.name`.
# isFull: set true means providing proxy for your entire service, or only a few controller. apply to springmvc/springcloud
```

#### Etcd Registry

**Soul-Admin**

Add dependency in pom.xml (Default has been added):

```xml
<dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-register-server-etcd</artifactId>
            <version>${project.version}</version>
</dependency>
```

Set the config in application.yml

```yml
soul:
  register:
    registerType: etcd
    serverLists : http://localhost:2379
    props:
      etcdTimeout: 5000
      etcdTTL: 5
```

**Soul-Client**

Add dependency in pom.xml (Default has been added):

```xml
<dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-register-client-etcd</artifactId>
            <version>${project.version}</version>
</dependency>
```

Set the config in application.yml

```yml
soul:
  client:
    registerType: etcd
    serverLists: http://localhost:2379
    props:
      contextPath: /http
      appName: http
      port: 8188  
      isFull: false
# registerType : register type, set etcd 
# serverList: when register type is etcd, add etcd address list
# port: your project port number; apply to springmvc/tars/grpc
# contextPath: your project's route prefix through soul gateway, such as /order ，/product etc，gateway will route based on it.
# appName：your project name,the default value is`spring.application.name`.
# isFull: set true means providing proxy for your entire service, or only a few controller. apply to springmvc/springcloud
```

#### Consul Registry

**Soul-Admin**

Add dependency in pom.xml :

```xml
<!--soul-register-server-consul (Default has been added)-->
<dependency>
    <groupId>org.dromara</groupId>
    <artifactId>soul-register-server-consul</artifactId>
    <version>${project.version}</version>
</dependency>

<!--spring-cloud-starter-consul-discovery need add by yourself, suggest use 2.2.6.RELEASE version, other version maybe can't work-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
```

Set the config in application.yml, additional need add spring.cloud.consul：

```yml
soul:
  register:
    registerType: consul
    props:
      delay: 1
      wait-time: 55

spring:
  cloud:
    consul:
      discovery:
        instance-id: soul-admin-1
        service-name: soul-admin
        tags-as-metadata: false
      host: localhost
      port: 8500

# registerType : register type, set consul.
# delay: The interval of each polling of monitoring metadata, in seconds, the default value is 1 second.
# wait-time: The waiting time for each polling of metadata monitoring, in seconds, the default value is 55 second.
# instance-id: Required, Consul needs to find specific services through instance-id.
# service-name: The name where the service is registered to consul. If not configured, the value of `spring.application.name` will be taken by default.
# host: Consul server host, the default value is localhost.
# port: Consul server port, the default value is 8500.
# tags-as-metadata: false, Required, This option must be set to false, otherwise the URI information will not be found, will cause to selector and upstream cache unable to update.
```

**Soul-Client**

**Note, consul registry is not compatible with current and SpringCloud will and Eureka / Nacos registry conflicts**

Add dependency in pom.xml (need add by yourself, suggest use 2.2.6.RELEASE version, other version maybe can’t work)：

```xml
<dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-consul-discovery</artifactId>
               <version>2.2.6.RELEASE</version>
</dependency>
```

Set the config in application.yml, additional need add spring.cloud.consul：

```yml
soul:
  client:
    registerType: consul 
    props:
      contextPath: /http
      appName: http
      port: 8188  
      isFull: false

spring:
  cloud:
    consul:
      discovery:
        instance-id: soul-http-1
        service-name: soul-http
      host: localhost
      port: 8500

# registerType : register type, set consul.
# port: your project port number; apply to springmvc/tars/grpc
# contextPath: your project's route prefix through soul gateway, such as /order ，/product etc，gateway will route based on it.
# appName：your project name,the default value is`spring.application.name`.
# isFull: set true means providing proxy for your entire service, or only a few controller. apply to springmvc
# instance-id: Required, Consul needs to find specific services through instance-id.
# service-name: The name where the service is registered to consul. If not configured, the value of `spring.application.name` will be taken by default.
# host: Consul server host, the default value is localhost.
# port: Consul server port, the default value is 8500.
```

#### Nacos Registry

**Soul-Admin**

Add dependency in pom.xml (Default has been added):

```xml
<dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-register-server-nacos</artifactId>
            <version>${project.version}</version>
</dependency>
```

Set the config in application.yml

```yml
soul:
  register:
    registerType: nacos
    serverLists : localhost:8848
    props:
      nacosNameSpace: SoulRegisterCenter
```

**Soul-Client**

Add dependency in pom.xml (Default has been added):

```xml
<dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-register-client-nacos</artifactId>
            <version>${project.version}</version>
</dependency>
```

Set the config in application.yml

```yml
soul:
  client:
    registerType: nacos
    serverLists: localhost:8848
    props:
      contextPath: /http
      appName: http
      port: 8188  
      isFull: false
      nacosNameSpace: SoulRegisterCenter
# registerType : register type, set etcd 
# serverList: when register type is etcd, add etcd address list
# port: your project port number; apply to springmvc/tars/grpc
# contextPath: your project's route prefix through soul gateway, such as /order ，/product etc，gateway will route based on it.
# appName：your project name,the default value is`spring.application.name`.
# isFull: set true means providing proxy for your entire service, or only a few controller. apply to springmvc/springcloud
# nacosNameSpace: nacos namespace

```

## User Guide

### Environment Setup

#### Features

 - Soul is an open source plugin framework, which is flexibility and extensibility since 2.2.0 version. With soul you can easily create application with your own gateway.
 - System Requirement: JDK 1.8+, Mysql 5.5.20 +.

#### Soul-Admin

##### remote download

 * 2.3.0 download `oul-admin-bin-2.3.0-RELEASE.tar.gz`
 * tar `soul-admin-bin-2.3.0-RELEASE.tar.gz`. cd `/bin`.
 * use `h2` store. 

```sh
> windows :  start.bat

> linux : ./start.sh --spring.profiles.active = h2
```

 * use `mysql` store. cd `/conf` and then modify `mysql` config in `application.yaml`.

```sh
> windwos : start.bat 

> linux : ./start.sh
```

##### docker

```sh
> docker network create soul
```

 * use `h2` store:

```sh
> docker run -d -p 9095:9095 --net soul soul-admin
```

 * use `mysql` store:

```sh
> docker run -e "SPRING_PROFILES_ACTIVE=mysql" -d -p 9095:9095 --net soul soul-admin
```

If you want to override environment variables, you can do like this.

```sh
docker run -e "SPRING_PROFILES_ACTIVE=mysql" -e "spring.datasource.url=jdbc:mysql://192.168.1.9:3306/soul?useUnicode=true&characterEncoding=utf-8&useSSL=false" -e "spring.datasource.password=123456" -d -p 9095:9095 --net soul soul-admin
```

Another way, bind volume and mounts

Put your `application.yml` in xxx directory, then run like this.

```sh
docker run -v D:\tmp\conf:/opt/soul-admin/conf/ -d -p 9095:9095 --net soul soul-admin
```

##### local

 * download

```sh
> cd soul
> mvn clean install -Dmaven.javadoc.skip=true -B -Drat.skip=true -Djacoco.skip=true -DskipITs -DskipTests
```

 * setup for `SoulAdminBootstrap`:
 	* if use `h2` store please set env `--spring.profiles.active = h2`
 	* if use `mysql` store, please modify `mysql` config in `application.yaml`.

Visit `http://localhost:9095/index.html` default username：`admin` password: `123456`.

#### Soul-Bootstrap

##### remote download

 * 2.3.0 download `soul-bootstrap-bin-2.3.0-RELEASE.tar.gz`
 * tar `soul-bootstrap-bin-2.3.0-RELEASE.tar.gz`, ant then cd `/bin`.

```sh
> windwos : start.bat 

> linux : ./start.sh 
```

##### docker

```sh
> docker network create soul
> docker run -d -p 9195:9195 --net soul soul-bootstrap
```

##### local

```sh
> cd soul
> mvn clean install -Dmaven.javadoc.skip=true -B -Drat.skip=true -Djacoco.skip=true -DskipITs -DskipTests
```

 * setup for `SoulBootstrap`.


#### Build your own gateway（recommend）

 * First of all, create a new Spring Boot project. You can refer to the way how you start the soul-bootstrap, or visit [Spring Initializer](https://spring.io/quickstart) 
 * Add these JAR into your Maven pom.xml:

```xml
  <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
        <version>2.2.2.RELEASE</version>
  </dependency>

  <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
        <version>2.2.2.RELEASE</version>
  </dependency>

  <!--soul gateway start-->
  <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>soul-spring-boot-starter-gateway</artifactId>
        <version>${last.version}</version>
  </dependency>
  
   <!--soul data sync start use websocket-->
   <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>soul-spring-boot-starter-sync-data-websocket</artifactId>
        <version>${last.version}</version>
   </dependency>
```

 * Add these config values into your application.yaml

```yml
spring:
  main:
    allow-bean-definition-overriding: true

management:
  health:
    defaults:
      enabled: false
soul :
  sync:
    websocket :
      urls: ws://localhost:9095/websocket
```

 * Environment Setup has finished, now your project is launched.

### Use Different Data-Sync Strategy

#### Features

 - Data synchronization is the key of gateway high performance, which is to sync `soul-admin` config data into the JVM memory of soul cluster.
 - Implementation principles, pls refer: [DataSync](#data-synchronization-design)
 - In the article, the gateway is the environment you setup. please refer to：[Enviroment Setup](##environment-setup)

#### Websocket sync（default method，recommend）

Gateway setting（note:restart）

Add these dependencies in `pom.xml`：

```xml
<!--soul data sync start use websocket-->
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-sync-data-websocket</artifactId>
  <version>${last.version}</version>
</dependency>
```

add these config values in springboot yaml file:

```yml
soul :
  sync:
      websocket :
           urls: ws://localhost:9095/websocket
#urls: address of soul-admin，multi-address will be splitted with (,).
```

soul-admin config, enable this parameter `--soul.sync.websocket=''` in soul admin, then restart service.

```yml
soul:
  sync:
     websocket:

```

When the connection is established, getting the full data once,then adding and upating data subsequently, which is a good performance.

Support disconnection and reconnection (default 30 sec).

#### Zookeeper Sync

Gateway setting（note: restart

Add these dependencies in `pom.xml`:

```xml
<!--soul data sync start use zookeeper-->
  <dependency>
       <groupId>org.dromara</groupId>
        <artifactId>soul-spring-boot-starter-sync-data-zookeeper</artifactId>
        <version>${last.version}</version>
  </dependency>
```

Add these dependencies in springboot yaml file:

```yml
soul :
  sync:
      zookeeper:
           url: localhost:2181
           sessionTimeout: 5000
           connectionTimeout: 2000
#url: config with your zk address, used by the cluster environment, splitted with (,).

```

soul-admin config: configure the soul-admin’s starting parameter with `--soul.sync.zookeeper.url='your address'`,then restart the service.


```yml
soul:
  sync:
    zookeeper:
        url: localhost:2181
        sessionTimeout: 5000
        connectionTimeout: 2000
```

It is good to use ZooKeeper synchronization mechanism with high timeliness, but we also have to deal with the unstable environment of ZK, cluster brain splitting and other problems.

#### Http long-polling sync

Gateway setting（note:restart）

Add these dependencies in `pom.xml`：

```xml
<!--soul data sync start use http-->
  <dependency>
       <groupId>org.dromara</groupId>
        <artifactId>soul-spring-boot-starter-sync-data-http</artifactId>
        <version>${last.version}</version>
  </dependency>
```

add these config values in your springboot yaml file:

```yml
  soul :
      sync:
          http:
               url: http://localhost:9095
  #url: config with your soul-admin's ip and port url, pls use (,) to split multi-admin cluster environment.
```

soul-admin config, configure the soul-admin’s starting parameter with `--soul.sync.http=''`, then restart service.

```yml
soul:
  sync:
     http:
```

HTTP long-polling makes the gateway lightweight, but less time-sensitive.

It pulls according to the group key, if the data is too large, it will have some influences, a small change under a group will pull the entire group.

It may hit bug in `soul-admin` cluster.

#### Nacos sync

Gateway setting（note:restart）

Add these dependencies in your `pom.xml`：

```xml
<!--soul data sync start use nacos-->
  <dependency>
       <groupId>org.dromara</groupId>
        <artifactId>soul-spring-boot-starter-sync-data-nacos</artifactId>
        <version>${last.version}</version>
  </dependency>

```

add these config values in the springboot yaml file:

```yml
  soul :
      sync:
         nacos:
              url: localhost:8848
              namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
              acm:
                enabled: false
                endpoint: acm.aliyun.com
                namespace:
                accessKey:
                secretKey:
  # url: config with your nacos address, pls use (,) to split your cluster environment.
  # other configure，pls refer to the naocs website.
```

soul-admin config: passing values one by one with ‘–’ operator in the soul-Admin startup parameter.

```yml
soul :
      sync:
         nacos:
              url: localhost:8848
              namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
              acm:
                enabled: false
                endpoint: acm.aliyun.com
                namespace:
                accessKey:
                secretKey:
```

## Developer Guide

### Filter Extension

#### Description

This doc shows a demo for how to extend `org.springframework.web.server.WebFliter`.

#### CORS Support

`org.dromara.soul.bootstrap.cors.CrossFilter` is designed for WebFilter implementation.

```java
 public class CrossFilter implements WebFilter {
 
     private static final String ALLOWED_HEADERS = "x-requested-with, authorization, Content-Type, Authorization, credential, X-XSRF-TOKEN,token,username,client";
 
     private static final String ALLOWED_METHODS = "*";
 
     private static final String ALLOWED_ORIGIN = "*";
 
     private static final String ALLOWED_EXPOSE = "*";
 
     private static final String MAX_AGE = "18000";
 
     @Override
     @SuppressWarnings("all")
     public Mono<Void> filter(final ServerWebExchange exchange, final WebFilterChain chain) {
         ServerHttpRequest request = exchange.getRequest();
         if (CorsUtils.isCorsRequest(request)) {
             ServerHttpResponse response = exchange.getResponse();
             HttpHeaders headers = response.getHeaders();
             headers.add("Access-Control-Allow-Origin", ALLOWED_ORIGIN);
             headers.add("Access-Control-Allow-Methods", ALLOWED_METHODS);
             headers.add("Access-Control-Max-Age", MAX_AGE);
             headers.add("Access-Control-Allow-Headers", ALLOWED_HEADERS);
             headers.add("Access-Control-Expose-Headers", ALLOWED_EXPOSE);
             headers.add("Access-Control-Allow-Credentials", "true");
             if (request.getMethod() == HttpMethod.OPTIONS) {
                 response.setStatusCode(HttpStatus.OK);
                 return Mono.empty();
             }
         }
         return chain.filter(exchange);
     }
 }

```

Registering CrossFilter as a Spring Bean and you are ready to go.

#### Filtering Spring Boot health check

You can control the order by applying `@Order` to the implementation class.

```java
@Component
@Order(-99)
public final class HealthFilter implements WebFilter {

    private static final String[] FILTER_TAG = {"/actuator/health", "/health_check"};

    @Override
    public Mono<Void> filter(@Nullable final ServerWebExchange exchange, @Nullable final WebFilterChain chain) {
        ServerHttpRequest request = Objects.requireNonNull(exchange).getRequest();
        String urlPath = request.getURI().getPath();
        for (String check : FILTER_TAG) {
            if (check.equals(urlPath)) {
                String result = JsonUtils.toJson(new Health.Builder().up().build());
                DataBuffer dataBuffer = exchange.getResponse().bufferFactory().wrap(result.getBytes());
                return exchange.getResponse().writeWith(Mono.just(dataBuffer));
            }
        }
        return Objects.requireNonNull(chain).filter(exchange);
    }
}

```

#### Extending

Add a new class and inherit from `org.dromara.soul.web.filter.AbstractWebFilter`.

Implement abstract methods of parent class.

```java
   /**
     * this is Template Method ,children Implement your own filtering logic.
     *
     * @param exchange the current server exchange
     * @param chain    provides a way to delegate to the next filter
     * @return {@code Mono<Boolean>} result：TRUE (is pass)，and flow next filter；FALSE (is not pass) execute doDenyResponse(ServerWebExchange exchange)
     */
    protected abstract Mono<Boolean> doFilter(ServerWebExchange exchange, WebFilterChain chain);

    /**
     * this is Template Method ,children Implement your own And response client.
     *
     * @param exchange the current server exchange.
     * @return {@code Mono<Void>} response msg.
     */
    protected abstract Mono<Void> doDenyResponse(ServerWebExchange exchange);

```

if method `doFilter` returns Mono, this filter is passing, but not vice versa. While rejecting, it will call method `doDenyResponse` and sending infos in response body to frontend.

### File Uploading And Downloading

#### Description

This doc gives a brief description for uploading and downloading files using soul.

#### File Uploading

 - File Uploading
 - For custom limitation, use`--file.size` with an integer variable. e.g.`--file.size = 30`
 - Upload your files just as way you did before

#### File Downloading

Soul supports downloading files in streams. There is no need to change anything.

### Fetching Correct IP Address And Host

#### Description

 - This doc demonstrates how to get correct IP address and host when soul serves behind nginx reverse proxy.
 - After fetched real IP and host, you can match them with plugins and selectors.

#### Default Implementation

 - The embedded implementation in soul is: `org.dromara.soul.web.forwarde.ForwardedRemoteAddressResolver`
 - You need to config `X-Forwarded-For` in `nginx` first to get correct IP address and host.

#### Implement through a Plugin

Declare a new class named “A” and implements `org.dromara.soul.plugin.api.RemoteAddressResolver`.

```java
public interface RemoteAddressResolver {

    /**
     * Resolve inet socket address.
     *
     * @param exchange the exchange
     * @return the inet socket address
     */
    default InetSocketAddress resolve(ServerWebExchange exchange) {
        return exchange.getRequest().getRemoteAddress();
    }

}
```

Register defined class as a Spring Bean.

```java
   @Bean
   public SignService a() {
         return new A
   }
```

### Custom Sign Algorithm

#### Description

Users can customize the signature authentication algorithm to achieve verification.

#### Extension

The default implementation is `org.dromara.soul.plugin.sign.service.DefaultSignService`.

Declare a new class named “A” and implements `org.dromara.soul.plugin.api.SignService`.

```java
 public interface SignService {
 
     /**
      * Sign verify pair.
      *
      * @param exchange   the exchange
      * @return the pair
      */
     Pair<Boolean, String> signVerify(ServerWebExchange exchange);
 }

```

When returning true in Pair, the sign verification passes. If there’s false, the String in Pair will be return to the frontend to show.

Register defined class as a Spring Bean.

```java
   @Bean
   public SignService a() {
         return new A
   }
```

### A multilingual HTTP client

#### Description

 - This document focuses on how to access gateways for HTTP services in other languages.
 - How to customize the development of soul-http-client.

#### Customize Http Client

 * Request Method: `POST`
 * Request Path: `http://soul-admin/soul-client/springmvc-register`, soul-admin represents `IP + Port` of admin
 * Request Params：passing `JSON` type parameters through the body.

```json
{
    "appName": "xxx", //required
    "context": "/xxx", //required
    "path": "xxx", //required
    "pathDesc": "xxx", 
    "rpcType": "http", //required
    "host": "xxx", //required
    "port": xxx, //required
    "ruleName": "xxx", //required
    "enabled": "true", //required
    "registerMetaData": "true" //required
}
```

### Thread Model

#### Description

This article gives an introduction to thread models in soul and usage in various scenarios.

#### IO And Work Thread

`spring-webflux` is one of dependencies of soul, and it uses Netty thread model in lower layer.

#### Business Thread

 - Use scheduling thread to execute by default.
 - A fixed thread pool manages business threads, the number of threads is count in this formula: `cpu * 2 + 1`.

#### Type Switching

 * `reactor.core.scheduler.Schedulers`.
 * `-Dsoul.scheduler.type=fixed` is a default config. If set to other value, a flexible thread pool will take place it.`Schedulers.elastic()`
 * `-Dsoul.work.threads = xx` is for configuring number of threads, the default value calculates in following formula `cpu * 2 + 1` with a minimum of 16 threads.

### Soul Optimize

#### Description

This doc shows how to do performance optimization for soul.

#### Time Consumption

Soul is JVM driven and processing time for a single request is nearly between `1-3` ms.

#### Netty Optimization

 - `spring-webflux` is one of dependencies of soul, and it uses Netty in lower layer.
 - The demo down below demonstrates tuning soul by customizing params in Netty.

```java
@Bean
 public NettyReactiveWebServerFactory nettyReactiveWebServerFactory() {
     NettyReactiveWebServerFactory webServerFactory = new NettyReactiveWebServerFactory();
     webServerFactory.addServerCustomizers(new EventLoopNettyCustomizer());
     return webServerFactory;
 }

private static class EventLoopNettyCustomizer implements NettyServerCustomizer {

    @Override
    public HttpServer apply(final HttpServer httpServer) {
        return httpServer
                .tcpConfiguration(tcpServer -> tcpServer
                        .runOn(LoopResources.create("soul-netty", 1, DEFAULT_IO_WORKER_COUNT, true), false)
                        .selectorOption(ChannelOption.SO_REUSEADDR, true)
                        .selectorOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                        .option(ChannelOption.TCP_NODELAY, true)
                        .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT));
    }
}
```

 - Soul-bootstrap offers this class, you may modify it when benchmarking your app if necessary.
 - You can get references of business thread model from [thread model](#thread-model)


# SOUL