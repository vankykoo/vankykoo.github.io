---
title: SPI + 工厂设计模式
date: 2024-10-09
draft: "false"
tags:
  - SPI
  - 设计模式
---


## 1. 什么是 SPI ？有什么用？

SPI（Service Provider Interface）是Java提供的一种用于实现可扩展性和插件机制的接口规范。它允许开发者定义或使用服务提供者（Service Providers），并且可以**在运行时动态地加载这些服务**，而不需要在编译时将服务的实现绑定到应用程序中。这种机制特别适合那些需要根据配置或条件加载不同服务实现的场景，例如数据库驱动、加密算法等。

### 1.1. SPI 的核心概念：

1. **服务接口（Service Interface）**：定义服务提供者需要实现的接口。例如，`java.sql.Driver`就是一个常见的服务接口，所有的数据库驱动都必须实现这个接口。
2. **服务提供者（Service Provider）**：服务接口的具体实现。例如，不同数据库厂商的驱动程序就是服务提供者。
3. **服务加载器（Service Loader）**：用于发现和加载服务提供者的机制。Java 提供了 `java.util.ServiceLoader` 来查找服务提供者。
4. **配置文件**：SPI 使用一个特殊的配置文件来声明服务提供者的实现类。配置文件放置在 `META-INF/services/` 目录下，文件名为服务接口的全限定名，文件内容为实现类的全限定名。例如，`META-INF/services/java.sql.Driver` 这个文件可能包含多个 JDBC 驱动实现类。

### 1.2. SPI的工作原理：

- 应用程序通过 `ServiceLoader` 查找指定服务接口的实现。
- JVM在 `META-INF/services/` 目录下查找对应的服务配置文件。
- 服务加载器解析配置文件并实例化指定的服务提供者类。

## 2. 实现

具体实现方法如下：

1. 指定 SPI 的配置目录，并且将配置再分为系统内置 SPI 和用户自定义 SPI，便于区分优先级和维护。
2. 编写 SpiLoader 加载器，实现读取配置、加载实现类的方法。
   1. 用 Map 来存储已加载的配置信息 键名 => 实现类。
   2. 通过 Hutool 工具库提供的 ResourceUtil.getResources 扫描指定路径，读取每个配置文件，获取到 键名 => 实现类 信息并存储在 Map 中。
   3. 定义获取实例方法，根据用户传入的接口和键名，从 Map 中找到对应的实现类，然后通过反射获取到实现类对象。可以维护一个对象实例缓存，创建过一次的对象从缓存中读取即可。
3. 重构序列化器工厂，改为从 SPI 加载指定的序列化器对象。

使用静态代码块调用 SPI 的加载方法，在工厂首次加载时，就会调用 SpiLoader 的 load 方法加载序列化器接口的所有实现类，之后就可以通过调用 getInstance 方法获取指定的实现类对象了。



以负载均衡器为例：

### 1. 配置文件 

![img](https://raw.githubusercontent.com/vankykoo/image/refs/heads/main/cut/014.png)

com.vanky.myrpc.loadbalancer.LoadBalancer：

```properties
roundRobin=com.vanky.myrpc.loadbalancer.RoundRobinLoadBalancer
random=com.vanky.myrpc.loadbalancer.RandomLoadBalancer
consistentHash=com.vanky.myrpc.loadbalancer.ConsistentHashLoadBalancer
```

### 2.SpiLoader

属性：

1. 1. loaderMap：存储某个接口有哪些实现类。
   2. instanceCache：存储实现类的实例，防止多次创建。

核心方法：

1. 1. load(Class<?> loadClass)：用于加载某个接口有哪些实现类，放入到 loaderMap 中。
   2. getInstance(Class<?> tClass, String key)：根据接口类型与实现类对应的键名，获取指定实现类实例。

```java
/**
 * Spi 加载器（支持键值对映射）
 * @author vanky
 * @create 2024/9/27 21:41
 */
@Slf4j
public class SpiLoader {

    /**
     *  存储已加载的类：接口名 => （key => 实现类）
     */
    private static Map<String, Map<String, Class<?>>> loaderMap = new ConcurrentHashMap<>();

    /**
     * 对象实例缓存（避免重复 new），类路径 => 对象实例，单例模式
     */
    private static Map<String, Object> instanceCache = new ConcurrentHashMap<>();

    /**
     * 系统 SPI 目录
     */
    private static final String RPC_SYSTEM_SPI_DIR = "META-INF/rpc/system/";

    /**
     * 用户自定义 SPI 目录
     */
    private static final String RPC_CUSTOM_SPI_DIR = "META-INF/rpc/custom/";

    /**
     * 扫描路径
     */
    private static final String[] SCAN_DIRS = new String[]{RPC_SYSTEM_SPI_DIR, RPC_CUSTOM_SPI_DIR};

    /**
     * 动态加载的类列表
     */
    private static final List<Class<?>> LOAD_CLASS_LIST = Arrays.asList(Serializer.class);

    /**
     * 加载所有类型
     */
    public static void loadAll(){
        log.info("加载所有 SPI");
        for (Class<?> aClass : LOAD_CLASS_LIST) {
            load(aClass);
        }
    }

    /**
     * 获取某个接口的实例
     * @param tClass
     * @param key
     * @return
     * @param <T>
     */
    public static <T> T getInstance(Class<?> tClass, String key) {
        String tClassName = tClass.getName();
        Map<String, Class<?>> keyClassMap = loaderMap.get(tClassName);
        if (keyClassMap == null){
            throw new RuntimeException(String.format("SpiLoader 未加载 %s 类型", tClassName));
        }
        if (!keyClassMap.containsKey(key)){
            throw new RuntimeException(String.format("SpiLoader 的 %s 不存在 key=%s 类型", tClassName, key));
        }
        // 获取到要加载的实现类型
        Class<?> implClass = keyClassMap.get(key);
        // 从实例缓存中加载指定类型的实例
        String implClassName = implClass.getName();
        if (!instanceCache.containsKey(implClassName)){
            try {
                instanceCache.put(implClassName, implClass.getDeclaredConstructor().newInstance());
            }catch (NoSuchMethodException | InvocationTargetException | InstantiationException | IllegalAccessException e){
                String errorMsg = String.format(" %s 类实例化失败", implClassName);
                throw new RuntimeException(errorMsg, e);
            }
        }

        return (T) instanceCache.get(implClassName);
    }

    /**
     * 加载某个类型
     * @param loadClass
     * @return
     */
    public static Map<String, Class<?>> load(Class<?> loadClass){
        log.info("加载类型为 {} 的 SPI", loadClass.getName());

        // 扫描路径，用户自定义的 SPI 优先级高于系统 SPI
        Map<String, Class<?>> keyClassMap = new HashMap<>();
        for (String scanDir : SCAN_DIRS) {
            // 获取 resource 文件夹下需要扫描的文件地址 + 文件名
            List<URL> resources = ResourceUtil.getResources(scanDir + loadClass.getName());
            for (URL resource : resources) {
                try {
                    InputStreamReader inputStreamReader = new InputStreamReader(resource.openStream());
                    BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
                    String line;
                    while ((line = bufferedReader.readLine()) != null){
                        String[] strArray = line.split("=");
                        if (strArray.length > 1){
                            String key = strArray[0];
                            String className = strArray[1];
                            keyClassMap.put(key, Class.forName(className));
                        }
                    }
                }catch (Exception e){
                    log.error("spi resource load error", e);
                }
            }
        }

        loaderMap.put(loadClass.getName(), keyClassMap);
        return keyClassMap;
    }
}

```

### 3. 配合工厂模式，获取实例

![img](https://raw.githubusercontent.com/vankykoo/image/refs/heads/main/cut/015.png)

```java
/**
 * 负载均衡器工厂
 * @author vanky
 * @create 2024/10/9 21:15
 */
public class LoadBalancerFactory {

    static {
        SpiLoader.load(LoadBalancer.class);
    }

    /**
     * 默认负载均衡器
     */
    public static final LoadBalancer DEFAULT_LOAD_BALANCER = new RoundRobinLoadBalancer();

    /**
     * 获取实例
     * @param key
     * @return
     */
    public static LoadBalancer getInstance(String key){
        return SpiLoader.getInstance(LoadBalancer.class, key);
    }

}
```

### 4.应用

```java
// 负载均衡
LoadBalancer loadBalancer = LoadBalancerFactory.getInstance(rpcConfig.getLoadBalancer());
```