---
layout: post
title: JDK动态代理
categories: 低层原理
excerpt: JDK动态代理是代理模式的一种实现方式，其只能代理接口。
tags: 动态代理
---

### 一、 使用步骤

1. 新建一个接口
2. 为接口创建一个实现类
3. 创建代理类实现java.lang.reflect.InvocationHandler接口
4. 生成动态代理

### 二、 代码案例

#### 新建一个接口
首先新建一个接口Subject
```java
public interface Subject {

    void doSomething();

    void doAnother();
}
```

#### 为接口创建一个实现类
然后为接口Subject新建一个实现类RealSubject
```java
public class RealSubject implements Subject{
    @Override
    public void doSomething() {
        System.out.println("RealSubject do some thing");
    }

    @Override
    public void doAnother() {
        System.out.println("RealSubject do another thing");
    }
}
```

#### 创建代理类实现java.lang.reflect.InvocationHandler接口
接着创建一个代理类MyInvocationHandler, 实现java.lang.reflect.InvocationHandler接口，重写invoke方法
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class MyInvocationHandler implements InvocationHandler {
    private Object target;

    public MyInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Do something before");
        Object result = method.invoke(target, args);
        System.out.println("Do something after");
        return result;
    }
}
```

#### 生成动态代理
新建测试类Client测试结果

类加载器和接口要从代理目标对象的实例对象realSubject获取。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

public class Client {
    public static void main(String[] args) {
        // 保存生成的代理类的字节码文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        Subject realSubject = new RealSubject();
        InvocationHandler invocationHandler = new MyInvocationHandler(realSubject);
        // jdk动态代理测试
        Subject proxyInstance = (Subject) Proxy.newProxyInstance(realSubject.getClass().getClassLoader(), realSubject.getClass().getInterfaces(), invocationHandler);
        proxyInstance.doSomething();
        proxyInstance.doAnother();
    }
}
```

输出结果：
```text
Do something before
RealSubject do some thing
Do something after
Do something before
RealSubject do another thing
Do something after
```

### 三 、 分析源代码
由于设置 

sun.misc.ProxyGenerator.saveGeneratedFiles 的值为true,所以代理类的字节码内容保存在了项目根目录下，文件名为$Proxy0.class



源码分析
这里查看JDK1.8.0_65的源码，通过debug学习JDK动态代理的实现原理

大概流程

1、为接口创建代理类的字节码文件

2、使用ClassLoader将字节码文件加载到JVM

3、创建代理类实例对象，执行对象的目标方法, 将所有方法调用都委托给InvocationHandler的invoke方法。

首先看Proxy类中的newProxyInstance方法：


动态代理涉及到的主要类：

java.lang.reflect.Proxy
java.lang.reflect.InvocationHandler
java.lang.reflect.WeakCache
sun.misc.ProxyGenerator

 

首先看Proxy类中的newProxyInstance方法：

```java
@CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
            throws IllegalArgumentException
    {
        // 判断InvocationHandler是否为空，若为空，抛出空指针异常
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * 生成接口的代理类的字节码文件
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * 使用自定义的InvocationHandler作为参数，调用构造函数获取代理类对象实例
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
newProxyInstance方法调用getProxyClass0方法生成代理类的字节码文件。

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        // 限定代理的接口不能超过65535个
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }
        // 如果缓存中已经存在相应接口的代理类，直接返回；否则，使用ProxyClassFactory创建代理类
        return proxyClassCache.get(loader, interfaces);
    }

```

 其中缓存使用的是WeakCache实现的，此处主要关注使用ProxyClassFactory创建代理的情况。ProxyClassFactory是Proxy类的静态内部类，实现了BiFunction接口，实现了BiFunction接口中的apply方法。

当WeakCache中没有缓存相应接口的代理类，则会调用ProxyClassFactory类的apply方法来创建代理类。
```java
private static final class ProxyClassFactory
            implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // 代理类前缀
        private static final String proxyClassNamePrefix = "$Proxy";
        // 生成代理类名称的计数器
        private static final AtomicLong nextUniqueNumber = new AtomicLong();
        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * 校验类加载器是否能通过接口名称加载该类
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
                 * 校验该类是否是接口类型
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                            interfaceClass.getName() + " is not an interface");
                }
                /*
                 * 校验接口是否重复
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                            "repeated interface: " + interfaceClass.getName());
                }
            }

            String proxyPkg = null;     // 代理类包名
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * 非public接口，代理类的包名与接口的包名相同
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
                // public代理接口，使用com.sun.proxy包名
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * 为代理类生成名字
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * 真正生成代理类的字节码文件的地方
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                    proxyName, interfaces, accessFlags);
            try {
                // 使用类加载器将代理类的字节码文件加载到JVM中
                return defineClass0(loader, proxyName,
                        proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```
 在ProxyClassFactory类的apply方法中可看出真正生成代理类字节码的地方是ProxyGenerator类中的generateProxyClass，
 该类未开源，但是可以使用IDEA、或者反编译工具jd-gui来查看。
```java
public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        final byte[] var4 = var3.generateClassFile();
        // 是否要将生成代理类的字节码文件保存到磁盘中
        if (saveGeneratedFiles) {
            // ....
        }
        return var4;
    }
```
在测试案例中，设置系统属性sun.misc.ProxyGenerator.saveGeneratedFiles值为true
```java
// 保存生成的代理类的字节码文件
 System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```


打开$Proxy0.class文件如下：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.sun.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import vip.sunjin.proxy.Subject;

public final class $Proxy0 extends Proxy implements Subject {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m4;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void doSomething() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void doAnother() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("vip.sunjin.proxy.Subject").getMethod("doSomething");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m4 = Class.forName("vip.sunjin.proxy.Subject").getMethod("doAnother");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```

可看到

1. 代理类继承了Proxy类并且实现了要代理的接口，由于java不支持多继承，所以JDK动态代理不能代理类
2. 重写了equals、hashCode、toString
3. 有一个静态代码块，通过反射获得代理类的所有方法
4. 通过invoke执行代理类中的目标方法doSomething，doAnother。
