---
layout: post
title: 使用 Spring REST Docs 创建REST服务文档
tags:  [Java, Spring Restdocs]
categories: [Java]
keywords: Java,Spring Restdocs
---

Spring REST Docs 可以帮助开发人员方便地编写服务文档。

它结合了用 [Asciidoctor](https://asciidoctor.org/) 编写的手写文档和使用 [Spring MVC](https://docs.spring.io/spring/docs/current/spring-framework-reference/#spring-mvc-test-framework) 测试自动生成的片段。这种方法使开发人员摆脱了 [Swagger](https://swagger.io/) 等工具所产生的文档的局限性，代码没有侵入性，方便管理和维护。




它有助于用户生成准确、简洁、结构良好的文档。这样的文档文档允许用户轻松获取所需信息。

### 要求

对于Java 7 和 Spring REST Docs 1.2.x，请使用1.0.x版本的 Spring Auto REST Docs。

对于Java 9 以上版本的JDK，请使用 spring-auto-restdocs-json-doclet-jdk9 作为 doclet 依赖项。


### 使用

Gradle 添加如下配置
```
buildscript {
    ext {
        springBootVersion = '2.1.0.RELEASE'
    }
    repositories {
        mavenCentral()
        maven { url 'http://repo.spring.io/plugins-release' }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath('org.asciidoctor:asciidoctor-gradle-plugin:1.5.7')
    }
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'
apply plugin: 'org.asciidoctor.convert'

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
    mavenCentral()
}


configurations {
    jsondoclet
}

ext {
    snippetsDir = file("$buildDir/generated-snippets")
    javadocJsonDir = file("$buildDir/generated-javadoc-json")
}

dependencies {
    compile('org.springframework.boot:spring-boot-starter-web')
    testCompile('org.springframework.boot:spring-boot-starter-test')

    asciidoctor('org.springframework.restdocs:spring-restdocs-asciidoctor:2.0.2.RELEASE')
    testCompile('org.springframework.restdocs:spring-restdocs-mockmvc:2.0.2.RELEASE')
    testCompile('capital.scalable:spring-auto-restdocs-core:2.0.2')
    jsondoclet('capital.scalable:spring-auto-restdocs-json-doclet-jdk9:2.0.2')
}

task jsonDoclet(type: Javadoc, dependsOn: compileJava) {
    source = sourceSets.main.allJava
    classpath = sourceSets.main.compileClasspath
    destinationDir = javadocJsonDir
    options.docletpath = configurations.jsondoclet.files.asType(List)
    options.doclet = 'capital.scalable.restdocs.jsondoclet.ExtractDocumentationAsJsonDoclet'
    options.memberLevel = JavadocMemberLevel.PACKAGE
}

test {
    systemProperty 'org.springframework.restdocs.outputDir', snippetsDir
    systemProperty 'org.springframework.restdocs.javadocJsonDir', javadocJsonDir

    dependsOn jsonDoclet
}

asciidoctor {
    sourceDir = file('src/main/asciidoc')
    outputDir = file("$buildDir/generated-docs")
    options backend: 'html', doctype: 'book'
    attributes 'source-highlighter': 'highlightjs', 'snippets': snippetsDir
    dependsOn test
}

asciidoctor.doLast {
    copy {
        from file("$buildDir/generated-docs/html5")
        into file("$sourceSets.main.output.classesDir/public")
        include 'index.html', 'images/**'
    }
}

jar {
    dependsOn asciidoctor
}
```

User 代码
```
public class User {
    private Integer id;
    /**
     * the user of name
     */
    private String name;
    private Integer age;
    private String sex;
    
    // 省略 setter getter
}
```

Controller 代码：
```
@RestController
@RequestMapping("user")
public class UserController {

    /**
     * add user
     * @param user
     * @return
     */
    @PostMapping("/addUser")
    public User addUser(@RequestBody User user) {
        return user;
    }

    /**
     * Get user by userId
     * @param userId
     * @return
     */
    @GetMapping("/{userId}")
    public User getUser(@PathVariable Integer userId) {
        User user = new User(userId, "lisi", 20, "man");
        return user;
    }
}
```

配置 MockMvc
```
@RunWith(SpringRunner.class)
@SpringBootTest
public abstract class MockMvcBase {

    @Autowired
    protected WebApplicationContext context;

    @Autowired
    protected ObjectMapper objectMapper;

    @Rule
    public final JUnitRestDocumentation restDocumentation = new JUnitRestDocumentation(resolveOutputDir());

    private String resolveOutputDir() {
        String outputDir = System.getProperties().getProperty(
                "org.springframework.restdocs.outputDir");
        if (outputDir == null) {
            outputDir = "target/generated-snippets";
        }
        return outputDir;
    }

    protected MockMvcSnippetConfigurer commonDocumentationConfiguration() {
        return documentationConfiguration(restDocumentation)
                .uris()
                .withScheme("http")
                .withHost("example.com")
                .withPort(80)
                .and().snippets()
                .withDefaults(curlRequest(), httpRequest(), httpResponse(), responseFields(), 
                        //requestFields(),pathParameters(), requestParameters(),
                        description(), methodAndPath(),
                        section());
    }

    protected MockMvc getRestDocumentationMockMvc(MockMvcSnippetConfigurer configurer) throws Exception {
        return MockMvcBuilders
                .webAppContextSetup(context)
                .alwaysDo(prepareJackson(objectMapper))
                .alwaysDo(commonDocumentation())
                .apply(configurer)
                .build();
    }

    protected RestDocumentationResultHandler commonDocumentation() {
        return document("{class-name}/{method-name}",
                preprocessRequest(), commonResponsePreprocessor());
    }

    protected OperationResponsePreprocessor commonResponsePreprocessor() {
        return preprocessResponse(replaceBinaryContent(), limitJsonArrayLength(objectMapper), prettyPrint());
    }

    protected String toJson(Object obj) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

添加单元测试
```
public class UserDoc extends MockMvcBase {

    @Test
    public void AddUser() throws Exception {
        User user = new User(userId, "lisi", 20, "man");
        String param = toJson(user);
        getRestDocumentationMockMvc(
                commonDocumentationConfiguration()
                        .withAdditionalDefaults(requestFields()))
                .perform(post("/user/addUser")
                        .contentType(MediaType.APPLICATION_JSON_UTF8).content(param))
                .andExpect(status().is2xxSuccessful());
    }

    @Test
    public void getUser() throws Exception {
        String result = getRestDocumentationMockMvc(
                commonDocumentationConfiguration()
                        .withAdditionalDefaults(pathParameters()))
                .perform(get("/user/{userId}", 100))
                .andExpect(status().isOk()).andReturn().getResponse().getContentAsString();
        System.out.println(result);
    }
}
```

在 src\main\asciidoc 下面新建一个 index.adoc 文件，并添加如下内容
```
= Spring Rest Doc Test
:toc: right
:toclevels: 2

[[resources-Test]]
== User
include::{snippets}/user-doc/add-user/auto-section.adoc[]
include::{snippets}/user-doc/get-user/auto-section.adoc[]
```

运行下面命令打包并生成文档
```
gradle clean build asciidoctor
```

打开下面的地址，就可以看到API 文档了。

项目路径/build/generated-docs/html5/index.html

![document image](https://i.loli.net/2018/11/13/5bea31b0d6f92.jpg)

