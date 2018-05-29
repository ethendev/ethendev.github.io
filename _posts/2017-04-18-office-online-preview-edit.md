---
layout: post
title:  "office web app实现文档的预览编辑"
categories: Java
tags: [Java, Office, 预览, 编辑]
keywords: Office,在线,预览,编辑
---

* content
{:toc}

最近项目中需要用到office文件在线编辑功能，然而很多解决方案都是收费的，于是决定采用微软免费的microsoft office online 2016和wopi 协议来实现。




### wopi 协议
>   WOPI的英文全称是“Web Application Open Platform Interface”，中文名为“Web应用程序开放平台接口协议”。WOPI协议提供一系列基于web方式的，使文档能在Office Web Apps中查看与编辑的接口服务（Web Service）。只要web application按照标准，实现了WOPI的接口，那么就可以调用Office Web Apps。例子很多，比如SharePoint，Exchange，SkyDriver，Dropbox集成Office Web Apps。  
> 
> 如果自己做的web应用也实现了相应接口，也是可以调用Office Web Apps。实现文档的在线编辑查看。在WOPI结构中，存放Office文档的web应用叫WOPI Host或者WOPI Server。把查看编辑操作Office文档的web应用叫WOPI Client或者叫WOPI applications。所以，Office Web Apps充当的就是WOPI Client的角色。SharePoint，Exchange，自己开发的文档管理系统充当的就是WOPI Host的角色。
> 

[Office开发团队对WOPI的介绍](http://blogs.msdn.com/b/officedevdocs/archive/2013/03/21/introducing-wopi.aspx).  
[Office Web Apps服务器概述](http://technet.microsoft.com/en-us/library/jj219437.aspx).  
请注意，WOPI主机必须响应OWA对内容的直接调用。 调用流程如下图所示：  
![](https://msdnshared.blob.core.windows.net/media/MSDNBlogsFS/prod.evol.blogs.msdn.com/CommunityServer.Blogs.Components.WeblogFiles/00/00/00/92/50/metablogapi/7128.Intro_WOPI_5C7408B7.jpg)



### 开发环境
office online的安装教材网上很多，这里就不再赘述了。安装好office online，然后按照下面的步骤进行wopihost的开发。我用的开发环境是jkd1.8，spring boot。

我们需要实现3个接口
GET    api/wopi/files/{name}?access_token={access_token}
GET    api/wopi/files/{name}/contents?access_token={access_token}     
POST  api/wopi/files/{name}/contents?access_token={access_token}

其中第一个接口获取文件的信息，返回的是json数据格式，第二个是获取文件流，第三个是保存修改文件。


### 接口实现
先看下第一个接口的实现：
```
@RequestMapping("/files/{name}")
@ResponseBody
public static Object checkFileInfo(@PathVariable(name = "name") String name, ServletRequest request, ServletResponse response) {
    HttpServletRequest httpRequest = (HttpServletRequest) request;
    HttpServletResponse httpResponse = (HttpServletResponse) response;
    String uri = httpRequest.getRequestURI();

    FileInfo info = new FileInfo();
    try {
        // 获取文件名
        String fileName = URLDecoder.decode(uri.substring(uri.indexOf("wopi/files/") + 11, uri.length()), "UTF-8");
        if (fileName != null && fileName.length() > 0) {
            String path = filePath + fileName;
            File file = new File(path);
            if (file.exists()) {
                // 取得文件名
                info.setBaseFileName(file.getName());
                info.setSize(file.length());
                info.setOwnerId("admin");
                info.setVersion(file.lastModified());
                info.setSha256(getHash256(file));
                info.setAllowExternalMarketplace(true);
                info.setUserCanWrite(true);
                info.setSupportsUpdate(true);
            }
        }
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    }
    return info;
}
```



然后是第二个接口的实现：

```
@RequestMapping(value="/files/{name}/contents", method= RequestMethod.GET)
public void getFile(@PathVariable(name = "name") String name, HttpServletResponse response) {
    try {
        // 文件的路径
        String path = filePath + name;
        File file = new File(path);
        // 取得文件名
        String filename = file.getName();
        String contentType = "application/octet-stream";
        // 以流的形式下载文件
        InputStream fis = new BufferedInputStream(new FileInputStream(path));
        byte[] buffer = new byte[fis.available()];
        fis.read(buffer);
        fis.close();
        // 清空response
        response.reset();

        // 设置response的Header
        response.addHeader("Content-Disposition", "attachment;filename=" + new String(filename.getBytes("utf-8"), "ISO-8859-1"));
        response.addHeader("Content-Length", "" + file.length());
        OutputStream toClient = new BufferedOutputStream(response.getOutputStream());
        response.setContentType(contentType);
        toClient.write(buffer);
        toClient.flush();
        toClient.close();
    } catch (IOException ex) {
        ex.printStackTrace();
    }
}    
```

保存文件修改的接口实现：

```
@RequestMapping(value="/files/{name}/contents", method= RequestMethod.POST)
public void postFile(@PathVariable(name = "name") String name, @RequestBody byte[] content) {
    // 文件的路径
    String path = filePath + name;
    File file = new File(path);

    try {
        if (!file.exists()) {
            file.createNewFile();//构建文件
        }
        FileOutputStream fop = new FileOutputStream(file);
        fop.write(content);
        fop.flush();
        fop.close();
        System.out.println("------------ save file ------------ ");
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```




### 接口访问

访问http://owas.contoso.com/hosting/discovery，owas.contoso.com是配置的office online的域名，当然也可以通过IP访问，请换成自己的地址，如图：
![这里写图片描述](http://img.blog.csdn.net/20170418114309314?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXVmZWl5YW5saXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在上面可以找到对应的文件类型的请求路径。然根据上面的URL+ WOPISrc=wopiHost的接口地址
就可以实现服务了。

例如word文档预览
http://[owas.domain]/wv/wordviewerframe.aspx?WOPISrc=http://[WopiHost.domain]:8080/wopi/files/test.docx&access_token=123456
![这里写图片描述](http://img.blog.csdn.net/20170418172425910?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXVmZWl5YW5saXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


word文档编辑
http://[owas.domain]/we/wordeditorframe.aspx?WOPISrc=http://[WopiHost.domain]:8080/wopi/files/test.docx&access_token=123456
![这里写图片描述](http://img.blog.csdn.net/20170418172534332?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXVmZWl5YW5saXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


> **注意：**web app上没有保存按钮，是自动保存的****。

---------
参考资料
>[部署 Office Online Server](https://msdn.microsoft.com/library/jj219455(v=office.16).aspx)  
>[WOPI Protocol Server Details](https://msdn.microsoft.com/en-us/library/hh643135(v=office.12).aspx)
>

Office Online Server下载地址
>[批量许可服务中心 (VLSC)](https://www.microsoft.com/Licensing/servicecenter/default.aspx)  
[备用下载地址](http://www.0daydown.com/10/630107.html)
>

代码已上传[Github](https://github.com/ethendev/wopihost)，有用的话记得start一下啊^__^。  