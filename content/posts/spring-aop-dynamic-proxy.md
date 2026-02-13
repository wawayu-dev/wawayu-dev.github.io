---
title: "Spring AOP 原理与动态代理实现深度解析"
date: 2024-04-10
tags: ["Spring", "设计模式", "源码分析", "最佳实践"]
categories: ["技术"]
description: "深入理解 AOP 的底层实现机制，掌握 JDK 动态代理与 CGLIB 代理的区别"
draft: false
---

AOP（面向切面编程）是 Spring 框架的核心特性之一。通过 AOP，我们可以将横切关注点（如日志、事务、权限）从业务逻辑中分离出来。但 AOP 是如何实现的？本文深入剖析 Spring AOP 的底层机制。

### 本文亮点
- [x] 理解 AOP 的核心概念：切面、切点、通知、连接点
- [x] 掌握 JDK 动态代理与 CGLIB 代理的实现原理
- [x] 学会 ProxyFactory 的创建流程
- [x] 了解切面执行顺序与责任链模式

---

## AOP 核心概念

**切面（Aspect）**

横切关注点的模块化，比如日志切面、事务切面。

```java
@Aspect
@Component
public class LogAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        log.info("方法执行前: {}", joinPoint.getSignature().getName());
    }
}
```

**切点（Pointcut）**

定义在哪些连接点上应用通知，通过表达式匹配。

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}
```

**通知（Advice）**

在切点上执行的动作，分为五种类型：

- @Before：前置通知
- @After：后置通知
- @AfterReturning：返回通知
- @AfterThrowing：异常通知
- @Around：环绕通知

**连接点（JoinPoint）**

程序执行的某个点，比如方法调用、异常抛出。

---

## JDK 动态代理 vs CGLIB 代理

Spring AOP 使用两种代理方式：

**JDK 动态代理**

基于接口的代理，要求目标类实现接口。

```java
public interface UserService {
    void addUser(User user);
}

@Service
public class UserServiceImpl implements UserService {
    @Override
    public void addUser(User user) {
        // 业务逻辑
    }
}
```

JDK 动态代理的实现：

```java
public class JdkProxyFactory implements InvocationHandler {
    
    private Object target;
    
    public Object createProxy(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            this
        );
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before method");
        Object result = method.invoke(target, args);
        System.out.println("After method");
        return result;
    }
}
```

**CGLIB 代理**

基于继承的代理，通过生成目标类的子类实现。

```java
@Service
public class UserService {  // 没有实现接口
    public void addUser(User user) {
        // 业务逻辑
    }
}
```

CGLIB 代理的实现：

```java
public class CglibProxyFactory implements MethodInterceptor {
    
    public Object createProxy(Object target) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }
    
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before method");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("After method");
        return result;
    }
}
```

**选择规则**

- 如果目标类实现了接口，默认使用 JDK 动态代理
- 如果目标类没有实现接口，使用 CGLIB 代理
- 可以通过配置强制使用 CGLIB：`@EnableAspectJAutoProxy(proxyTargetClass = true)`

> **架构思考：** JDK 动态代理和 CGLIB 代理体现了"策略模式"的思想。Spring 根据目标类的特征，选择合适的代理策略。这种设计让框架更加灵活，能够适应不同的使用场景。

---

## ProxyFactory 创建流程

Spring AOP 通过 ProxyFactory 创建代理对象：

```java
@Component
public class AopProxyCreator {
    
    public Object createProxy(Object target) {
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.setTarget(target);
        
        // 添加通知
        proxyFactory.addAdvice(new MethodBeforeAdvice() {
            @Override
            public void before(Method method, Object[] args, Object target) {
                System.out.println("Before: " + method.getName());
            }
        });
        
        return proxyFactory.getProxy();
    }
}
```

ProxyFactory 的创建流程：

1. 创建 AopProxyFactory（默认是 DefaultAopProxyFactory）
2. 根据目标类选择代理方式（JDK 或 CGLIB）
3. 创建 AopProxy 对象（JdkDynamicAopProxy 或 CglibAopProxy）
4. 生成代理对象

---

## 切面执行顺序

当有多个切面时，执行顺序由 @Order 注解控制：

```java
@Aspect
@Component
@Order(1)
public class LogAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore() {
        System.out.println("Log Before");
    }
}

@Aspect
@Component
@Order(2)
public class TransactionAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void transactionBefore() {
        System.out.println("Transaction Before");
    }
}
```

执行顺序：

```
Log Before
Transaction Before
目标方法执行
Transaction After
Log After
```

这是责任链模式的典型应用。

---

## 实战案例：自定义注解实现权限校验

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();
}

@Aspect
@Component
public class PermissionAspect {
    
    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint, RequirePermission requirePermission) throws Throwable {
        String permission = requirePermission.value();
        User user = UserContext.getUser();
        
        if (!user.hasPermission(permission)) {
            throw new PermissionDeniedException("无权限访问");
        }
        
        return joinPoint.proceed();
    }
}

// 使用
@Service
public class UserService {
    
    @RequirePermission("user:delete")
    public void deleteUser(Long userId) {
        // 业务逻辑
    }
}
```

---

## 总结与思考

Spring AOP 的核心是动态代理，通过代理对象在目标方法前后插入横切逻辑。

- JDK 动态代理基于接口，CGLIB 代理基于继承
- ProxyFactory 负责创建代理对象
- 多个切面通过责任链模式串联执行

理解 AOP 的实现原理，才能更好地使用它，避免踩坑。

下次当你需要实现横切关注点时，不妨试试 AOP，让代码更加优雅。
