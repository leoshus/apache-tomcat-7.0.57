====================
### research the source about apache-tomcat-7.0.57
====================
* /org/apache/catalina/startup/Bootstrap---2014/11/28
* /org/apache/catalina/startup/Catalina---2014/11/28
* 当Bootstrap启动入口传入"startd"指令时,daemon.load(args);daemon.start();ps:daemon为Bootstrap实例对象
* daemon.load(args);引领一个初始化流程：通过反射调用 Catalina.load()方法--> getServer().init() -引 发->LifecycleBase.init()->StandardServer.initInternal()/StandardService().initInternal()->StandardEngine.initInternal()
* daemon.start();触发了start事件。通过反射调用Catalina.start()->StandardServer继承的Lifecycle接口的默认实现类LifecycleBase.start()这时的LifecycleState为INITIALIZED(daemon.load(args)完成的初始化过程)，并注册了一个关闭钩子，为tomcat异常关闭，清除内存做准备。->StandardServer.initInternal()->

```
fireLifecycleEvent(CONFIGURE_START_EVENT, null);
        setState(LifecycleState.STARTING);
        globalNamingResources.start();//启动GlobalNamingResources,可能有JNDI配置 或者是用户自定义database协助Realm验证user
        // Start our defined Services
        synchronized (services) {
            for (int i = 0; i < services.length; i++) {
                services[i].start();//启动server关联的service容器
            }
        }
```

->
```
protected void startInternal() throws LifecycleException {
        if(log.isInfoEnabled())
            log.info(sm.getString("standardService.start.name", this.name));
        setState(LifecycleState.STARTING);
        // Start our defined Container first
        if (container != null) {
            synchronized (container) {
                container.start();//启动service组件下的engine 默认情况
            }
        }
        synchronized (executors) {
            for (Executor executor: executors) {
                executor.start();
            }
        }
        // Start our defined Connectors second
        synchronized (connectors) {
            for (Connector connector: connectors) {
                try {
                    // If it has already failed, don't try and start it
                    if (connector.getState() != LifecycleState.FAILED) {
                        connector.start();//启动service组件下关联的链接器 Connector.java->protocolHandler.start();映射监听器mapperListener.start();
                    }
                } catch (Exception e) {
                    log.error(sm.getString(
                            "standardService.connector.startFailed",
                            connector), e);
                }
            }
        }
    }
```

engine容器会首先调用自己继承的父类containerBase中的startInternal()方法
```
protected synchronized void startInternal() throws LifecycleException {
        // Start our subordinate components, if any
        if ((loader != null) && (loader instanceof Lifecycle))
            ((Lifecycle) loader).start();//启动加载器
        logger = null;
        getLogger();
        if ((manager != null) && (manager instanceof Lifecycle))
            ((Lifecycle) manager).start();//启动管理器 session管理
        if ((cluster != null) && (cluster instanceof Lifecycle))
            ((Lifecycle) cluster).start();//集群启动
        Realm realm = getRealmInternal();
        if ((realm != null) && (realm instanceof Lifecycle))
            ((Lifecycle) realm).start();//领域启动
        if ((resources != null) && (resources instanceof Lifecycle))
            ((Lifecycle) resources).start();
        // Start our child containers, if any
        Container children[] = findChildren();//查找当前容器的子容器 engine->host->context-wrapper
        List<Future<Void>> results = new ArrayList<Future<Void>>();
        for (int i = 0; i < children.length; i++) {
            results.add(startStopExecutor.submit(new StartChild(children[i])));
        }
        boolean fail = false;
        for (Future<Void> result : results) {
            try {
                result.get();
            } catch (Exception e) {
                log.error(sm.getString("containerBase.threadedStartFailed"), e);
                fail = true;
            }
        }
        if (fail) {
            throw new LifecycleException(
                    sm.getString("containerBase.threadedStartFailed"));
        }
        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle)
            ((Lifecycle) pipeline).start();
        setState(LifecycleState.STARTING);
        // Start our thread
        threadStart();
    }
```
*  接下来的启动流程依次为 engine->host->context-wrapper，由于StandardEngine/StandardHost/StandardContext/StandardWrapper均继承了ContainerBase 故启动流程类似于Engine容器。

