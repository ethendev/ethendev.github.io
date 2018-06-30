---
layout: post
title: SpringBoot跨域解决办法
tags:  [SpringBoot, 跨域]
categories: [SpringBoot]
keywords: SpringBoot,跨域
---


项目中经常会遇到前后端分离的情况，分离之后会碰到跨域问题，前端无法访问后端的接口。可以通过如下3种方式解决跨域问题。



### 过滤器实现
```
public class CorsFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest)servletRequest;
        HttpServletResponse response = (HttpServletResponse)servletResponse;
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PUT");
        response.setHeader("Access-Control-Allow-Headers", "x-reqted-with");

        // 如果是option请求，直接返回200
        if (request.getMethod().equals(HttpMethod.OPTIONS.name())) {
            response.setStatus(HttpServletResponse.SC_OK);
            return;
        }
        chain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```

### Controller方法CORS配置
```
@CrossOrigin(origins = "*")
@GetMapping("/getAll")
public List<User> getAll() {
    return mapper.getList();
}
```

### WebMvc实现全局CORS配置
```
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowCredentials(true)
                .allowedHeaders("*")
                .allowedOrigins("*")
                .allowedMethods("*");
    }
}
```
