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

## 代码跟踪

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

这里主要看最后一部分，对于contentLength的计算，getContentLength(t,headers.getContentType())

 AbstractJackson2HttpMessageConverter.getContentLength

```java
@Override
	protected Long getContentLength(Object object, MediaType contentType) throws IOException {
		if (object instanceof MappingJacksonValue) {
			object = ((MappingJacksonValue) object).getValue();
		}
		return super.getContentLength(object, contentType);
	}
```

super.getContentLength(object, contentType)方法返回的是一个null。

对于contentLength的解释：

```html
Content-Length用于描述HTTP消息实体的传输长度the transfer-length of the message-body。在HTTP协议中，消息实体长度和消息实体的传输长度是有区别，比如说gzip压缩下，消息实体长度是压缩前的长度，消息实体 的传输长度是gzip压缩后的长度。
如果有Transfer-Encoding，则优先采用Transfer-Encoding里面的方法来找到对应的长度。比如说Chunked模式。

简单总结后如下：
1、Content-Length如果存在并且有效的话，则必须和消息内容的传输长度完全一致。（经过测试，如果过短则会截断，过长则会导致超时。）
2、如果存在Transfer-Encoding（重点是chunked），则在header中不能有Content-Length，有也会被忽视。
3、如果采用短连接，则直接可以通过服务器关闭连接来确定消息的传输长度。（这个很容易懂）
结合HTTP协议其他的特点，比如说Http1.1之前的不支持keep alive。那么可以得出以下结论：
1、在Http 1.0及之前版本中，content-length字段可有可无。
2、在http1.1及之后版本。如果是keep alive，则content-length和chunk必然是二选一。若是非keep alive，则和http1.0一样。content-length可有可无。
```

那就能理解我们传的Map对象为什么接收不到东西了。因为Map对象不能被HttpMessageConverter处理，只能被MappingJackson2HttpMessageConverter对象处理，并且返回的是Content-Length是null，所以即使有内容，也会被截断成空。

## 解决方案

既然我们熟悉的Map对象不能用来进行传输，那有什么好的方法嘛。那就是网上很多人给的解决方法用MultiValueMap对象来代替。

```java
MultiValueMap<String, Object> multiValueMap = new LinkedMultiValueMap<String, Object>();
```

### 为什么这个MultiValueMap就可以呢？

我们可以回到HttpMessageConverter对象列表中看看。有一个叫AllEncompassingFormHttpMessageConverter的对象。粗略一看是不是跟我们平常理解的post表单提交的表单的意思。

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

这就是我们为什么要用MultiValueMap对象的理由了。





