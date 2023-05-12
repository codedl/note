---
title: @Value注解处理
tags:
notebook: spring源码解析
---
spring在创建bean时通过`AutowiredAnnotationBeanPostProcessor`处理属性，实现依赖的自动注入，@Value注解也是在这个时候被处理的。完成bean的实例化后会调用populateBean()方法对bean的属性进行填充，此时`AutowiredAnnotationBeanPostProcessor`后置处理器登场，调用postProcessProperties()处理@Value注解。首先获取属性的类型，在QualifierAnnotationAutowireCandidateResolver.java中通过getSuggestedValue()方法为属性赋值。
1. 获取作用在属性的上的@Value注解的value属性元数据
2. PropertySourcesPlaceholderConfigurer.java的对@Value注解中的属性的占位符进行解析，最终的解析是在PropertyPlaceholderHelper.java的parseStringValue()方法中。
   1. 计算"${"在字符串中的位置，不存在直接返回
   2. 向后偏移两个字符，即忽略"${"
   3. 对字符串中的每个字符挨个处理，检查"}"是否存在，每次遍历index都在加加，这样就能知道"}"在字符串中的位置。最终返回"}"在字符串中的位置
   4. 根据${} 对字符串进行截取，即去除 ${}
   ```java
   //遍历每个字符，判断是否为}
   	public static boolean substringMatch(CharSequence str, int index, CharSequence substring) {
		if (index + substring.length() > str.length()) {
			return false;
		}
		for (int i = 0; i < substring.length(); i++) {
			if (str.charAt(index + i) != substring.charAt(i)) {
				return false;
			}
		}
		return true;
	}
    //去除${}
    String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
   ```
3. 获取当前环境的属性源，从属性源中获取属性值
4. 对属性值的类型进行转换，如果与目标类型一致则不用转换
5. 判断是否包含表达式，如果包含表达式则需要执行表达式并返回执行的结果
6. 缓存解析结果
7. 以反射方式将值设置到bean的实例字段中
```java
if (value != null) {
    ReflectionUtils.makeAccessible(field);
    field.set(bean, value);
}
```
```java
//对表达式字符串解析，然后从属性源获取真实的值
protected String parseStringValue(
        String value, PlaceholderResolver placeholderResolver, @Nullable Set<String> visitedPlaceholders) {

    int startIndex = value.indexOf(this.placeholderPrefix);
    if (startIndex == -1) {
        return value;
    }

    StringBuilder result = new StringBuilder(value);
    while (startIndex != -1) {
        int endIndex = findPlaceholderEndIndex(result, startIndex);
        if (endIndex != -1) {
            String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
            String originalPlaceholder = placeholder;
            if (visitedPlaceholders == null) {
                visitedPlaceholders = new HashSet<>(4);
            }
            if (!visitedPlaceholders.add(originalPlaceholder)) {
                throw new IllegalArgumentException(
                        "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
            }
            // Recursive invocation, parsing placeholders contained in the placeholder key.
            placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
            // Now obtain the value for the fully resolved key...
            String propVal = placeholderResolver.resolvePlaceholder(placeholder);
            if (propVal == null && this.valueSeparator != null) {
                int separatorIndex = placeholder.indexOf(this.valueSeparator);
                if (separatorIndex != -1) {
                    String actualPlaceholder = placeholder.substring(0, separatorIndex);
                    String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
                    propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
                    if (propVal == null) {
                        propVal = defaultValue;
                    }
                }
            }
            if (propVal != null) {
                // Recursive invocation, parsing placeholders contained in the
                // previously resolved placeholder value.
                propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
                result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
                if (logger.isTraceEnabled()) {
                    logger.trace("Resolved placeholder '" + placeholder + "'");
                }
                startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
            }
            else if (this.ignoreUnresolvablePlaceholders) {
                // Proceed with unprocessed value.
                startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
            }
            else {
                throw new IllegalArgumentException("Could not resolve placeholder '" +
                        placeholder + "'" + " in value \"" + value + "\"");
            }
            visitedPlaceholders.remove(originalPlaceholder);
        }
        else {
            startIndex = -1;
        }
    }
    return result.toString();
}
```
