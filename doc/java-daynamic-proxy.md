# JAVA之动态代理

### 起因：

Hadoop的RPC部分客户端调用RPC.getProxy获取服务端协议接口的代理对象。好奇内部是怎么做到的。

> Hadoop Client RPC Demo:

```java
/**
 * LoginController is a RPC client
 */
public class LoginController {
    public static void main(String[] args) throws Exception {
        //☆通过RPC拿到LoginServiceInterface的代理对象
        LoginServiceInterface loginService =
                RPC.getProxy(LoginServiceInterface.class, 1L, new InetSocketAddress("hadoop01", 10000), new Configuration());
 
        //像本地调用一样调用服务端的方法
        String result = loginService.login("flyne");
        System.out.println(result);
 
        //关闭连接
        RPC.stopProxy(loginService);
    }
}
```

> 查看RPC.getProxy源码。发现是通过getProtocolProxy返回ProtocolProxy对象，然后get里面的proxy属性，所以要查看getProtocolProxy怎么生成ProtocolProxy对象的

```java
public static <T> T getProxy(Class<T> protocol,
                              long clientVersion,
                              InetSocketAddress addr, Configuration conf)
  throws IOException {

  return getProtocolProxy(protocol, clientVersion, addr, conf).getProxy();
}
```

> 查看PRC.getProtocolProxy源码。首先是调用getProtocolEngine返回一个协议引擎，然后通过这个RpcEngine生成ProtocolProxy，所以我们着重看一下RpcEngine.getProxy方法

```java
 public static <T> ProtocolProxy<T> getProtocolProxy(Class<T> protocol,
                              long clientVersion,
                              InetSocketAddress addr,
                              UserGroupInformation ticket,
                              Configuration conf,
                              SocketFactory factory,
                              int rpcTimeout,
                              RetryPolicy connectionRetryPolicy,
                              AtomicBoolean fallbackToSimpleAuth)
     throws IOException {
  if (UserGroupInformation.isSecurityEnabled()) {
    SaslRpcServer.init(conf);
  }
  return getProtocolEngine(protocol, conf).getProxy(protocol, clientVersion,
      addr, ticket, conf, factory, rpcTimeout, connectionRetryPolicy,
      fallbackToSimpleAuth);
}
```

> 查看WritableRpcEngine.getProxy源码。发现这里proxy的生成，是通过Proxy.newProxyInstance方法的，Proxy类是java.lang.reflect包下类。于是开始探索，这个类的newProxyInstance方法做了什么，能够返回一个实现protocol接口的对象（可以直接强制转型使用），而且能够把protocol的操作代理到Invoker类完成。

```java
/** Construct a client-side proxy object that implements the named protocol,
 * talking to a server at the named address. 
 * @param <T>*/
@Override
@SuppressWarnings("unchecked")
public <T> ProtocolProxy<T> getProxy(Class<T> protocol, long clientVersion,
                       InetSocketAddress addr, UserGroupInformation ticket,
                       Configuration conf, SocketFactory factory,
                       int rpcTimeout, RetryPolicy connectionRetryPolicy,
                       AtomicBoolean fallbackToSimpleAuth)
  throws IOException {    

  if (connectionRetryPolicy != null) {
    throw new UnsupportedOperationException(
        "Not supported: connectionRetryPolicy=" + connectionRetryPolicy);
  }

  T proxy = (T) Proxy.newProxyInstance(protocol.getClassLoader(),
      new Class[] { protocol }, new Invoker(protocol, addr, ticket, conf,
          factory, rpcTimeout, fallbackToSimpleAuth));
  return new ProtocolProxy<T>(protocol, proxy, true);
}
```

* <u>为什么说是实现protocol接口呢？</u>

  因为默认这个对象会继承Proxy类，由于java的单继承机制，只能实现其他的接口。所以这里的Class<T> protocol只能是一个接口。

### 初探：

经过以上的起因，我们发现Proxy.newProxyInstance的作用类似于在运行时生成一个实现protocol接口的对象，具体方法的实现，在Invoker中定义（因为RPC的服务端地址等信息在Invoker中）。

##### Proxy.newInstance

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    Objects.requireNonNull(h);

    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    /*
     * Look up or generate the designated proxy class. 查找或者生成目标的代理类
     */
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
     * Invoke its constructor with the designated invocation handler. 调用目标代理类的构造方法，并且把handler赋值进去。
     */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

总结下，Proxy.newInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)做了两件事.

1. 根据参数loader和interfaces调用方法 getProxyClass(loader, interfaces)创建代理类$Proxy.\$Proxy0类**<u>实现了interfaces的接口,并继承了Proxy类.</u>**
2. 实例化$Proxy0并在构造方法中把BusinessHandler传过去,接着\$Proxy0调用父类Proxy的构造器,为h赋值,如下:

```java
class Proxy{
   InvocationHandler h=null;
   protected Proxy(InvocationHandler h) {
     this.h = h;
   }
   ...
}
```

下面我们来看看具体是怎么生成$Proxy0这个类的，即` Class<?> cl = getProxyClass0(loader, intfs);`做了什么：

##### Proxy.getProxyClass0

```java
/**
 * Generate a proxy class.  Must call the checkProxyAccess method
 * to perform permission checks before calling this.
 */
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    // 直接从proxyClassCache中获取，如果没有则通过ProxyClassFactory创建
    return proxyClassCache.get(loader, interfaces);
}
```

这个方法我们发现逻辑很简单，就是从一个代理类的缓存中获取，如果没有，则会自动创建。其实看到这里就可以了，但是本着刨根问底的科学探究精神，我觉得继续看下去。

所以这里我们需要再查看具体是怎么创建的，看下这个proxyClassCache对象。

```java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
    proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

这是一个WeakCache对象，对应这里的构造方法是：

```java
/**
 * Construct an instance of {@code WeakCache}
 *
 * @param subKeyFactory a function mapping a pair of
 *                      {@code (key, parameter) -> sub-key}
 * @param valueFactory  a function mapping a pair of
 *                      {@code (key, parameter) -> value}
 * @throws NullPointerException if {@code subKeyFactory} or
 *                              {@code valueFactory} is null.
 */
public WeakCache(BiFunction<K, P, ?> subKeyFactory,
                 BiFunction<K, P, V> valueFactory) {
    this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
    this.valueFactory = Objects.requireNonNull(valueFactory);
}
```

KeyFactory就是生成的Proxy对应的key的方法，ProxyClassFactory就是生成Proxy的方法。

于是我们根据这些基本前提看下proxyClassCache.get方法是怎么获取缓存中的类以及创建没有缓存的类的。

##### WeakCache.get

```java
/**
 * Look-up the value through the cache. This always evaluates the
 * {@code subKeyFactory} function and optionally evaluates
 * {@code valueFactory} function if there is no entry in the cache for given
 * pair of (key, subKey) or the entry has already been cleared.
 *
 * @param key       possibly null key
 * @param parameter parameter used together with key to create sub-key and
 *                  value (should not be null)
 * @return the cached value (never null)
 * @throws NullPointerException if {@code parameter} passed in or
 *                              {@code sub-key} calculated by
 *                              {@code subKeyFactory} or {@code value}
 *                              calculated by {@code valueFactory} is null.
 */
public V get(K key, P parameter) {
    Objects.requireNonNull(parameter);

    expungeStaleEntries();

    Object cacheKey = CacheKey.valueOf(key, refQueue);

    // lazily install the 2nd level valuesMap for the particular cacheKey
    // 懒加载【二级】cacheKey的valuesMap。为什么这里说是二级的，可以看下map的结构
    // ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }

    // create subKey and retrieve the possible Supplier<V> stored by that
    // subKey from valuesMap
    // 这里是通过刚刚的valuesMap获取subKey对应的Supplier，这里可以看到上面WeakCache构造方法提到的subKeyFactory，来生成二级的key
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    Supplier<V> supplier = valuesMap.get(subKey);
    Factory factory = null;

    while (true) {
        if (supplier != null) {
            // supplier might be a Factory or a CacheValue<V> instance
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
        // else no supplier in cache
        // or a supplier that returned null (could be a cleared CacheValue
        // or a Factory that wasn't successful in installing the CacheValue)

        // lazily construct a Factory
        if (factory == null) {
            // 这里就是我们要找的，真正生成代理类的地方了
            factory = new Factory(key, parameter, subKey, valuesMap);
        }

        if (supplier == null) {
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                // successfully installed Factory
                supplier = factory;
            }
            // else retry with winning supplier
        } else {
            if (valuesMap.replace(subKey, supplier, factory)) {
                // successfully replaced
                // cleared CacheEntry / unsuccessful Factory
                // with our Factory
                supplier = factory;
            } else {
                // retry with current supplier
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```

我们发现代理类是包装在一个Supplier类中，并调用Supplier.get方法返回的。于是我们需要查看这里的Supplier的实现对象Factory。即`factory = new Factory(key, parameter, subKey, valuesMap);`生成的类是什么样的。

##### WeakCahce.Factory

```java
/**
 * A factory {@link Supplier} that implements the lazy synchronized
 * construction of the value and installment of it into the cache.
 */
private final class Factory implements Supplier<V> {

    private final K key;
    private final P parameter;
    private final Object subKey;
    private final ConcurrentMap<Object, Supplier<V>> valuesMap;

    Factory(K key, P parameter, Object subKey,
            ConcurrentMap<Object, Supplier<V>> valuesMap) {
        this.key = key;
        this.parameter = parameter;
        this.subKey = subKey;
        this.valuesMap = valuesMap;
    }

    @Override
    public synchronized V get() { // serialize access
        // re-check
        Supplier<V> supplier = valuesMap.get(subKey);
        if (supplier != this) {
            // something changed while we were waiting:
            // might be that we were replaced by a CacheValue
            // or were removed because of failure ->
            // return null to signal WeakCache.get() to retry
            // the loop
            return null;
        }
        // else still us (supplier == this)

        // create new value
        V value = null;
        try {
            value = Objects.requireNonNull(valueFactory.apply(key, parameter));
        } finally {
            if (value == null) { // remove us on failure
                valuesMap.remove(subKey, this);
            }
        }
        // the only path to reach here is with non-null value
        assert value != null;

        // wrap value with CacheValue (WeakReference)
        CacheValue<V> cacheValue = new CacheValue<>(value);

        // try replacing us with CacheValue (this should always succeed)
        if (valuesMap.replace(subKey, this, cacheValue)) {
            // put also in reverseMap
            reverseMap.put(cacheValue, Boolean.TRUE);
        } else {
            throw new AssertionError("Should not reach here");
        }

        // successfully replaced us with new CacheValue -> return the value
        // wrapped by it
        return value;
    }
}
```

在这里我们发现，这个Supplier的get方法，首先检查确定自己是这个subKey对应的supplier。然后这里见到了我们的老朋友，WeakCache构造方法里输入的valueFactory，来生成真实的目标类。然后用CacheValue包装（目的是hashcode和equal方法重写，这边有待继续探究这么做的原因//todo），替换掉自己。

这里我们找到了最本质生成目标类的调用，`valueFactory.apply(key, parameter)`所以，我们来看下这个工厂的apply是怎么来生成的，之前有提到这个valueFactory是ProxyClassFactory对象。因此我们来看下它。目测这是最终篇章了，加油骚年。

##### Proxy.ProxyClassFactory

```java
/**
 * A factory function that generates, defines and returns the proxy class given
 * the ClassLoader and array of interfaces.
 */
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    // prefix for all proxy class names
    private static final String proxyClassNamePrefix = "$Proxy";

    // next number to use for generation of unique proxy class names
    private static final AtomicLong nextUniqueNumber = new AtomicLong();

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            /*
             * Verify that the class loader resolves the name of this
             * interface to the same Class object.
             */
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            /*
             * Verify that the Class object actually represents an
             * interface.
             */
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            /*
             * Verify that this interface is not a duplicate.
             */
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }

        String proxyPkg = null;     // package to define proxy class in
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        /*
         * Record the package of a non-public proxy interface so that the
         * proxy class will be defined in the same package.  Verify that
         * all non-public proxy interfaces are in the same package.
         */
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {
            // if no non-public proxy interfaces, use com.sun.proxy package
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        /*
         * Choose a name for the proxy class to generate.
         */
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        /*
         * Generate the specified proxy class.
         */
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            /*
             * A ClassFormatError here means that (barring bugs in the
             * proxy class generation code) there was some other
             * invalid aspect of the arguments supplied to the proxy
             * class creation (such as virtual machine limitations
             * exceeded).
             */
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

可以看到这里分为四步。

- 验证代理接口和ClassLoader是否匹配

- 生成包名（根据是否存在非公共接口）-> 生成代理类的名字

- 生成代理类字节码`ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags);`

  ProxyGenerator是sum.misc包中的类。ProxyGenerator.generateProxyClass方法会调用generateClassFile方法用于生产字节码，大致扫了一眼，应该是以一定规则，把Proxy类和需要实现的接口，组合写入一个文件。<u>所以生成的类，是继承了Proxy接口，实现了规定的接口的。</u>//todo

- 根据二进制字节码，返回Class实例

  `defineClass0(loader, proxyName, proxyClassFile, 0, proxyClassFile.length);  `

  这是一个native方法。没有细看。//todo

至此，Java动态代理的源码分析告一段落，今后如果遇到需要深入了解的时候，还会继续往下研究。时间有限，还得去看hadoop源码，RPC部分还没看完，尴尬。就到此为止了～

### 应用：

#### AOP：

如Spring中的aspect的编写，面向切片编程，本质上就是通过动态代理机制来实现的。

```java
public interface IVehical {

    void run();
    
}

//concrete implementation
public class Car implements IVehical{

    public void run() {
    System.out.println("Car is running");
    }

}

//proxy class
public class VehicalProxy {

    private IVehical vehical;

    public VehicalProxy(IVehical vehical) {
    this.vehical = vehical;
    }

    public IVehical create(){
    final Class<?>[] interfaces = new Class[]{IVehical.class};
    final VehicalInvacationHandler handler = new VehicalInvacationHandler(vehical);
    
    return (IVehical) Proxy.newProxyInstance(IVehical.class.getClassLoader(), interfaces, handler);
    }
    
    public class VehicalInvacationHandler implements InvocationHandler{

    private final IVehical vehical;
    
    public VehicalInvacationHandler(IVehical vehical) {
        this.vehical = vehical;
    }

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable {

        System.out.println("--before running...");
        Object ret = method.invoke(vehical, args);
        System.out.println("--after running...");
        
        return ret;
    }
    
    }
}

public class Main {
    public static void main(String[] args) {
    
    IVehical car = new Car();
    VehicalProxy proxy = new VehicalProxy(car);
    
    IVehical proxyObj = proxy.create();
    proxyObj.run();
    }
}
/*
 * output:
 * --before running...
 * Car is running
 * --after running...
 * */
```

#### Proxy具备访问控制能力：

联想设计模式中的代理模式：为其他对象提供一种代理以控制对这个对象的访问。比如把一个多接口的对象，通过代理，只暴露单个接口的方法出来。或者是在Handler实现的内部，对方法的访问做限制。

#### 配置文件的接口化：

在项目中，可以使用动态代理，获取配置文件。只需要把配置文件的key注解到对应的接口上即可。

个人理解：<u>补全需要代理的接口</u>。（和一般的Handler中保存接口的子类不同，那个相当于装饰器，这个相当于自己实现，通过一些外部信息）(不过，如果把配置文件类比成一个接口的子类的话，那么也相当于装饰器～只不过是没有装饰什么的装饰器哈哈哈又点拗口啦)

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {

    /**
     * The actual value expression: e.g. "#{systemProperties.myProp}".
     */
    String value();
}

/**
 * config interfaces, map the config properties file:
 * db.url = 
 * db.validation = true
 * db.pool.size = 100
 */
public interface IConfig {
    
    @Value("db.url")
    String dbUrl();
    
    @Value("db.validation")
    boolean isValidated();
    
    @Value("db.pool.size")
    int poolSize();
    
}

//proxy class
public final class ConfigFactory {

    private ConfigFactory() {}

    public static IConfig create(final InputStream is) throws IOException{
    
    final Properties properties = new Properties();
    properties.load(is);
    
    return (IConfig) Proxy.newProxyInstance(IConfig.class.getClassLoader(),
        new Class[] { IConfig.class }, new PropertyMapper(properties));
    
    }

    public static final class PropertyMapper implements InvocationHandler {

    private final Properties properties;

    public PropertyMapper(Properties properties) {
        this.properties = properties;
    }

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable {

        final Value value = method.getAnnotation(Value.class);

        if (value == null) return null;
          
          String property = properties.getProperty(value.value());
          if (property == null) return (null);
          
          final Class<?> returns = method.getReturnType();
          if (returns.isPrimitive())
          {
            if (returns.equals(int.class)) return (Integer.valueOf(property));
            else if (returns.equals(long.class)) return (Long.valueOf(property));
            else if (returns.equals(double.class)) return (Double.valueOf(property));
            else if (returns.equals(float.class)) return (Float.valueOf(property));
            else if (returns.equals(boolean.class)) return (Boolean.valueOf(property));
          }
        
        return property;
    }

    }
}

public static void main(String[] args) throws FileNotFoundException, IOException {
    IConfig config = ConfigFactory.create(new 		 FileInputStream("config/config.properties"));
    String dbUrl = config.dbUrl();
    boolean isLoginValidated = config.isValidated();
    int dbPoolSize = config.poolSize();
}
```



Reference:

[Hadoop RPC机制](http://www.flyne.org/article/1095)

[动态代理源码初探](http://nemotan.github.io/2015/11/java%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F-%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F/)

[动态代理的应用](https://www.cnblogs.com/techyc/p/3455950.html)

