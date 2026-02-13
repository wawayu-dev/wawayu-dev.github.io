---
title: "Spring Bean 生命周期完全解析"
date: 2024-06-20
tags: ["Spring", "Spring Boot", "源码分析", "最佳实践"]
categories: ["技术"]
description: "从实例化到销毁的完整流程，深入理解 BeanPostProcessor 与循环依赖"
draft: false
---

理解 Spring Bean 的生命周期，是掌握 Spring 框架的关键。本文从源码层面剖析 Bean 的创建过程，帮助你深入理解 Spring 的核心机制。

### 本文亮点
- [x] 掌握 Bean 生命周期的完整流程
- [x] 理解 BeanPostProcessor 的执行时机
- [x] 学会解决循环依赖问题
- [x] 了解 InitializingBean vs @PostConstruct 的区别

---

## Bean 生命周期概览

```
实例化 → 属性填充 → 初始化前 → 初始化 → 初始化后 → 使用 → 销毁
```

详细流程：

1. 实例化（Instantiation）：通过反射创建 Bean 实例
2. 属性填充（Populate）：注入依赖的 Bean
3. Aware 接口回调：BeanNameAware、BeanFactoryAware 等
4. BeanPostProcessor.postProcessBeforeInitialization
5. InitializingBean.afterPropertiesSet
6. 自定义 init-method
7. BeanPostProcessor.postProcessAfterInitialization
8. Bean 可以使用
9. DisposableBean.destroy
10. 自定义 destroy-method

---

## 实例化：反射创建对象

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 1. 实例化 Bean
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    Object bean = instanceWrapper.getWrappedInstance();
    
    // 2. 属性填充
    populateBean(beanName, mbd, instanceWrapper);
    
    // 3. 初始化
    Object exposedObject = initializeBean(beanName, bean, mbd);
    
    return exposedObject;
}
```

---

## 属性填充：依赖注入

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // 获取属性值
    PropertyValues pvs = mbd.getPropertyValues();
    
    // 自动装配（byName、byType）
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
    }
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
    }
    
    // 应用属性值
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

---

## 初始化：回调与自定义逻辑

```java
protected Object initializeBean(String beanName, Object bean, RootBeanDefinition mbd) {
    // 1. Aware 接口回调
    invokeAwareMethods(beanName, bean);
    
    // 2. BeanPostProcessor 前置处理
    Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);
    
    // 3. 初始化方法
    invokeInitMethods(beanName, wrappedBean, mbd);
    
    // 4. BeanPostProcessor 后置处理
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    
    return wrappedBean;
}
```

---

## BeanPostProcessor：扩展点

```java
public interface BeanPostProcessor {
    
    // 初始化前回调
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
    
    // 初始化后回调
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

应用场景：

- @Autowired 注解的处理（AutowiredAnnotationBeanPostProcessor）
- AOP 代理的创建（AbstractAutoProxyCreator）
- @PostConstruct 注解的处理（CommonAnnotationBeanPostProcessor）

---

## 循环依赖：三级缓存

```java
@Service
public class A {
    @Autowired
    private B b;
}

@Service
public class B {
    @Autowired
    private A a;
}
```

Spring 通过三级缓存解决循环依赖：

```java
// 一级缓存：完整的 Bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

// 二级缓存：早期的 Bean（已实例化，未初始化）
private final Map<String, Object> earlySingletonObjects = new HashMap<>();

// 三级缓存：Bean 工厂
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>();
```

解决流程：

1. 创建 A，实例化后放入三级缓存
2. 填充 A 的属性，发现依赖 B
3. 创建 B，实例化后放入三级缓存
4. 填充 B 的属性，发现依赖 A
5. 从三级缓存获取 A 的早期引用，注入到 B
6. B 初始化完成，注入到 A
7. A 初始化完成

> **避坑提示：** 构造器注入无法解决循环依赖，因为实例化时就需要依赖对象。只有 setter 注入和字段注入可以解决循环依赖。

---

## InitializingBean vs @PostConstruct

三种初始化方式的执行顺序：

```java
@Component
public class MyBean implements InitializingBean {
    
    @PostConstruct
    public void postConstruct() {
        System.out.println("1. @PostConstruct");
    }
    
    @Override
    public void afterPropertiesSet() {
        System.out.println("2. InitializingBean.afterPropertiesSet");
    }
    
    public void initMethod() {
        System.out.println("3. init-method");
    }
}
```

推荐使用 @PostConstruct，因为它是 JSR-250 标准，不依赖 Spring。

---

## 总结与思考

Spring Bean 的生命周期体现了"模板方法模式"和"责任链模式"的设计思想。

- 模板方法：定义了 Bean 创建的骨架流程
- 责任链：BeanPostProcessor 形成处理链

理解 Bean 生命周期，才能更好地扩展 Spring 框架。

下次当你需要在 Bean 初始化时执行逻辑，不妨试试 @PostConstruct 或 BeanPostProcessor。
