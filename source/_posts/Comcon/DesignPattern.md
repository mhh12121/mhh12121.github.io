---
title: Notes about Design Pattern（java）
date: 2018-04-30 23:48
tags: Design Pattern
---
<!--more-->
For reviewing some most likely used design pattern:

## **Singleton**

Most commonly
```java
public class Singleton{
    private static final Singleton singleton = new Singleton();

    private Singleton(){}//in case of creating a new object

    public static Singleton getSingleton(){
        return singleton;
    }
    public static other(){
        //other method
    }
}    

```

Pros:
1. Because only one instance exists in memory, it greatly decrease the usage of memory.Especially in the case that an object needs to be initialized and deleted frequently,while the instantiation and deletion can't be optimized, Singleton pattern is the best choice.

2. Prevent multiple domination towards the same source.For example, an operation of writing in a file.

3. 

Cons:
1. Obviously, because singleton has instatiated itself, so it's impossible to extend interface to it (abstract class,interface will not be instantiated )

Scenes:
1. generate unique key;
2. IO;
3. global stored in memory for calculating purpose.For exmaple, Spring Bean is set as a singleton by default so that spring can manage this bean life cycle(but if Bean use non-singleton like prototype pattern,after the initialization of Bean, it will be transfered to J2EE container,Spring no longer manages this Bean)


## **Proxy**

### **static proxy**
We assume there is an interface:
```java
public interface IUser{
    void login();
}
```

Our target UserDAO:
```java
public class User implements IUser{
    public void login(){
        System.out.Println("you've logined");
    }
}
```
Proxy Object: UserProxy
```java
public class UserProxy implements IUser{
    private User target;
    public UserProxy (User target){
        this.target=target;
    }
    public void login(){
        System.out.println("begin login");
        target.login();
        System.out.println("login end");
    }
}
```

**How to Use:**

```java
public class Test{
    public static void main(String[] args){
        User target=new User();
        UserProxy p=new UserProxy(target);
        p.login();
    }
}
```
#### Pros:
 It can extend the target's function without change target's code
#### Cons:
If methods in interface increase, both proxy class and target class have to be recoded
Proxy class **must achieve the same interface** as target class's, so it may cause loads of proxy class


### **dynamic proxy**

#### JDK
Spring 默认使用JDK，**proxy-target-class=false**时使用jdk，true的时候用CGllib代理
Able to solve cons in Static proxy:
1. you **don't have to implements interface{} in Proxy class(But the target class must implement interface)**
动态代理的区别主要就是，代理的proxy class并**不用自己实现接口**，而是在runtime的时候动态在内存生成


Creating a dynamic proxy is a static method in **java.lang.reflect.Proxy** package
```java
static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,InvocationHandler h){

}
```


用法：
```java
public class GamePlayIH implements InvocationHandler {
     //被代理者
     Class cls =null;
     //被代理的实例
     Object obj = null;
     //我要代理谁
     public GamePlayIH(Object _obj){
             this.obj = _obj;
     }
     //调用被代理的方法
     public Object invoke(Object proxy, Method method, Object[] args)
                    throws Throwable {
                      
             Object result = method.invoke(this.obj, args);
             return result;
     }
}
```
```java
public class Test{
    public static void main(){
        User u = new User();//注意要有一个实例才可以
        InvocationHandler i=new GamePlayIH(u);
        
        IUser dynamicUser=(IUser) Proxy.newProxyInstance(User.class.getClassLoader(),User.class.getInterfaces(),i);
        dynamicUser.login();
    }
}
```
看下newProxyInstance源码：
```java
public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h) throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);
        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
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
So how to generate the Proxy Class? **getProxyClass0**
```java
private static Class<?> getProxyClass0(ClassLoader loader， Class<?>... interfaces) {
    if (interfaces.length > 65535) {//为啥不能超过65535？数字怎么来？//网络MTU？？jvm限制方法数目不能大于这个数
        throw new IllegalArgumentException("interface limit exceeded");
    }
    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);//proxyClass的获取可以从Cache里面得到，也可以从proxyFactory里面得到：
}
```
COntinue to **proxyFactory**

```java
/**
     * A factory function that generates, defines and returns the proxy class given
     * the ClassLoader and array of interfaces.
     */
    private static final class ProxyClassFactory implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // Proxy class会附带前缀 $Proxy
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
                if (interfaceClass != intf) {//可能会对classloader隐藏
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
            String proxyName = proxyPkg + proxyClassNamePrefix + num;//这里就是动态生成proxy名字的地方

            /*
             * Generate the specified proxy class.
             */
             //这里就是生成proxy class的地方
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,//defineClass0似乎是到字节码的代码了，没法进入
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


#### CGLIB（Code Generation Library)

**If the target object doesn't implement any interface{}**
you can use **Cglib proxy**

Its principle is that it will create an object **inherit** to target class 
整个CGlib是基于ASM字节码的生成和转换的库
也是用上面的例子,没有implement任何interface

```java
public class UserNo{
    public void login(){
        System.out.println("login without interface");
    }
}
```
How to use:

```java
public class UserMethodInterceptor implements MethodInterceptor{
    //..
    @Override
    public Object intercept(Obj obj, Method method, Object[] args,MethodProxy proxy) throws Throwable{
        System.out.println("method name:"+ method.getName());
        System.out.println("method declaring CLass:"+ method.getDeclaringClass());
        System.out.println("this is "+ (String)proxy.invokeSuper(obj,args));
        System.out.println("method name:"+ method.getName());
        
    }
}
```
```java
public class Test{
    public static void main(){
        Enhancer enhancer=new Enhancer();
        enhancer.setSuperClass(User.class);//根本不用类的实例传入（与JDK有点不一样）
        enhancer.setCallback(new UserMethodINterceptor);

        User u=(User) enhancer.create();
        
        System.out.println("login proxy:");
        u.login();
    }
}

```
前面说到因为这是基于继承的动态代理，所以因为 **final** 修饰的方法等是无法被继承的，所以像
getClass(),wait()等并不会进行代理
如果要强行代理，便会出错：
```java
java.lang.IllegalArgumentException: Cannot subclass final class cglib.HelloConcrete
```


源码在**java.net.sf.cglib**

里面暂时注意Enhancer class和MethodInterceptor Class

