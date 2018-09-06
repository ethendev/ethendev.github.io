---
layout: post
title: Java对URI路径中的分号进行编码
tags:  [Java，URI]
categories: [Java]
keywords: Java,URI,semicolon,分号编码
---


在URL中，由于 ";" 是保留字符，Java 默认不会对它转码，在某些情况下会出现问题。




在 influxDB 中，从多个 measurement 中查询数据的SQL使用 ";" 分隔，使用 CURL 能得到正确结果
```
curl -v -G 'http://127.0.0.1:8086/query?db=test' --data-urlencode 'q=show databases;SHOW MEASUREMENTS'
...
> GET /query?db=test&q=show%20databases%3BSHOW%20MEASUREMENTS HTTP/1.1
...
<
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "databases",
                    "columns": [
                        "name"
                    ],
                    "values": [
                        [
                            "_internal"
                        ],
                        [
                            "test"
                        ]
                    ]
                }
            ]
        },
        {
            "statement_id": 1,
            "series": [
                {
                    "name": "measurements",
                    "columns": [
                        "name"
                    ],
                    "values": [
                        [
                            "proxy"
                        ]
                    ]
                }
            ]
        }
    ]
}
```

但是在Java中，只返回第一个SQL结果。和上面 curl 的请求对比，发现是参数 q 中的 ";" 分号没有转码成 %3B 导致的。 
```
public class Test {
    public static void main(String[] args) {
        String url = "http://127.0.0.1:8086/query?db=test&q={query}";
        URI uri = new UriTemplate(url).expand("show databases;show measurements");

        System.out.println(uri.toString());

        RestTemplate restTemplate = new RestTemplate();
        HttpEntity<String> response = restTemplate.getForEntity(uri, String.class);
        System.out.println(response.getBody());
    }
}

```

运行结果
```
http://127.0.0.1:8086/query?db=test&q=show%20databases;show%20measurements
{"results":[{"statement_id":0,"series":[{"name":"databases","columns":["name"],"values":[["_internal"],["test"]]}]}]}
```

不能直接使用 'uri.toString().replace(";", "%3B")' 将分号替换成 "%3B", 因为后面还会再一次编码变为 "%253B" 而报错。


后来在 spring-web-5.0.4.RELEASE.jar 中发现了 DefaultUriBuilderFactory 类。首先看一下其源码
```
public class DefaultUriBuilderFactory implements UriBuilderFactory {
    private final UriComponentsBuilder baseUri;
    private final Map<String, Object> defaultUriVariables;
    private DefaultUriBuilderFactory.EncodingMode encodingMode;
    private boolean parsePath;

    public DefaultUriBuilderFactory() {
        this(UriComponentsBuilder.newInstance());
    }

    public DefaultUriBuilderFactory(String baseUriTemplate) {
        this(UriComponentsBuilder.fromUriString(baseUriTemplate));
    }

    public DefaultUriBuilderFactory(UriComponentsBuilder baseUri) {
        this.defaultUriVariables = new HashMap();
        this.encodingMode = DefaultUriBuilderFactory.EncodingMode.URI_COMPONENT;
        this.parsePath = true;
        Assert.notNull(baseUri, "'baseUri' is required");
        this.baseUri = baseUri;
    }
    
    public void setEncodingMode(DefaultUriBuilderFactory.EncodingMode encodingMode) {
        this.encodingMode = encodingMode;
    }
    
    
    ......
}
```

DefaultUriBuilderFactory 中有一个变量 encodingMode，可以通过它设置编码模式。EncodingMode 是 DefaultUriBuilderFactory 的内部类，里面定义了3中编码模式，代码如下：
```
public class DefaultUriBuilderFactory implements UriBuilderFactory {
    ......
    
    public static enum EncodingMode {
        URI_COMPONENT,
        VALUES_ONLY,
        NONE;

        private EncodingMode() {
        }
    }
}
```

使用 DefaultUriBuilderFactory 来改写上面有问题的代码后，可以正确获取结果了
```
public class Test {
    public static void main(String[] args) {
        String url = "http://127.0.0.1:8086/query?db=test&q={query}";

        RestTemplate restTemplate = new RestTemplate();

        DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory();
        factory.setEncodingMode(DefaultUriBuilderFactory.EncodingMode.VALUES_ONLY);
        URI uri = factory.expand(url,"show databases;show measurements");
        
        System.out.println(uri.toString());

        HttpEntity<String> response = restTemplate.getForEntity(uri, String.class);
        System.out.println(response.getBody());
    }
}
```

运行结果
```
http://127.0.0.1:8086/query?db=test&q=show%20databases%3Bshow%20measurements
{"results":[{"statement_id":0,"series":[{"name":"databases","columns":["name"],"values":[["_internal"],["test"]]}]},
{"statement_id":1,"series":[{"name":"measurements","columns":["name"],"values":[["proxy"]]}]}]}
```