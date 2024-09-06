# Dubbo AOP 和 IOC 机制详解

## 1.IOC 解析

Dubbo IoC 的过程实现在 **`com.alibaba.dubbo.common.extension.ExtensionLoader#injectExtension`** 方法中，该方法使用在下面的几处地方：

```java{.line-numbers}
createAdaptiveExtension() --> injectExtension((T) getAdaptiveExtensionClass().newInstance()) // 对已实例化的自适应扩展类(适配类)实例进行属性注入
createExtension(String name)
    // 对已实例化的扩展点实现类实例进行属性注入
    --> injectExtension(instance)
    // 对已实例化的包装类进行属性注入
    --> injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance))
```

下面分析一下 injectExtension 方法的源代码：

```java{.line-numbers}
// ExtensionLoader#injectExtension
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            //遍历目标类的所有方法
            for (Method method : instance.getClass().getMethods()) {
                //检测方法是否以 set 开头，且方法仅有一个参数，且方法访问级别为 public
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        //获取属性名，比如 setName 方法对应属性名 name
                        String property = method.getName().length() > 3 ?
                            method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        //从 ObjectFactory 中获取依赖对象
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            //通过反射调用 setter 方法设置依赖
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

Dubbo IoC 是通过 setter 方法注入依赖。Dubbo 首先会通过反射获取到实例的所有方法，然后再遍历方法列表，检测方法名是否具有 setter 方法特征。若有，则通过 ObjectFactory 获取依赖对象，最后通过反射调用 setter 方法将依赖设置到目标对象中。上述的 injectExtension() 方法，完成了对 instance 对象的属性的注入，通过 setter 方法进行反射调用实现。主要逻辑如下：

1. 得到 setter 方法，获取 setter 方法参数的类型和名称
2. 使用 objectFactory 获取对应参数名和参数类型的参数实例
3. setter 方法，反射注入

objectFactory 变量的类型为 AdaptiveExtensionFactory，AdaptiveExtensionFactory 内部维护了一个 ExtensionFactory 列表，用于存储其他类型的 ExtensionFactor，也就是说 factories = [SpringExtensionFactory intance, SpiExtensionFactory instance]。Dubbo 目前提供了两种 ExtensionFactory，分别是 SpiExtensionFactory 和 SpringExtensionFactory。前者用于创建自适应的拓展，后者是用于从 Spring 的 IOC 容器中获取所需的拓展。

```java{.line-numbers}
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

ExtensionLoader 中 objectFactory 是在构造函数中初始化的，AdaptiveExtensionLoader 类有 @Adaptive 注解，如果扩展类上面有 @Adaptive 注解，会使用该类作为自适应类，否则就会使用 dubbo 自己动态生成的自适应类，比如 Protocol$Adaptive。在 ExtensionLoader 方法的构造函数中，**`ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()`** 会返回一个 AdaptiveExtensionFactory 对象，作为自适应扩展实例。

### 1.1 AdaptiveExtensionFactory 类

```java{.line-numbers}
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    /**
     * AdaptiveExtensionLoader会遍历所有的ExtensionFactory实现，尝试着去加载扩展。如果找到了，返回。如果没有，在下一个ExtensionFactory中继续找。
     * Dubbo内置了两个ExtensionFactory，分别从Dubbo自身的扩展机制（SpiExtensionFactory）和Spring容器中去寻找（SpringExtensionFactory）。
     * 由于ExtensionFactory本身也是一个扩展点，我们可以实现自己的ExtensionFactory，让Dubbo的自动装配支持我们自定义的组件。
     */
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
            return null;
        }
    }
}
```

### 1.2 SpiExtensionFactory 类

```java{.line-numbers}
public class SpiExtensionFactory implements ExtensionFactory {

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        // Class type对应的必须是接口，并且需要注解了@SPI注解
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            // 判断该扩展点接口是否有扩展点实现类
            if (loader.getSupportedExtensions().size() > 0) {
                // 获取type对应接口的自适应类(适配类)实例。如果有@Adaptive注解的类,则返回该类的实例,否则返回一个动态生成类的实例(如Protocol$Adpative的实例)
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}
```

### 1.3 SpringExtensionFactory 类

```java{.line-numbers}
public class SpringExtensionFactory implements ExtensionFactory {

    /** 全局的Spring应用上下文集合 */
    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
	// ServiceBean初始化的时候调用(在setApplicationContext()方法)，更新contexts集合
    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);

    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {
        // 遍历ApplicationContext
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) { // 如果 context 包含名称为 name 的bean
                // 获取名称为 name 的 bean。如果是懒加载或原型的 bean,此时会实例化名称为 name 的 bean
                Object bean = context.getBean(name);
                // 判断 bean 是否是 type 对应的类的实例或者是其子类的实例，等于instanceof
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }
        return null;
    }
}
```

## 2.AOP 解析

以下面的代码执行过程为例，分析 Dubbo AOP 的执行流程。

```java{.line-numbers}
 1 ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);// 1
 2 Protocol dubboProtocol = loader.getExtension("dubbo");// 2 
```

### 2.1 ExtensionLoader 实例化

在第一行代码执行完毕之后，ExtensionLoader 会有如下实例属性：

```java{.line-numbers}
- type: interface com.alibaba.dubbo.rpc.Protocol
- ExtensionFactory objectFactory = AdaptiveExtensionFactory(自适应类(适配类))
- factories = [SpringExtensionFactory instance, SpiExtensionFactory instance] 
```

### 2.2 getExtension 方法解析

这个方法是扩展点具体实现类实例的获取函数，调用路径如下：

```java{.line-numbers}
getExtension(String name)
    createExtension(String name)
        getExtensionClasses().get(name) // 根据 name 获取扩展点实现类 Class 对象
            loadExtensionClasses()
                loadFile(Map<String, Class<?>> extensionClasses, String dir)
        injectExtension(T instance) // IoC 注入
        包装类包装 // AOP
```

从上面可以看出，createExtension 方法最为重要，代码如下：

```java{.line-numbers}
// ExtensionLoader#createExtension
private T createExtension(String name) {
    // getExtensionClasses用来获取配置文件中所有的拓展类，可以得到 "配置项名称" -> "配置类" 的对应关系
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例，并且放入到EXTENSION_INSTANCES中保存起来
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 满足创建的实例instance的依赖关系，通过其 set 方法注入对应的实例，注入到 instance 实例的属性不仅可以通过SPI的方式也可以通过Spring的bean工厂获取
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        // 循环创建 Wrapper 实例，将真正拓展对象包裹在Wrapper对象中，实现了Dubbo AOP的功能
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 循环创建 Wrapper 实例,形成Wrapper包装链
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " + type
                + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```

在 createExtension(String name) 方法调用过程中，会进行 Wrapper 包装，同时根据 META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol 中的配置可知，com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper/com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper 这两个类是实现了 Protocol 接口的包装类，分别具有一个以 Protocol 作为参数的构造器且类上无 @Adaptive 注解，放置在 ExtensionLoader 的私有属性 cachedWrapperClasses 中。

这时的 ExtensionLoader 实例有如下的实例属性：

```java{.line-numbers}
1 - type: interface com.alibaba.dubbo.rpc.Protocol
2 - ExtensionFactory objectFactory = AdaptiveExtensionFactory(自适应类(适配类))
3 	- factories = [SpringExtensionFactory instance, SpiExtensionFactory instance]
4 - cachedWrapperClasses = [class ProtocolListenerWrapper, class ProtocolFilterWrapper]
``` 

在 createExtension(String name) 方法中的 AOP 包装部分的逻辑如下：

- 获取 ProtocolListenerWrapper 的单参构造器，以 DubboProtocol 实例为构造器参数创建 ProtocolListenerWrapper 实例，并完成对 ProtocolListenerWrapper 实例的属性注入。此时的instance=ProtocolListenerWrapper 实例，不再是之前的 DubboProtocol 实例。
- 使用 ProtocolFilterWrapper 以同样的方式对 ProtocolListenerWrapper 实例进行包装。

Wrapper 包装之后的关系如下所示：

```java{.line-numbers}
instance = ProtocolFilterWrapper实例 {
    protocol = ProtocolListenerWrapper实例 {
        protocol = DubboProtocol实例
    }
}
```

因此上面的第二行代码执行之后，返回的 Protocol 实例是 ProtocolFilterWrapper 对象，也就是包装之后的对象：

<div align="center">
    <img src="apache_dubbo_static//15.png" width="400"/>
</div>

以上便是 Dubbo IOC、AOP 的整个过程。

## 3.Dubbo AOP 和 IOC 总结

Dubbo 中 AOP 和 IOC 都是在 ExtensionLoader#createExtension 方法中实现的。createExtension 主要进行以下4个方面的操作：

1. 从 ClassPath 下 META-INF 文件夹下读取特定扩展点配置文件，并获取配置文件中的各个子类，然后将目标 name 对应的子类返回（返回的为 Class<?> 类型的对象）
2. 通过反射创建实例
3. 为实例注入依赖（通过反射调用实例中的 set 方法），这是 IOC 的实现
   1. 获取依赖的属性名，比如 setName 方法对应属性名 name
   2. 从 objectFactory 中获取依赖对象，也就是从 dubbo 自身的扩展机制中去找是否有对应扩展点的实现类实例，或者去 Spring 的 bean 工厂中去获取。
   3. 通过反射调用实例的 setter 方法设置依赖
4. 获取配置文件中定义的 wrapper 对象，然后使用该 wrapper 对象封装目标对象，并且还会调用其 set 方法为 wrapper 对象注入其所依赖的属性，这是 AOP 的实现

关于 wrapper 对象，这里需要说明的是，其主要作用是为目标对象实现 AOP。Wrapper 对象有两个特点：

- 与目标对象实现了同一个目标接口
- 有一个以目标接口为参数类型的拷贝构造函数。

这也就是上述 createExtension() 方法最后封装 wrapper 对象时传入的构造函数实例始终可以为 instance 实例的原因

