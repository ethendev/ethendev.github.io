---
layout: post
title: 自定义 RestTemplate 异常处理
tags:  [Java]
categories: [Java]
keywords: RestTemplate,ResponseErrorHandler
---


一些 API 的报错信息通过 Response 的 body返回。使用 HttpClient 能正常获取到 StatusCode 和 body 中的错误提示。然而使用 RestTemplate ，会直接抛出下面的异常。如果想获取原始的信息并进一步处理会比较麻烦。




```
org.springframework.web.client.HttpClientErrorException: 404 null
	at org.springframework.web.client.DefaultResponseErrorHandler.handleError(DefaultResponseErrorHandler.java:94)
	at org.springframework.web.client.DefaultResponseErrorHandler.handleError(DefaultResponseErrorHandler.java:79)
	at org.springframework.web.client.ResponseErrorHandler.handleError(ResponseErrorHandler.java:63)
	at org.springframework.web.client.RestTemplate.handleResponse(RestTemplate.java:777)
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:730)
	at org.springframework.web.client.RestTemplate.execute(RestTemplate.java:704)
	at org.springframework.web.client.RestTemplate.exchange(RestTemplate.java:621)
```

### RestTemplate 异常处理流程

下面看一下原因， RestTemplate 中的 getForObject, getForEntity 和 exchange 等常用方法最终都是调用 doExecute 方法。下面是 doExecute 方法源码：
```
public class RestTemplate extends InterceptingHttpAccessor implements RestOperations {

    private ResponseErrorHandler errorHandler;
    ......
    
    @Nullable
    protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback, @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
        Assert.notNull(url, "'url' must not be null");
        Assert.notNull(method, "'method' must not be null");
        ClientHttpResponse response = null;

        String resource;
        try {
            ClientHttpRequest request = this.createRequest(url, method);
            if (requestCallback != null) {
                requestCallback.doWithRequest(request);
            }

            response = request.execute();
            // 处理 Response
            this.handleResponse(url, method, response);
            if (responseExtractor != null) {
                Object var14 = responseExtractor.extractData(response);
                return var14;
            }

            resource = null;
        } catch (IOException var12) {
            resource = url.toString();
            String query = url.getRawQuery();
            resource = query != null ? resource.substring(0, resource.indexOf(63)) : resource;
            throw new ResourceAccessException("I/O error on " + method.name() + " request for \"" + resource + "\": " + var12.getMessage(), var12);
        } finally {
            if (response != null) {
                response.close();
            }

        }

        return resource;
    }

    protected void handleResponse(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
        ResponseErrorHandler errorHandler = this.getErrorHandler();
        boolean hasError = errorHandler.hasError(response);
        if (this.logger.isDebugEnabled()) {
            try {
                this.logger.debug(method.name() + " request for \"" + url + "\" resulted in " + response.getRawStatusCode() + " (" + response.getStatusText() + ")" + (hasError ? "; invoking error handler" : ""));
            } catch (IOException var7) {
                ;
            }
        }
        // 异常处理
        if (hasError) {
            errorHandler.handleError(url, method, response);
        }

    }
}
```

从下面的代码可以看出，DefaultResponseErrorHandler 捕获并抛出了异常。
```
public class DefaultResponseErrorHandler implements ResponseErrorHandler {
    ...
    
    protected void handleError(ClientHttpResponse response, HttpStatus statusCode) throws IOException {
        switch(statusCode.series()) {
        case CLIENT_ERROR:
            throw new HttpClientErrorException(statusCode, response.getStatusText(), response.getHeaders(), this.getResponseBody(response), this.getCharset(response));
        case SERVER_ERROR:
            throw new HttpServerErrorException(statusCode, response.getStatusText(), response.getHeaders(), this.getResponseBody(response), this.getCharset(response));
        default:
            throw new UnknownHttpStatusCodeException(statusCode.value(), response.getStatusText(), response.getHeaders(), this.getResponseBody(response), this.getCharset(response));
        }
    }
}
```

如果想自己捕获异常信息，自己处理异常的话可以通过实现 ResponseErrorHandler 类来实现。其源码如下：

```
public interface ResponseErrorHandler {

    // 标示 Response 是否存在任何错误。实现类通常会检查 Response 的 HttpStatus。
    boolean hasError(ClientHttpResponse var1) throws IOException;

    // 处理 Response 中的错误, 当 HasError 返回 true 时才调用此方法。
    void handleError(ClientHttpResponse var1) throws IOException;

    // handleError 的替代方案，提供访问请求URL和HTTP方法的额外信息。
    default void handleError(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
        this.handleError(response);
    }
}
```

### 自定义 RestTemplate 异常处理

如果想像 HttpClient 一样直接从 Response 获取 HttpStatus 和 body 中的报错信息 而不抛出异常，可以通过下面的代码实现：

```
public class CustomErrorHandler implements ResponseErrorHandler {

    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return true;
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {

    }
}
```

设置 RestTemplate 的异常处理类
```
restTemplate.setErrorHandler(new CustomErrorHandler());
ResponseEntity<String> response = restTemplate.exchange(uri, HttpMethod.GET, null, String.class);
System.out.println(response.getBody());
```

输出结果
```
{"code":404,"result":null,"message":"Resources not found"}
```