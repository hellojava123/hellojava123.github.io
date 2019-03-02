---
layout: post
title: 详解Spirng-目标方法参数-Reader和@RequestBody在Spring中如何抉择
date: 2017-08-04 11:11:11.000000000 +09:00
---
首先，我们都知道在使用spring框架的时候，无论是springmvc还是springboot，都可以在Controller层使用一些参数来取得我们想要的目的。

首先我们debug，进入最核心的方法中，发现核心方法就是ServletInvocableHandlerMethod类中的invokeAndHandle方法。
````java
方法入口和结束的地方就是ServletInvocableHandlerMethod.invokeAndHandle()方法中
public void invokeAndHandle(ServletWebRequest webRequest, 
          ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
````
````java
//  此处进入项目代码
 Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
// 此处接受项目代码返回值
 setResponseStatus(webRequest);

 if (returnValue == null) {
  if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
   mavContainer.setRequestHandled(true);
   return;
  }
 }
 else if (StringUtils.hasText(this.responseReason)) {
  mavContainer.setRequestHandled(true);
  return;
 }

 mavContainer.setRequestHandled(false);
 try {
  this.returnValueHandlers.handleReturnValue(
    returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
 }
 catch (Exception ex) {
  if (logger.isTraceEnabled()) {
   logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
  }
  throw ex;
 }
}

重点看invokeForRequest：
调用Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);

核心代码：InvocableHandlerMethod.getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,  Object... providedArgs) throws Exception {

private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
  Object... providedArgs) throws Exception {
// 获取所有的参数
 MethodParameter[] parameters = getMethodParameters();
// 创建数组
 Object[] args = new Object[parameters.length];
// 遍历参数
 for (int i = 0; i < parameters.length; i++) {
// 得到具体参数
  MethodParameter parameter = parameters[i];

  parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
  GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
  args[i] = resolveProvidedArgument(parameter, providedArgs);
  if (args[i] != null) {
   continue;
  }
  if (this.argumentResolvers.supportsParameter(parameter)) {
   try {
// 注意，此处是核心，进入此方法，则改变返回的到底之什么值
    args[i] = this.argumentResolvers.resolveArgument(
      parameter, mavContainer, request, this.dataBinderFactory);
    continue;
   }
   catch (Exception ex) {
    if (logger.isDebugEnabled()) {
     logger.debug(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
    }
    throw ex;
   }
  }
  if (args[i] == null) {
   String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
   throw new IllegalStateException(msg);
  }
 }
 return args;
}
````

####调用HandlerMethodArgumentResolverComposite.resolveArgument（）
```java
@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
// 此处很重要，会返回不同的子类，不同的子类调用下面的方法有不同的实现
 HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
 if (resolver == null) {
  throw new IllegalArgumentException("Unknown parameter type [" + parameter.getParameterType().getName() + "]");
 }
 return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```
####然后调用resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory)；根据不同的resolver对象返回不同的值。这就是spring的抉择。应该算是状态模式吧。
那我们看看每次返回的具体子类是什么：
// Reader的方法参数的实现类是@ServletRequestMethodArgumentResolver
// @RequestBody注解的实现是@RequestResponseBodyMethodProcessor
// Map参数的实现类是@HandlerMethodArgumentResolverComposite

然后他们调用各自的resolveArgument(parameter, mavContainer, webRequest, binderFactory)方法
去实现自己的逻辑去解析。从而返回不同的参数。
```java
/**
 * Find a registered {@link HandlerMethodArgumentResolver} that supports the given method parameter.
 */
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
// 此方法决定返回什么值
 HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
 if (result == null) {
  for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
   if (logger.isTraceEnabled()) {
    logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
      parameter.getGenericParameterType() + "]");
   }
// 就是根据这个，是否支持某参数来确定resolver的返回的子类
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
#####下面的方法返回了Reader
```java
@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

 Class<?> paramType = parameter.getParameterType();
 if (WebRequest.class.isAssignableFrom(paramType)) {
  return webRequest;
 }

 HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
 if (ServletRequest.class.isAssignableFrom(paramType) || MultipartRequest.class.isAssignableFrom(paramType)) {
  Object nativeRequest = webRequest.getNativeRequest(paramType);
  if (nativeRequest == null) {
   throw new IllegalStateException(
     "Current request is not of type [" + paramType.getName() + "]: " + request);
  }
  return nativeRequest;
 }
 else if (HttpSession.class.isAssignableFrom(paramType)) {
  return request.getSession();
 }
 else if (HttpMethod.class == paramType) {
  return ((ServletWebRequest) webRequest).getHttpMethod();
 }
 else if (Principal.class.isAssignableFrom(paramType)) {
  return request.getUserPrincipal();
 }
 else if (Locale.class == paramType) {
  return RequestContextUtils.getLocale(request);
 }
 else if (TimeZone.class == paramType) {
  TimeZone timeZone = RequestContextUtils.getTimeZone(request);
  return (timeZone != null ? timeZone : TimeZone.getDefault());
 }
 else if ("java.time.ZoneId".equals(paramType.getName())) {
  return ZoneIdResolver.resolveZoneId(request);
 }
 else if (InputStream.class.isAssignableFrom(paramType)) {
  return request.getInputStream();
 }
 else if (Reader.class.isAssignableFrom(paramType)) {
// 返回了Reader
  return request.getReader();
 }
 else {
  // should never happen...
  throw new UnsupportedOperationException(
    "Unknown parameter type: " + paramType + " in method: " + parameter.getMethod());
 }
}


```

######此方法返回了Json对象
```java
@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

 parameter = parameter.nestedIfOptional();
// 就是下面的方法，层层调用，最后使用ObjectMapper解析了字符串，变成了@RequestBody中的对象
 Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
 String name = Conventions.getVariableNameForParameter(parameter);

 WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
 if (arg != null) {
  validateIfApplicable(binder, parameter);
  if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
   throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
  }
 }
 mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());

 return adaptArgumentIfNecessary(arg, parameter);
}


我们重写了解析了json的类，重写了240行的代码，方便我们解析参数
package org.springframework.http.converter.json;

@Override
public Object read(Type type, Class<?> contextClass, HttpInputMessage inputMessage)
    throws IOException, HttpMessageNotReadableException {

  JavaType javaType = getJavaType(type, contextClass);
  return readJavaType(javaType, inputMessage);
}

private Object readJavaType(JavaType javaType, HttpInputMessage inputMessage) {
  try {
    if (inputMessage instanceof MappingJacksonInputMessage) {
      Class<?> deserializationView = ((MappingJacksonInputMessage) inputMessage).getDeserializationView();
      if (deserializationView != null) {
        return this.objectMapper.readerWithView(deserializationView).forType(javaType).
            readValue(inputMessage.getBody());
      }
    }
//在这里修改源码
    InputStream inputStream = inputMessage.getBody();
    String body = FileCopyUtils.copyToString(new InputStreamReader(inputStream,"UTF-8"));
    body = mxr(body);
    return this.objectMapper.readValue(body, javaType);
  }
  catch (IOException ex) {
    throw new HttpMessageNotReadableException("Could not read document: " + ex.getMessage(), ex);
  }
}


```
这篇文章以代码居多，通过对spring如何修改参数的设置，我们可以在他的逻辑里定义自己想要的逻辑，比如修改参数。
