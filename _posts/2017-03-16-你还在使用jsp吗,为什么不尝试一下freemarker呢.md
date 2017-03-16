---
layout: post
title:  你还在使用jsp吗,为什么不尝试一下FreeMarker呢
date:   2017-03-16 10:48:00 +0800
tags: 能工巧匠集
---

目前国内java web项目开发，常用的页面模板引擎是jsp+jstl、FreeMarker和velocity。关于谁优谁劣的问题网上讨论的也比较多。



### 支持使用FreeMarker模板的开发者大多给出以下几条理由：

#### 1、类加载没有PermGen问题:
如果你已经开发Java Web应用程序一段时间，那么对于 JVM 的 PermGen 问题可能并不陌生。由于 FreeMarker 模板不编译成类，它们不占用 PermGen 空间，并不需要一个新的类加载器加载。

#### 2、模板加载器:
直接从数据源加载页面和模板岂不是很好？也许从数据库。也许你只想把它们放在一个地方，可以不重新部署整个应用程序就能更新它们。那么在 JSP 中你是很难做到这一点的，但 FreeMarker 提供的模板加载器就是为了 这个目的。你可以使用内建类或者创建你自己的实现。

#### 3、可以在运行时嵌入模板:
FreeMarker 能让你创建真正的模板，而不只是片段 ，这些是jsp做不到的,你见过header.jsp 和 footer.jsp吗? FreeMarker 允许你使用一个模板（在本例中为 head.ftl）
```
<head>
<title>${title}</title>
</head>
```
并将其添加到另一个模板（site.ftl body区域）。
```
<html>
${body}
</html>
```

#### 4、没有导入:
JSP 要求你导入每个你需要使用的类，就像一个常规的 Java 类一样。FreeMarker 模板，嗯，仅仅是模板。可以被包括在另一个模板中，但目前还不需要导入类。

#### 5、支持jsp标签:
使用 Jsp 的一个理由是有可用性很好的标签库。好消息是 FreeMarker 支持 JSP 标签。但是它们使用 FreeMarker 的语法，不是 JSP 语法。

#### 6、内置空字符串处理:
FreeMarker 和 Jsp 都可以在表达式语言中处理空值，但 FreeMarker 在可用性上更先进一些。

请参见[处理缺少的值](http://freemarker.org/docs/dgui_template_exp.html)了解更多细节。

#### 7、共享变量:
FreeMarker 的共享变量是我最喜欢的“隐藏”功能之一。此功能可以让你设置自动添加到所有模板的值。 例如，可以设置应用程序的名称作为共享变量。
```
Configuration configuration = new Configuration();
configuration.setSharedVariable("app", "StackHunter");
```
然后像任何其他变量一样访问它。
```
App: ${app}
```
在过去使用共享变量一般引用资源包 然后使用像 ${i18n.resourceBundle.key} 这样的表达式来获取值。
```
${i18n.countries.CA}
${i18n.countries['CA']}
${i18n.countries[countryCode]}
```
上面这些行都引用 countries_en.properties 资源包内的 key “CA”对应的值。你需要执行自己的 TemplateHashModel，然后将其添加为一个共享变量来实现这一目标。

#### 8、支持JSON:
FreeMarker 内置 JSON 支持。 比方说你有以下的 JSON 存储到变量命名 user 的字符串中。
```
{ 'firstName': 'John', 'lastName': 'Smith', 'age': 25, 'address': { 'streetAddress': '21 2nd Street', 'city': 'New York', 'state': 'NY', 'postalCode': 10021 }}
```
使用 ?eval 将从字符串转换为一个 JSON 对象，然后像其他数据一样在表达式中使用。
```
<#assign user = user?eval>
User: ${user.firstName}, ${user.address.city}
```

#### 9、可以生产电子邮件，XML映射等:
与 JSP 不同的是FreeMarker 模板可以在 servlet 容器之外使用。可以使用它们来生成电子邮件、 配置文件、 XML 映射等。你甚至可以使用它们来生成 web 页 并将它们保存在服务器端的缓存中。

### 但是
个人感觉这些都是无数据的空谈，或者重点比较语法、标签特性，但是作为业务开发而言，真正需要关心的是谁最简单、最高效。

---

下面构建一个基于spring mvc的测试demo，比较（jsp/freemarker/velocity）的性能。
#### 前置条件 测试服务器的配置及环境清单：
```
CentOS release 6.5 (Final)
Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz
Java(TM) SE Runtime Environment (build 1.8.0_66-b17)
Apache Tomcat/8.0.32
```

#### 测试模板的版本清单：
```
<dependency>
	<groupId>jstl</groupId>
	<artifactId>jstl</artifactId>
	<version>1.2</version>
</dependency>
<dependency>
	<groupId>org.freemarker</groupId>
	<artifactId>freemarker</artifactId>
	<version>2.3.23</version>
</dependency>
<dependency>
	<groupId>org.apache.velocity</groupId>
	<artifactId>velocity</artifactId>
	<version>1.7</version>
</dependency>
```
核心代码清单 测试过程是模拟一个简单的SpringMVC的web项目，后台随机的生成10个对象的集合，前端页面将这10个对象展示在表格里面，会使用的循环、流程控制和格式化标签，工程同时支持jsp、freemarker和velocity共3种视图解析。

#### SpringMVC配置多视图解析配置：
```
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
	<property name="prefix" value="/WEB-INF/jsp" />
	<property name="suffix" value="" />
	<property name="viewNames" value="*.jsp" />
</bean>

<bean class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
	<property name="templateLoaderPath" value="/WEB-INF/jsp" />
	<property name="defaultEncoding" value="UTF-8" />
</bean>
<bean class="org.springframework.web.servlet.view.freemarker.FreeMarkerViewResolver">
	<property name="prefix" value="" />
	<property name="suffix" value="" />
	<property name="viewNames" value="*.ftl" />
	<property name="contentType" value="text/html;charset=UTF-8"></property>
</bean>

<bean class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
	<property name="resourceLoaderPath" value="/WEB-INF/jsp" />
	<property name="velocityProperties">
		<props>
			<prop key="input.encoding">UTF-8</prop>
			<prop key="output.encoding">UTF-8</prop>
		</props>
	</property>
</bean>
<bean class="org.springframework.web.servlet.view.velocity.VelocityViewResolver">
	<property name="prefix" value="" />
	<property name="suffix" value="" />
	<property name="viewNames" value="*.vm" />
	<property name="contentType" value="text/html;charset=UTF-8"></property>
</bean>
控制器代码主要是随机生产10个对象的集合，返回给前端页面，代码清单如下：

@Controller
@RequestMapping("/template")
public class TemplateController {
	
	@RequestMapping("/{val}")
	public String bench(@PathVariable(value = "val") String val, Model model) {
		Random ra =new Random();
		List<UserInfo> list = Lists.newArrayList();
		for(int i=0;i<10;i++)
		{
			UserInfo userInfo = new UserInfo();
			userInfo.setId(ra.nextInt(10));
			userInfo.setSex(ra.nextInt(2));
			userInfo.setName("maxchen_"+userInfo.getId());
			userInfo.setEmail(userInfo.getName()+"@mail.com");
			userInfo.setBirthday(new Date());
			userInfo.setRegister(new Date());
			list.add(userInfo);
		}
		model.addAttribute("list", list);
		model.addAttribute("df", new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
		return "/template."+val;
	}
}
```
模板页面负责循环读取控制器生产的集合，将10个对象以表格的形式呈现出来，其中会用到循环、流程控制和格式化标签，这个页面属于繁简适中，很符合实际业务场景。

#### template.jsp代码清单如下：
```
<html lang="zh-CN">
<body>
<table>
	<c:forEach items="${list}" var="e">
		<tr>
		 	<td>${e.id}</td>
		 	<td>${e.name}</td>
		 	<td><c:if test="${e.sex==0}">男</c:if><c:if test="${e.sex==1}">女</c:if></td>
		 	<td>${e.email}</td>
		 	<td><fmt:formatDate value="${e.birthday}" pattern="yyyy-MM-dd"/></td>
		 	<td><fmt:formatDate value="${e.register}" pattern="yyyy-MM-dd HH:mm:ss"/></td>
		</tr>
	</c:forEach>
</table>
</body>
</html>
```

#### template.ftl代码清单如下：
```
<html lang="zh-CN">
<body>
<table>
	<#list list as e>
		<tr>
		 	<td>${e.id}</td>
		 	<td>${e.name}</td>
		 	<td><#if e.sex==0>男<#else>女</#if></td>
		 	<td>${e.email}</td>
		 	<td>${e.birthday?string("yyyy-MM-dd")}</td>
		 	<td>${e.register?string("yyyy-MM-dd HH:mm:ss")}</td>
		</tr>
	</#list>
</table>
</body>
</html>
```

#### template.vm代码清单如下：
```
<html lang="zh-CN">
<body>
<table>
	<#list list as e>
		<tr>
		 	<td>${e.id}</td>
		 	<td>${e.name}</td>
		 	<td><#if e.sex==0>男<#else>女</#if></td>
		 	<td>${e.email}</td>
		 	<td>${e.birthday?string("yyyy-MM-dd")}</td>
		 	<td>${e.register?string("yyyy-MM-dd HH:mm:ss")}</td>
		</tr>
	</#list>
</table>
</body>
</html>
```

#### 测试过程及结果 先进行预热，手工访问若干次测试url后，对每个视图解析地址使用ab并发100个线程共10000个请求，连续10次进行测试取平均值：
```
# ab -c 100 -n 10000 http://192.168.56.2:8080/demo-webapp/template/jsp
# ab -c 100 -n 10000 http://192.168.56.2:8080/demo-webapp/template/ftl
# ab -c 100 -n 10000 http://192.168.56.2:8080/demo-webapp/template/vm
```

#### 最终测试耗时结果如下 模板|耗时(秒) —|— jsp+jstl|1.62 freemarker|0.737 velocity|0.729
#### 对测试结果的分析和结论：
- freemarker和velocity的性能基本持平，jsp+jstl相对落后，具体到每个请求差距约9ms。
- 9ms相对于业务的处理和网络传输的耗时而言，具体占比需要根据实际情况核算；如果占比较大可以考虑废弃jsp，否则模板引擎不是影响系统性能的关键因素。
- velocity的格式化使用体验不是很好，且该项目自2010年起已未更新，不推荐使用。

[此段测试引用了这位大神的文章](http://chenyibo.org/template-compare)

---

想要知道为什么FreeMarker性能比jsp好，就需要探究一下FreeMarker以及jsp的工作原理。



FreeMarker是一个基于Java的开发包和类库的一种将模板和数据进行整合并输出文本的通用工具，Freemarker本质上是实现页面静态化，其原理是将页面所需要的样式写入到FreeMarker模板文件中，然后将页面所需要的数据进行动态绑定并放入到Map中，然后通过FreeMarker的模板解析类process()方法完成静态页面的生成。如图：

![](/assets/images/2017/freemarker_work.jpg)


jsp作为一种动态页面加载技术，其交互性非常良好，但是为达到交互就需要每打开一个页面就需要服务端进行一次处理，如果访问人数很多，也就会对服务器增加很大的负荷，从而影响网站运行速度。

通过上面的技术比较，不难看出jsp的性能插在哪里，就是以牺牲性能的代价来达到良好的交互体验。而目前，我们通常提升网站性能的方法通常有HTML静态化、图片服务器分离、数据库集群、负载均衡等。其中HTML静态化也就是，我今天所表达的FreeMarker的性能优势。

最后，如果你需要建设一个大数据，高并发的网站，那么请考虑在你的下个web应用中尝试使用FreeMarker。
