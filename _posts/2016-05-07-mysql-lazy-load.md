---
layout: post
title: MyBatis延迟加载
tags:  [Java, MyBatis, 延迟加载]
categories: [MyBatis]
keywords: Java,MyBatis,延迟加载
---


MyBatis的resultMap支持三种级联方式：一对一关系（assocation）、一对多关系（collection）、鉴别器（discriminator）。
使用级联方式获取数据时，数据是及时加载的，会带来性能问题。有些时候想需要数据的时候再按需加载，可以采用MyBatis延迟加载。




举个例子，查询学生信息的时候关联查询学生成绩。如果先查询学生信息即可满足要求，当需要查询成绩时再查询成绩信息。这种按需查询就是延迟加载。


### 延迟加载配置

在mybatis的配置中有两个全局参数 lazyLoadingEnabled 和 aggressiveLazyLoading ：

<!-- 
| 设置项 |  描述 | 可选值 | 默认值  |
| -      |  :-   | :-     | :-      |
| lazyLoadingEnabled | 全局懒加载设置 | true \| false | false |
| aggressiveLazyLoading | 对任意延迟属性的调用会使带有延迟加载属性的对象完整加载 | true \| false | true | 
-->
<table>
  <thead>
    <tr>
      <th style="text-align: left ;width: 20%">设置项</th>
      <th style="text-align: left ;width: 50%">描述</th>
      <th style="text-align: left">可选值</th>
      <th style="text-align: left">默认值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>lazyLoadingEnabled</td>
      <td style="text-align: left">全局懒加载设置</td>
      <td style="text-align: left">true | false</td>
      <td style="text-align: left">false</td>
    </tr>
    <tr>
      <td>aggressiveLazyLoading</td>
      <td style="text-align: left">对任意延迟属性的调用会使带有延迟加载属性的对象完整加载</td>
      <td style="text-align: left">true | false</td>
      <td style="text-align: left">true</td>
    </tr>
  </tbody>
</table>

mybatis默认没有开启延迟加载，需要在mybatis-config.xml的settings进行如下配置。
```
 <settings>
    ......
      <!-- 开启延迟加载 -->
      <setting name="lazyLoadingEnabled" value="true"/>
      <!-- 按需加载 -->
      <setting name="aggressiveLazyLoading" value="false"/>
    ......
  </settings>
```

成绩信息类
```
public class ScoreVo {

    private Long scoreId;
    private String course;
    private Double score;
    private Long studentId;

    // 省略set get方法
}
```

学生信息类
```
public class StudentVo {

    private Long studentId;
    private String name;
    private Integer sex;
    private Date createTime;
    private Date updateTime;
    private List<ScoreVo> scoreList;
    
    // 省略set get方法
}
```

Mapper配置
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.ethen.travel.student.StudentMapper">

    <resultMap id="studentMap" type="com.ethen.travel.student.StudentVo">
        <id property="studentId"  column="student_id"/>
        <result property="name" column="name"/>
        <result property="sex" column="sex"/>
        <collection property="scoreList" column="student_id" select="getScore"/>
    </resultMap>

    <select id="getList" resultMap="studentMap">
        SELECT * FROM student
    </select>

    <select id="getScore" resultType="com.ethen.travel.student.ScoreVo">
        SELECT * FROM score where student_id = #{student_id}
    </select>
</mapper>
```

上面的配置已经能实现延迟加载了。下面我们来看一下效果：
![SQL 日志](https://i.loli.net/2018/07/08/5b4195792776b.png)

从图中可以看出，执行mapper.getList()方法只查询了学生信息，并没有查询成绩，而是在下面循环的时候才去查的成绩。

上面的配置是全局配置，不能指定哪些属性可以立即加载，哪些属性可以延迟加载，不太灵活。MyBatis有局部延迟加载的功能。可以在 association、collection和discriminator 元素上加上fetchType 属性，它有两个取值范围，即eager（立即加载）和lazy（延迟加载），默认值是eager。

### 实现原理

延迟加载的实现原理是通过动态代理来实现的。默认情况下，MyBatis在3.3或者以上版本时，才采用JAVASSIST的动态代理，低版本是CGLIB。它生成一个动态代理对象，里面保存着相关的参数和sql, 一旦我们使用这个代理对象的方法，它就会进入到动态代理对象的方法里，通过发送SQL和参数，把对应的结果从数据库里查找回来，这便是其实现原理。