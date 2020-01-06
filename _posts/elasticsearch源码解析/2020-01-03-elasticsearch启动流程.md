---
layout:     post
title:      "elasticsearch启动流程"
date:       2020-01-03 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 源码
    - 搜索引擎
---
#### Elasticsearch启动涉及到的入口类
>org.elasticsearch.bootstrap.Elasticsearch->
>org.elasticsearch.bootstrap.Bootstrap->
>org.elasticsearch.node.Node

#### Elasticsearch类
main函数在Elasticsearch里面,最终调用<strong>public final int main(String[] args, Terminal terminal) throws Exception</strong><br/>
来正式进入启动逻辑代码如下
```java
public final int main(String[] args, Terminal terminal) throws Exception {
        if (addShutdownHook()) {

            shutdownHookThread = new Thread(() -> {
                try {
                    this.close();<1>
                } catch (final IOException e) {
                }
            });
            Runtime.getRuntime().addShutdownHook(shutdownHookThread);
        }
        beforeMain.run();<2>
        try {
            mainWithoutErrorHandling(args, terminal);<3>
        } catch (OptionException e) {
        } catch (UserException e) {
        }
        return ExitCodes.OK;
    }
```
1. 注册钩子事件,用来在进程关闭之前释放资源
2. 启动之前的准备工作,目前里面没有做任何处理
3. 启动elasticsearch节点逻辑

然后Elasticsearch里面调用流程如下,主逻辑在<code>init()</code>里面,如下<br/>
<code>mainWithoutErrorHandling()->execute()->init()</code>
```java
try {
    Bootstrap.init(!daemonize, pidFile, quiet, initialEnv);
} catch (BootstrapException | RuntimeException e) {
    // format exceptions to the console in a special way
    // to avoid 2MB stacktraces from guice, etc.
    throw new StartupException(e);
}
```
直接调用Bootstrap.init()进行启动,至此Elasticsearch类的任务已完成,进入Bootstrap阶段

#### Bootstrap类

```java
static void init(
        final boolean foreground,
        final Path pidFile,
        final boolean quiet,
        final Environment initialEnv) throws BootstrapException, NodeValidationException, UserException {
    BootstrapInfo.init();
    INSTANCE = new Bootstrap();<1>
    final SecureSettings keystore = loadSecureSettings(initialEnv);
    final Environment environment = createEnvironment(foreground, pidFile, keystore, initialEnv.settings(), initialEnv.configFile());
    final boolean closeStandardStreams = (foreground == false) || quiet;
    try {
        INSTANCE.setup(true, environment);<2>

        INSTANCE.start();<3>

    } catch (NodeValidationException | RuntimeException e) {
       
    }
}

private void setup(boolean addShutdownHook, Environment environment) throws BootstrapException {
    
    //do something
    node = new Node(environment) {<4>
        @Override
        protected void validateNodeBeforeAcceptingRequests(
            final BootstrapContext context,
            final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
            BootstrapChecks.check(context, boundTransportAddress, checks);
        }

        @Override
        protected void registerDerivedNodeNameWithLogger(String nodeName) {
            LogConfigurator.setNodeName(nodeName);
        }
    };
}

private void start() throws NodeValidationException {
    node.start();<5>
    keepAliveThread.start();
}
```
1. 实例化Bootstrap
2. 对Bootstrap进行初始化,例如:创建Node实例
3. 启动Elasticsearch服务
4. 创建Node实例
5. 启动Node,至此进入org.elasticsearch.node.Node阶段

#### org.elasticsearch.node.Node类
>之前的处理都是准备工作,在Node阶段才真正意义进入服务的启动逻辑阶段
>主要的启动逻辑都在方法org.elasticsearch.node.Node#start里面
##### Node#start
```java
pluginLifecycleComponents.forEach(LifecycleComponent::start);
injector.getInstance(XXX.class).start();
```
里面最核心的就是把所有的组件启动起来,而组件的构建工作在Node的构造函数里面


