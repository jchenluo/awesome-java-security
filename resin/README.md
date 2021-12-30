**目录**

- 0x01-支持.jspf后缀
- 0x02-类IIS6.0的解析漏洞
- 0x03-Resin 4.0.36 信息泄露漏洞(ZSL-2013-5144)


### 0x01-支持.jspf后缀
配置文件

> E:\Resin\resin-4.0.65\conf\app-default.xml

![image](vulnerability-research.assets/144174160-82c02d3b-a775-4b71-acaf-d9f03f2b3653-164000593249077.png)

可见Resin不仅支持.jsp、.jspx，也支持.jspf。

```jsp
<%
    response.getWriter().write("Hello Resin !!!");
%>
```
![image](vulnerability-research.assets/144174179-d1e5af4c-c1cc-4f41-a5da-7fa2eb977b66-164000593053376.png)

### 0x02-类IIS6.0的解析漏洞

先看测试效果图

![image](vulnerability-research.assets/144174242-db437f8b-0feb-4683-8e46-7e7586905a15-164000592925175.png)

希望传达的意思

- 若文件夹名为`xxx.jsp`，其中放置的任意后缀的文件都将被当作JSP文件解析。

#### 1、为什么会这样？

分三步跟一下http请求的处理过程，来到关键函数下个断点

- com.caucho.server.dispatch.UrlMap#map

**第1步：jsp文件**

![image](vulnerability-research.assets/144174286-61ce59f9-da8f-47da-bb5a-60c65de85aab-164000592677374.png)

![image](vulnerability-research.assets/144174296-2f6a4527-c1bb-4199-b5b9-d108216991bc-164000592521673.png)

正常进入jsp的解析逻辑

![image](vulnerability-research.assets/144174317-03477b55-7f9c-4550-9e06-cb21fb4cd300-164000592371772.png)

**第2步：非jsp文件**

![image](vulnerability-research.assets/144174351-15c3b0f6-df52-4c02-9322-bb0f76a3b2bf-164000592174971.png)

![image](vulnerability-research.assets/144174357-ba30fda0-d499-4929-8234-f0778f09039b-164000592062570.png)

进入resin-file的处理逻辑

![image](vulnerability-research.assets/144174378-bf20140b-fedf-4507-bef2-445187820ab2-164000591919169.png)

处理结果

![image](vulnerability-research.assets/144174406-2259125d-b101-4073-94d5-01b8f9d67d96-164000591779667.png)

**第3步：x.jsp文件夹 + 非.jsp文件**

![image](vulnerability-research.assets/144174432-3c2e4d49-7cc2-48ae-928e-60c9af933411-164000591588465.png)

![image](vulnerability-research.assets/144174451-3cd87542-0dad-41de-ad7f-48a9359d8ef2-164000591406663.png)

![image](vulnerability-research.assets/144174460-5f803d3c-8b6f-42e6-9f81-4def07970343-164000591286061.png)

也进入resin-file的处理逻辑

![image](vulnerability-research.assets/144174477-b242ffb6-6d62-442c-98a7-ea6a7cb11206-164000591148059.png)

#### 2、造成这种处理差异的原理是什么？

![image](vulnerability-research.assets/144174511-0cdabaf9-33c1-4c6e-aca5-c27c4ade0801-164000590875157.png)

map方法将会对url路径进行正则表达式，然后根据匹配结果进入不同的处理逻辑

> /hello.jsp

![image](vulnerability-research.assets/144174547-64dc2dba-d06b-4591-8f01-3ad408648d96-164000590703855.png)

> /hello.hello

![image](vulnerability-research.assets/144174573-43a536d0-d35f-40e2-8ecd-0b79f1d66723-164000590498553.png)

> /x.jsp/hello.hello

![image](vulnerability-research.assets/144174584-858aca20-2946-4f46-808d-7da2c1b733ad-164000590275651.png)


### 0x03 Resin 4.0.36 信息泄露漏洞(ZSL-2013-5144)

- https://www.zeroscience.mk/en/vulnerabilities/ZSL-2013-5144.php

测试效果
> 读取index.jsp

![image](vulnerability-research.assets/144178194-d2717d65-d9ed-4f3c-8903-4f4a624d848f-164000590059149.png)

> 读取resin-admin.xml

![image](vulnerability-research.assets/144181449-d6b81379-429e-49a0-b02a-72c5c860b6d2-164000589886547.png)


#### 漏洞分析

从上面的分析中知道了可以从com.caucho.server.dispatch.UrlMap观察resin对http请求的处理逻辑，下断点调试

![image](vulnerability-research.assets/144178592-1ee0f23f-5b67-4cd7-8dc4-a0437cb67168-164000589598845.png)

一路跟到`ServletMapping`

![image](vulnerability-research.assets/144178671-718bf816-6494-4676-a40f-3b46d9f10c74-164000589357743.png)

很明显，到这里应该就知道漏洞成因估计是该版本的resin-web.xml默认添加了路由为/viewfile/*的servlet

文件位置
> E:\Resin\resin-pro-4.0.36\doc\resin-doc\WEB-INF\resin-web.xml

![image](vulnerability-research.assets/144179072-662fff09-1c54-4ee2-a25b-923a542aaf40-164000589084541.png)

跟进对应的类
- com.caucho.doc.ViewFileServlet

![image](vulnerability-research.assets/144179200-719d6a33-731d-402d-9907-cc15ea2ca4bf-164000587496637.png)


继续断点

![image](vulnerability-research.assets/144179705-96c69246-482e-43d3-8d96-b3181bc2c07c-164000587935539.png)

然后通过viewFile打印文件内容

![image](vulnerability-research.assets/144183728-c145ad4b-eca7-4ee1-866c-e6c039910117.png)


![image](vulnerability-research.assets/144183814-9994ff06-4e7a-458b-92c1-c881e1834c82.png)