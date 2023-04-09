---
title: getForObject方法解析
tags:
notebook: RestTemplate源码解析
---
在RestTemplate实例化的时候，会使用RestTemplate的类加载器去加载一些预定义的http消息转换器类，例如：
```java
	private static boolean romePresent =
			ClassUtils.isPresent("com.rometools.rome.feed.WireFeed",
					RestTemplate.class.getClassLoader());
```
当类被加载后进行实例化的时候，会向属性messageConverters(ArrayList)中添加预置的消息转换器，并且会添加当前项目中存在的消息转换器，这样就完成了类的实例化。

通过getForObject()发送请求时，会根据响应类型实例化一个AcceptHeaderRequestCallback对象，通过响应类型和消息转换器实例化HttpMessageConverterExtractor对象，最后通过execute()方法执行http请求，核心逻辑在这个方法中。
1. 获取uri模板处理器，即RestTemplate的属性uriTemplateHandler指向的对象，在实例化的时候创建了uriTemplateHandler的实例对象。`private UriTemplateHandler uriTemplateHandler = new DefaultUriBuilderFactory();`然后调用expand()对请求进行扩展。通过fromUriString()创建封装了请求地址、端口的UriComponentsBuilder对象，再通过DefaultUriBuilderFactory.java类中的build()构建URI对象
```java
public static UriComponentsBuilder fromUriString(String uri) {
	Assert.notNull(uri, "URI must not be null");
	//判断当前uri是否为标准可以处理的uri
	Matcher matcher = URI_PATTERN.matcher(uri);
	if (matcher.matches()) {
		//将请求的uri信息进行封装UriComponentsBuilder对象
		UriComponentsBuilder builder = new UriComponentsBuilder();
		//根据Matcher提前uri字符串中的部分字符串，本质是字符串的截取操作
		String scheme = matcher.group(2);
		String userInfo = matcher.group(5);
		String host = matcher.group(6);
		String port = matcher.group(8);
		String path = matcher.group(9);
		String query = matcher.group(11);
		String fragment = matcher.group(13);
		boolean opaque = false;
		if (StringUtils.hasLength(scheme)) {
			String rest = uri.substring(scheme.length());
			if (!rest.startsWith(":/")) {
				opaque = true;
			}
		}
		builder.scheme(scheme);
		if (opaque) {
			String ssp = uri.substring(scheme.length()).substring(1);
			if (StringUtils.hasLength(fragment)) {
				ssp = ssp.substring(0, ssp.length() - (fragment.length() + 1));
			}
			builder.schemeSpecificPart(ssp);
		}
		else {
			//设置请求地址，端口，请求路径
			builder.userInfo(userInfo);
			builder.host(host);
			if (StringUtils.hasLength(port)) {
				builder.port(port);
			}
			builder.path(path);
			builder.query(query);
		}
		if (StringUtils.hasText(fragment)) {
			builder.fragment(fragment);
		}
		return builder;
	}
	else {
		throw new IllegalArgumentException("[" + uri + "] is not a valid URI");
	}
}
```
2. 根据请求路径和请求方法创建ClientHttpRequest对象。获取创建请求的工厂，RestTemplate类的HttpAccessor父类的属性requestFactory指向SimpleClientHttpRequestFactory对象为默认的请求工厂，通过SimpleClientHttpRequestFactory.java类中的createRequest()方法创建ClientHttpRequest对象。
	1. 首先通过jdk原生的URL的openConnection()方法建立连接URLConnection
	2. 为URLConnection设置超时时间，请求方式
	3. 返回包含URLConnection的SimpleBufferingClientHttpRequest对象
3. 接下来就要处理请求，这里还是在请求的预处理阶段。获取当前客户端能够处理的响应数据类型：获取调用getForObject()方法时声明的返回值类型，再遍历所有的消息转换器，依次调用canRead()方法找到处理这种类型的消息转换器，而消息转换器可以处理多种类型，最终我们将所有能被处理的类型添加到请求头Accept->text/plain, application/json, application/*+json, */*。
4. 调用SimpleBufferingClientHttpRequest对象中的connection(URLConnection)的connect()发送请求，如果服务端能够正确处理请求则将响应返回到connection(URLConnection)中，由此可见RestTemplate本质还是对URLConnection的封装。
5. 获取到服务器返回的响应之后，接下来就是对响应内容的处理。之前发送请求时实例化的HttpMessageConverterExtractor对象，即消息提取器派上用场了。由于此对象包含所有的消息转化器，所以可以找到一个合适的消息转换器对返回值进行处理。这里以StringHttpMessageConverter为例进行说明，看看消息转换器怎么工作的。

调用readInternal()方法进行实质的响应数据处理。
```java
	protected String readInternal(Class<? extends String> clazz, HttpInputMessage inputMessage) throws IOException {
		//字符集默认为 UTF-8
		Charset charset = getContentTypeCharset(inputMessage.getHeaders().getContentType());
		//获取响应的InputStream供数据的读取
		return StreamUtils.copyToString(inputMessage.getBody(), charset);
	}
```
真实的数据处理在这里进行的，逻辑很简单，以utf-8对输入字节流编码成字符串，然后返回就行了，这也就是我们调用getForObject()方法的返回值。
```java
public static String copyToString(@Nullable InputStream in, Charset charset) throws IOException {
	if (in == null) {
		return "";
	}

	StringBuilder out = new StringBuilder();
	InputStreamReader reader = new InputStreamReader(in, charset);
	char[] buffer = new char[BUFFER_SIZE];
	int bytesRead = -1;
	while ((bytesRead = reader.read(buffer)) != -1) {
		out.append(buffer, 0, bytesRead);
	}
	return out.toString();
}
```