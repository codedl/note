---
title: Jdk动态代理
tags:
notebook: Java开发笔记
---
实现步骤：
1. 声明要被增强的接口
2. 定义接口的实现类，在实现类中实现接口中声明的方法
3. 定义InvocationHandler的实现类，在invoke方法中添加要增强的逻辑，这些逻辑对接口中声明的每个方法都有效果。invoke方法中会以反射方式调用被增强对象的方法。
4. 通过Proxy.newProxyInstance创建代理类对象。当前调用代理类对象中的方法时就会对真实对象的方法进行增强。  

原理：Proxy类的静态方法newProxyInstance会动态地创建代理类对象。方法中第一个参数是加载代理类的类加载器，第二个参数是要被增强的接口，第三个参数InvocationHandler的实现类，第三个参数将会作为代理类的构造方法的参数。生成的代理类是实现了增强接口的Proxy的子类，由于Java中只能继承一个类，所以不能再继承其他类，因从只能对接口进行增强。  
[`final class $Proxy0 extends Proxy implements Owner`](#代理类class)
```
//被增强的接口为Owner，在接口中声明方法rent，rent为被增强的方法
interface Owner{
    void rent();
}
```
```
//定义接口的实现类，实现接口中声明的方法
class HouseOwner implements Owner{
    @Override
    public void rent() {
        System.out.println("租房完成");
    }

}
```
```
class Intermediary implements InvocationHandler{
    private Owner target;

    public Intermediary(Owner target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("找房....");
        Object result = method.invoke(target, args);
        System.out.println("收房租....");
        return result;
    }
}
```
```
public static void main(String[] args) {
    //设置属性sun.misc.ProxyGenerator.saveGeneratedFiles用来保存代理类class
    System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
    HouseOwner houseOwner = new HouseOwner();
    //先创建代理类，再生成代理类对象
    Owner owner = (Owner) Proxy.newProxyInstance(houseOwner.getClass().getClassLoader(),
                                                    houseOwner.getClass().getInterfaces(),
                                                    new Intermediary(houseOwner));
    owner.rent();
}
```
```
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

    //生成代理类class
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
        * Invoke its constructor with the designated invocation handler.
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
        //以反射方式调用代理的构造函数创建代理类对象
        //参数为InvocationHandler接口，因此要定义InvocationHandler的实现类
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
# 代理类class
```
final class $Proxy0 extends Proxy implements Owner {
    private static Method m1;
    private static Method m3;
    private static Method m2;
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

    public final void rent() throws  {
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
            m3 = Class.forName("com.example.springsource.aop.Owner").getMethod("rent");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

