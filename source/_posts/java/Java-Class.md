---
title: class loading
date: 2019-06-05 23:48
tags: Java
---

<!--more-->
##类的加载

### 双亲委托模型
双亲委派指的就是
类加载器需要加载类的时候，会先把请求委托给父类加载器，依次递归，如果父类能完成，就返回，只有父类或祖类等无法完成，才自己加载;

JVM预定义的三个类型的加载器（classloader）
1. Bootstrap加载器
    一般在<JAVA_RUNTIME_HOME>/lib下的类库加载到内存中，因为其涉及JVM本身实现细节，使用C++实现，所以开发者没法获得其引用
2. Extension类加载器
    sun的ExtClassloader负责<JAVA_RUNTIME_HOME>/lib/ext下或由java.ext.dir指定位置中的类库加载到内存中;
3. System类加载器
    AppClassloader实现，负责把classpath中指定的类加载;


####目的
防止内存出现多个同样的字节码，也能保证用到真正JDK里面提供的类;

然而这个**并不是强制模型**

#### 需要注意：
1. 加载的时候，会派出**当前线程**的类加载器（当前类加载器通过Thread.getContextClassLoader()，也可以通过setContextLoader()设置类加载器
2. 如果加载该类的时候，有其他类引用此类，其他类也会被该类加载器加载
3. 可直接调用Classloader.loadClass()指定某个类加载器

SPI（service Provider Interface）
//todo