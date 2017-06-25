---
layout:     post
title:      "SpringBoot集成Swagger2"
subtitle:   " \"SwaggerUI with springfox\""
date:       2017-06-25 00:00:00
author:     "hxwhou"
header-img: "img/in-post/20170625/01.png"
header-mask: 0.5
wechat-qrcode: "img/wechat-qrcode.jpg"
catalog: true
tags:
    - spring
    - swagger
    - API
---
>Swagger是目前世界上最受欢迎的API工具，是API开发者最常用来实现OpenAPI规范的框架，其使得开发贯穿整个API生命周期，从设计和文档化到测试与部署。通过Swagger提供的相关tools可以很直观的在网页上浏览、测试API。

最近在使用SpringBoot开发一个项目，刚开始的时候都是使用Doc文档来记录所有的API的接口定义（包括输入参数，参数类型，访问类型，输出信息等等），随着项目的不断深入，API数目越来越多，API的修改也越来越频繁（主要是API的前期设计不好，加上需求变化），这时候维护这个API文档是一件非常令人头痛的事情。并且，老大们想要将API文档实时的给客户（API使用方）看，这就要求API的修改必须实时的体现在API文档中。

基于上面的需求，我们决定将API文档整理到Swagger上，使用Swagger提供的各种工具生成在线的API文档直观的展示给用户。

## Swagger是什么？
Swagger是目前世界上最受欢迎的API工具，是API开发者最常用来实现OpenAPI规范的框架，其使得开发贯穿整个API生命周期，从设计和文档化到测试与部署。通过Swagger提供的相关tools可以很直观的在网页上浏览你的API、测试API、甚至通过解析Swagger Definitions（一个YAML or JSON格式的文档），直接自动生成相关代码（swagger-codegen）。
![img](/img/in-post/20170625/02.png)

* Swagger Editor：可让使用者在浏览器里以YAML格式编辑Swagger API规范并实时预览文档。可以生成有效的Swagger JSON描述，并用于所有Swagger工具（代码生成、文档等等）中。
* Swagger Codegen：一个模板驱动的引擎，通过解析用户的Swagger文档来生成不同语言的客户端代码。
* Swagger UI：一个无依赖的HTML、JS和CSS集合，可以通过解析定义好的Swagger Definitions构建对应的API用户界面。

## 为什么要用Swagger？
工具从来都是为了提高效率而产生的，想想从前光写个API文档都要花好些时间，而且其中还会出现理解上的偏差，需要前后端反复沟通确认，不仅浪费了 时间还没有什么效率。通过使用Swagger，我们既可以用它来作为API的文档，也可以用来测试API，前端 or 客户端的同学通过对各个API的点击可以实时的看到返回结果，节约了大量的沟通时间，极大的提高了开发效率。
## OpenAPI Specification是什么？
众所周知，OpenAPI Specification的目标是定义标准的、独立于语言的指向 REST API 的接口，使得服务能力无需访问源代码、文档，或是借助于网络流量检查，就可被人类和计算机发现并理解。通过对 OpenAPI 做适当定义后，消费者可使用最小数量的实现逻辑理解远程服务，并与远程服务交互。

OpenAPI 基于 Swagger 2.0 构建，Swagger 是 SmartBear 贡献给 Linux 基金会的。意在构建具有中立管理模型的新组织，以引领 Swagger 更上一层楼。

## Swagger与Java的集成
在swagger官网的open source integrations页面可以看到目前可以和swagger进行集成的各种语言。在Java语言集成这部分，可以发现与Spring MVC进行集成的swagger类库springfox。
![img](/img/in-post/20170625/03.png)
本文将主要介绍如何使用springfox将swagger2集成到使用Spring Boot框架开发的RESTful API中。

#### 添加swagger依赖包
在项目开发阶段，用到的构建工具是gradle，其配置如下：
![img](/img/in-post/20170625/04.png)
#### 添加Swagger配置类
![img](/img/in-post/20170625/05.png)![img](/img/in-post/20170625/06.png)
在Spring Boot项目中只需要增加如上的配置类就可以启动swagger服务。配置完成后，直接启动Spring Boot，访问http://localhost:8080/swagger-ui.html 就可以访问Swagger自动生成的UI界面。
![img](/img/in-post/20170625/07.png)
在定义REST API的时候，我们通常会在API的Request Header中加上token进行授权校验，防止非法用户访问。这时候我们就需要在swagger中配置授权，结合上面的SwaggerConfig类，我们在47行，48行分别引入了securitySchemes和securityContexts来增加API访问控制。对应的在SwaggerUI界面的右上角就会出现一个Authorize的button，点击这个按钮，就可以增加授权的token。在弹出来的Dialog中填入可用的token就完成了API调用的授权，这样当你访问API的时候相当于给每个请求的header中加入了key为token的Header。
![img](/img/in-post/20170625/08.png)
需要注意的是，因为我们在SwaggerConfig类中配置了相应的securityContexts并设置了匹配路径为/rest/.*，这样所有以/rest开头的API都会在被访问的时候进行授权验证，所以如上图可以看到此API的右侧出现了一个红色的带感叹号的圈，表示当前没有得到授权，当我们在上图的弹出框中输入了token value并获取到授权以后，上面的红圈将变成蓝色。
![img](/img/in-post/20170625/09.png)
Swagger会默认把所有Controller中的RequestMapping方法都提取出来显示在UI界面。如上所示，我们在BondController中定义了一个接口方法，其方法名为getBondDetail。这是Springfox类库根据Controller类和方法的声明自动帮我们生成的。通常情况下，我们需要自定义接口方法的说明，以及每个参数的具体描述，甚至在显示的时候控制哪些API隐藏掉。这就需要在相应的API接口方法上添加相应的注解来达到这些目的。

#### 增加Swagger注解
如下所示，我们在getBondDetail接口方法上增加了相应的注解，@ApiOperation用来描述类的接口方法，@ApiImplicitParams用来描述接口参数，可以对参数类型，参数的数据类型以及描述进行说明。参数类型paramType 有五个可选值 ，分别是path, query, body, header, form。
![img](/img/in-post/20170625/10.png)
![img](/img/in-post/20170625/11.png)
如果想隐藏某些API，可以使用注解@ApiIgnore来解决，如果应用在Controller范围上，则当前Controller中的所有方法都会被忽略，如果应用在方法上，则对应用的方法会被忽略。
swagger常用的5个注解：

```
@Api：修饰整个类，描述Controller的作用
@ApiOperation：描述一个类的一个方法，或者说一个接口
@ApiParam：单个参数描述
@ApiModel：用对象来接收参数
@ApiProperty：用对象接收参数时，描述对象的一个字段
```
#### 查看生成的JSON文件
上文我们说过，springfox会将Spring RESTful API提取出来生成UI页面，其实在此过程中会首先生成相应的JSON文件用来记录每个API的说明。这个JSON文件的存放路径即上图中显示的/v2/api-docs，访问这个路径会看到对应的内容。拷贝这个json文件，然后就可以通过Swagger UI对其内容进行显示，而不需要依赖源代码。
![img](/img/in-post/20170625/12.png)
至此，swagger与Spring Boot的集成告一段落！

