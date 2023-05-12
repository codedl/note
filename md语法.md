---
title: markdown文档语法
tags:
notebook: 常用工具
---
[跳转到一级标题](#一级标题)  
[跳转到二级标题](#二级标题)  
[跳转到三级标题](#三级标题)  
>一级引用开头1，由>开头由>结尾的中间内容内容就是引用
>>二级引用开头 
>    
>一级引用开头2 
>
+ 列表一级标题  
+ 列表一级标题
+ 列表一级标题
  + 列表二级标题
  + 列表二级标题
  + 列表二级标题
    + 列表三级标题
    + 列表三级标题
    + 列表三级标题  
 
1. 带数字的列表  
2. 带数字的列表  
3. 带数字的列表  
    1. 带数字的列表
    2. 带数字的列表
    3. 带数字的列表  

新行  

  
*斜体*  
**粗体**  
***粗斜体***  
***换行符是space+space+enter***
# 一级标题  
这是一级标题的内容，是getSingleton方法的说明
&ensp;&ensp;两个空格
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
```
## 二级标题
二级标题对应的内容是removeSingleton方法
```
    protected void removeSingleton(String beanName) {
        synchronized (this.singletonObjects) {
            this.singletonObjects.remove(beanName);
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.remove(beanName);
        }
    }
```
### 三级标题
```
    protected void beforeSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }
```
![Alt](./pic/user.png)


