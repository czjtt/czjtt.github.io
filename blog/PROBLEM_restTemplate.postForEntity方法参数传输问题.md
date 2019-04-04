## 问题

在一次springcloud-ribbon的使用中，调用了restTemplate对象的postForEntity方法。

消费者	传输的参数对象类型为Map<String, Object> map = new HashMap()<>;

接受者	接收不了消费者传输过来的数据。

```java
Map<String, Object> map = new HashMap<>();
map.put("pa", "post_ribbon-consumer");
return restTemplate.postForEntity("http://RIBBON-PROVIDER/posthello", map, String.class).getBody();
```

```java
@RequestMapping(value = "/posthello", method = RequestMethod.POST)
    public String postHello(
            @RequestParam(name = "pa", required = false, defaultValue = "1") Object a,
            HttpServletRequest request, HttpServletResponse response
    ){
        return "Hello world!" + a;
    }
```

```html
Hello world!1
```

以上是实际代码和返回的结果。正常如果传输成功应该返回的是：

```html
Hello world!post_ribbon-consumer
```

## 代码跟踪但是发现Map对象是正常传输过去的

RestTemplate.class

​	postForEntity  --> execute --> doExecute --> requestCallback.doWithRequest(request) 

代码跟踪流程，doWithRequest通过名字来理解，request请求回调开始执行。

doWithRequest方法的注释：

```html
Gets called by RestTemplate.execute with an opened ClientHttpRequest. Does not need to care about closing the request or about handling errors: this will all be handled by the RestTemplate.
```

方法代码：

```java
@Override
		@SuppressWarnings("unchecked")
		public void doWithRequest(ClientHttpRequest httpRequest) throws IOException {
			super.doWithRequest(httpRequest);
			if (!this.requestEntity.hasBody()) {
				HttpHeaders httpHeaders = httpRequest.getHeaders();
				HttpHeaders requestHeaders = this.requestEntity.getHeaders();
				if (!requestHeaders.isEmpty()) {
					httpHeaders.putAll(requestHeaders);
				}
				if (httpHeaders.getContentLength() < 0) {
					httpHeaders.setContentLength(0L);
				}
			}
			else {
				Object requestBody = this.requestEntity.getBody();
				Class<?> requestBodyClass = requestBody.getClass();
				Type requestBodyType = (this.requestEntity instanceof RequestEntity ?
						((RequestEntity<?>)this.requestEntity).getType() : requestBodyClass);
				HttpHeaders requestHeaders = this.requestEntity.getHeaders();
				MediaType requestContentType = requestHeaders.getContentType();
				for (HttpMessageConverter<?> messageConverter : getMessageConverters()) {
					if (messageConverter instanceof GenericHttpMessageConverter) {
						GenericHttpMessageConverter<Object> genericMessageConverter = (GenericHttpMessageConverter<Object>) messageConverter;
						if (genericMessageConverter.canWrite(requestBodyType, requestBodyClass, requestContentType)) {
							if (!requestHeaders.isEmpty()) {
								httpRequest.getHeaders().putAll(requestHeaders);
							}
							if (logger.isDebugEnabled()) {
								if (requestContentType != null) {
									logger.debug("Writing [" + requestBody + "] as \"" + requestContentType +
											"\" using [" + messageConverter + "]");
								}
								else {
									logger.debug("Writing [" + requestBody + "] using [" + messageConverter + "]");
								}

							}
							genericMessageConverter.write(
									requestBody, requestBodyType, requestContentType, httpRequest);
							return;
						}
					}
					else if (messageConverter.canWrite(requestBodyClass, requestContentType)) {
						if (!requestHeaders.isEmpty()) {
							httpRequest.getHeaders().putAll(requestHeaders);
						}
						if (logger.isDebugEnabled()) {
							if (requestContentType != null) {
								logger.debug("Writing [" + requestBody + "] as \"" + requestContentType +
										"\" using [" + messageConverter + "]");
							}
							else {
								logger.debug("Writing [" + requestBody + "] using [" + messageConverter + "]");
							}

						}
						((HttpMessageConverter<Object>) messageConverter).write(
								requestBody, requestContentType, httpRequest);
						return;
					}
				}
				String message = "Could not write request: no suitable HttpMessageConverter found for request type [" +
						requestBodyClass.getName() + "]";
				if (requestContentType != null) {
					message += " and content type [" + requestContentType + "]";
				}
				throw new RestClientException(message);
			}
		}
```

我们主要看for循环中的处理。

 getMessageConverters()方法会返回一下其中类型的HttpMessageConverter对象。

0 = {ByteArrayHttpMessageConverter@8500} 
1 = {StringHttpMessageConverter@8506} 
2 = {ResourceHttpMessageConverter@8507} 
3 = {SourceHttpMessageConverter@8508} 
4 = {AllEncompassingFormHttpMessageConverter@8509} 
5 = {Jaxb2RootElementHttpMessageConverter@8510} 
6 = {MappingJackson2HttpMessageConverter@8245} 

Map对象的参数 前面6种HttpMessageConverter中的canWrite()方法返回的都是false。即Map对象不符合前面六种messageConverter的要求。只能是默认的MappingJackson2HttpMessageConverter来处理了。还好我们的GenericHttpMessageConverter对象不挑食，canWrite方法返回的是true。

既然能writer那就看看writer方法吧。这里的contentType参数是null, 继续跟着代码流程：

AbstractGenericHttpMessageConverter.wirte() --> AbstractHttpMessageConverter.addDefaultHeaders()

```java
protected void addDefaultHeaders(HttpHeaders headers, T t, MediaType contentType) throws IOException{
		if (headers.getContentType() == null) {
			MediaType contentTypeToUse = contentType;
			if (contentType == null || contentType.isWildcardType() || contentType.isWildcardSubtype()) {
				contentTypeToUse = getDefaultContentType(t);
			}
			else if (MediaType.APPLICATION_OCTET_STREAM.equals(contentType)) {
				MediaType mediaType = getDefaultContentType(t);
				contentTypeToUse = (mediaType != null ? mediaType : contentTypeToUse);
			}
			if (contentTypeToUse != null) {
				if (contentTypeToUse.getCharset() == null) {
					Charset defaultCharset = getDefaultCharset();
					if (defaultCharset != null) {
						contentTypeToUse = new MediaType(contentTypeToUse, defaultCharset);
					}
				}
				headers.setContentType(contentTypeToUse);
			}
		}
		if (headers.getContentLength() < 0 && !headers.containsKey(HttpHeaders.TRANSFER_ENCODING)) {
			Long contentLength = getContentLength(t, headers.getContentType());
			if (contentLength != null) {
				headers.setContentLength(contentLength);
			}
		}
	}
```

怎么说呢，代码追踪到request.execute()那一步其实Map对象的参数也是传过去的。

接收端来看

RequestMappingHandlerAdapter.invokeHandlerMethod(...) --> ServletInvocableHandlerMethod.invokeAndHandle(...) --> InvocableHandlerMethod.invokeForRequest(...) --> InvocableHandlerMethod.getMethodArgumentValues(...) --> HandlerMethodArgumentResolverComposite.resolveArgument(...)

```java
@Override
	public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

		HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
		if (resolver == null) {
			throw new IllegalArgumentException("Unknown parameter type [" + parameter.getParameterType().getName() + "]");
		}
		return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
	}
```

这个方法就到了分叉口。这个resolver对象有很多的实现类，当然是根据getArgumentResolver(parameter)来获取的。

```java
/**
	 * Find a registered {@link HandlerMethodArgumentResolver} that supports the given method parameter.
	 */
	private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		if (result == null) {
			for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
				if (logger.isTraceEnabled()) {
					logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
							parameter.getGenericParameterType() + "]");
				}
				if (methodArgumentResolver.supportsParameter(parameter)) {
					result = methodArgumentResolver;
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```

当然是根据parameter的类型来判断了，this.argumentResolverCache里面总共有26种HandlerMethodArgumentResolver对象。可以说是HandlerMethodArgumentResolver的所有叶子节点的实现类了。

map对象对应的是RequestResponseBodyMethodProcessor实现类

MultiValueMap对象对应的是RequestParamMethodArgumentResolver实现类(执行resolveArgument()方法的是它的父类AbstractNamedValueMethodArgumentResolver)

## 解决方案

### 用Map传输

代码追踪到request.execute()那一步其实Map对象的参数也是传过去的，但是为什么接收不到呢？

其实原因在接收端，接收的时候是我的接收方式写的不对。接收端应该这样写：

```java
@RequestMapping(value = "/posthello", method = RequestMethod.POST)
    public String posthello(
            @RequestBody Map map,
            HttpServletRequest request, HttpServletResponse response
    ){
        return "Hello world!" + map.get("pa");
    }
```



### 为什么这个MultiValueMap就可以呢？

网上很多人给的解决方法用MultiValueMap对象来代替。MultiValueMap就不用像Map对象那样修改接收端就可以正常接收了。

```java
MultiValueMap<String, Object> multiValueMap = new LinkedMultiValueMap<String, Object>();
```

我们可以回到HttpMessageConverter对象列表中看看。有一个叫AllEncompassingFormHttpMessageConverter的对象。这个对象的继承了FormHttpMessageConverter对象，粗略一看是不是跟我们平常理解的post表单提交的表单的意思。

看看它的messageConverter.canWrite方法。

```java
@Override
	public boolean canWrite(Class<?> clazz, MediaType mediaType) {
		if (!MultiValueMap.class.isAssignableFrom(clazz)) {
			return false;
		}
		if (mediaType == null || MediaType.ALL.equals(mediaType)) {
			return true;
		}
		for (MediaType supportedMediaType : getSupportedMediaTypes()) {
			if (supportedMediaType.isCompatibleWith(mediaType)) {
				return true;
			}
		}
		return false;
	}
```

同样的在接收端其实也会经过messageConverter的判断，所以通过了FormHttpMessageConverter这个对象的canRead方法，就沿用了read方法，read方法做什么的事情：

```java
@Override
	public MultiValueMap<String, String> read(Class<? extends MultiValueMap<String, ?>> clazz,
			HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {

		MediaType contentType = inputMessage.getHeaders().getContentType();
		Charset charset = (contentType.getCharset() != null ? contentType.getCharset() : this.charset);
		String body = StreamUtils.copyToString(inputMessage.getBody(), charset);

		String[] pairs = StringUtils.tokenizeToStringArray(body, "&");
		MultiValueMap<String, String> result = new LinkedMultiValueMap<String, String>(pairs.length);
		for (String pair : pairs) {
			int idx = pair.indexOf('=');
			if (idx == -1) {
				result.add(URLDecoder.decode(pair, charset.name()), null);
			}
			else {
				String name = URLDecoder.decode(pair.substring(0, idx), charset.name());
				String value = URLDecoder.decode(pair.substring(idx + 1), charset.name());
				result.add(name, value);
			}
		}
		return result;
	}
```

它相当于重新把参数都设置了一下，成为了name,value格式。所以接收端可以不用和Map对象那样进行接收也可以接收到。

### 使用Pojo对象进行传输

其实和Map对象的传递接收是一样的形式。

Pojo对象：

```java
public class User implements Serializable {

    private int id;
    private String name;
    private int age;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

消费者：
```java
User user = new User();
user.setAge(20);
user.setId(1);
user.setName("nihao");
return restTemplate.postForEntity("http://RIBBON-PROVIDER/posthelloUser", user, String.class).getBody();
```

提供者：
```java
@RequestMapping(value = "/posthelloUser", method = RequestMethod.POST)
    public String posthelloUser(
            @RequestBody User user,
            HttpServletRequest request, HttpServletResponse response
    ){
        
        return "Hello world!" + ",user:" + user.toString();
    }
```

输出结果：
```java
Hello world!,user:User{id=1, name='nihao', age=20}
```

