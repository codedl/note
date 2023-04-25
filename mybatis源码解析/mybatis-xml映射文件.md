+ collection标签
collection标签说明这是个集合；property表示在类中定义的集合属性名为posts；javaType表示类中定义的集合类型为ArrayList；column表示传给子查询的参数为属性id，可以表示成{pid = id, display = queryDisplay}，意思将属性id的值映射到pid上，queryDisplay的值映射到display上；ofType表示集合中存储的元素类型为Post；select表示集合中元素的值是通过selectPostsForBlog方法获取的。
```xml
<collection property="posts" 
            javaType="ArrayList" 
            column="id" 
            ofType="Post" 
            select="selectPostsForBlog">
<collection/>
```
