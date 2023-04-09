---
title: mybatis5-总结
tags:
notebook: mybatis源码解析
---
### "#"与"$"的区别
1. mybatis处理"${}"时是通过StringBuilder的append()方法处理的，进行字符串拼接的，会存在sql侵入的风险，比如"select * from ${table}"，如果传参是"duty_a"，则sql变成"select * from duty_a"；处理"#"时是用"?"替换"#{}"，然后调用PreparedStatement中的带有类型的set方法将参数值写到sql中，"select * from 'duty_a'",不会有sql侵入的风险。
2. 如果sql带有"${}"则是动态sql，会在运行的时候解析，影响效率；"#{}"会进行预编译，在运行之前就用"?"替换
### 缓存失效的情况
1. 不同的 SqlSession 对象对应不同的一级缓存，即使查询相同的数据，也要重新访问数据库；
2. 同一个 SqlSession 对象，但是查询的条件不同；
3. 同一个 SqlSession 对象两次查询期间执行了任何的“增删改”操作，无论这些“增删改”操作是否影响到了缓存的数据；
4. 同一个 SqlSession 对象两次查询期间手动清空了缓存（调用了 SqlSession 对象的 clearCache() 方法）。
