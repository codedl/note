java中提供了Optional可以优先地操作对象，进行判空处理。看下有哪些常见方法吧。
+ of  
创建Optional对象，参数不为空
+ ofNullable  
创建Optional对象，参数可为空
+ get  
获取Optional对象保存的值
+ orElse  
获取对象，如果值不存在则返回默认值
+ orElseGet  
获取对象，如果值不存在则通过方法获取值
+ isPresent  
判断对象是否为空
+ ifPresent  
如果对象不为空就执行方法
+ filter  
对封装的对象进行测试，通过测试对象存在否则返回空
+ map  
转换对象，可以返回任意值
+ flatMap  
转换对象，只能返回非空的Optional对象
