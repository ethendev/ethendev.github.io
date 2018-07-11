---
layout: post
title: Spring MVC不能直接接收List类型参数的问题
tags:  [Java, Spring MVC]
categories: [Java]
keywords: Java,Spring MVC,List,参数
---


前端使用jquery向后台传递数组类型的参数，Java后台直接通过List类型接收，会发现无法取到参数。就像下面的情况：




#### 前端代码
```
$.ajax{
      url:"xxxx",
      data:{
          areaList: ["123", "456", "789"]
      }
      ......
}
```


#### 后台代码
```
@RequestMapping("/getEventData")
public void getEventData(List<String> areaList) {
    // TODO
}
```


这么写的话，在java后台是无法取到参数的，因为jQuery需要调用jQuery.param序列化参数。


#### jQuery param() 方法
语法:  jQuery.param(object, traditional)

|    参数    |        描述      |
|     :-      |        :-      |
|object      | 必需。规定要序列化的数组或对象。 |
|traditional | 可选。布尔值，指定是否使用参数序列化的传统样式。 |


如果后台非要用List接收参数的话，有2种方法可以实现。


方法一 ：ajax中添加traditional:true。

```
$.ajax{
      url:"xxxx",
      traditional: true,
      data:{
          areaList: ["123", "456", "789"]
      }
      ......
}
```

方法二：创建一个对象，将list类型的参数封装在对象中。  

先定义一个ParamVo对象，里面声明一个areaList属性。然后将后台代码改成下面的样子就可以接收到前端的参数了。
```
public class ParamVo {
 
	private List<String> areaList;
 
	public List<String> getAreaList() {
		return areaList;
	}
 
	public void setAreaList(List<String> areaList) {
		this.areaList = areaList;
	}
 
}
```

```
@RequestMapping("/getEventData")
public void getEventData(ParamVo param) {
    // TODO
}
```


方法三：POST方法时Java后端使用@RequestBody注解接收参数
```
$.ajax{
    url:"xxxx",
    type : 'POST',
    dataType:"json",      
    contentType:"application/json", 
    data: JSON.stringify(["123", "456"]),
    ......
}
```

```
@RequestMapping("/getEventData")
public void getEventData(@RequestBody List<String> areaList) {
    // TODO
}
```
