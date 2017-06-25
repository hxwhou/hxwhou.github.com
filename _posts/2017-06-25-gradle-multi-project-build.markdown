---
layout:     post
title:      "Gradle构建多模块项目"
subtitle:   " \"Multi project build with gradle\""
date:       2017-06-25 15:00:00
author:     "hxwhou"
header-img: "img/post-bg-gradle.png"
header-mask: 0.2
wechat-qrcode: "img/wechat-qrcode.jpg"
catalog: true
tags:
    - spring
    - gradle
    - 构建
---

最近在公司开发一个大型项目，整个项目切分成好几个子项目进行开发，某些项目之间又需要用到相同的服务（即复用一些common的代码）。但是整个项目的用户群又不是特别大，所以在技术选型的时候也就没有考虑用微服务这种高大上的东西，而是采用了更加平民更加符合当前需求的方式，即通过Gradle构建多模块项目来达到代码复用。

## 前言
既然是通过构建的方式来复用代码，就不得不提一下java世界中的构建工具。使用Java的人应该都知道Ant和Maven这两种项目构建工具，Ant是早期兴起的构建工具，通过XML文件编写构建脚本，定义构建的target，从而将软件编译、测试、部署等步骤联系在一起加以自动化。在Maven出现之前，Ant一直是java开发者必备的构建工具。然而，随着项目变得越来越大，框架也越来越多，项目与项目之间的依赖变得越来越复杂，项目中jar包的依赖冲突越来越频繁，这时候就急需一个新的构建来解决这个问题。这时候一个叫Maven的项目管理与构建工具应运而生了，其凭借优秀的依赖管理方式，COC（Convention Over Configuration）已经项目构建规则重用性等特点，使其迅速在java社区流行起来，并成为Java构建工具的事实标准。

就这样，Maven在Java社区大红大紫，一直火了很多年。然而，冗余的依赖管理配置、复杂并且难以扩展的构建生命周期，都成为Maven的软肋。厌倦了maven的繁杂后，开发者将目光转移到了下一代构建工具Gradle上。

Gradle作为新的构建工具，获得了2010 Springy大奖，并入围了2011的Jax最佳Java技术发明奖。它是基于Groovy语言的构建工具，既保持了Maven的优点，又通过使用Groovy定义的DSL，克服了Maven中使用XML繁冗以及不灵活等缺点。在Micro-Service架构风格流行的今天，在一个项目里面包含多个Module已成为一种趋势。Gradle天然支持多module，并且提供了很多手段来简化构建脚本。

开始正文之前，还不清楚Gradle是什么东西的亲自行解决！

好了，废话不多说了，开始进入正题👉

## 多模块项目结构
在多模块的项目中，Gradle遵循惯例优于配置 （Convention Over Configuration）原则。项目中的某些模块需要依赖于其他模块，Gradle为每个模块都创建了一个Project对象，并且可以通过模块的名字引用到该对象。在配置模块之间的依赖时，采用引用Project对象这种方式可以告诉Gradle当前模块依赖了哪些子模块。

在父项目的根目录下寻找settings.gradle文件，在该文件中设置想要包括到项目构建中的子项目。在构建的初始化阶段（Initialization），Gradle会根据settings.gradle文件来判断有哪些子项目被include到了构建中，并为每一个子项目初始化一个Project对象，在构建脚本中通过project(':sub-project-name')来引用子项目对应的Project对象。

通常，多模块项目的目录结构要求将子模块放在父项目的根目录下，但是如果有特殊的目录结构，可以在settings.gradle文件中配置。

## 初窥gradle多模块
如果你使用的是Eclipse IDE工具，并且安装了Gradle插件，那么可以很方便的新建一个gradle多模块的Demo项目，其过程如下：
![img](/img/in-post/20170625/gradle/01.png)

#### settings.gradle文件
新建完成后会发现生成了三个project，DemoRoot就是这里的父项目，其下有一个settings.gradle文件，内容如下所示：
![img](/img/in-post/20170625/gradle/02.png)
可以看到其包括两个子模块，分别为"my-lib"和"product"。my-lib模块中是一些共用的类，product模块会依赖这个模块中声明的类。
![img](/img/in-post/20170625/gradle/03.png)

#### build.gradle文件
父项目的build.gradle脚本里，给subprojects传一个包含配置信息的闭包，可以配置所有子项目共有的设置，比如共同的插件、repositories、依赖版本以及依赖配置等，这里的DemoRoot中的具体内容如下：
```
subprojects {
    apply plugin: 'java'
    apply plugin: 'eclipse'

    repositories {
       mavenCentral()
    }

    dependencies {
        testCompile 'junit:junit:4.8.2'
    }

    version = '1.0'

    jar {
        manifest.attributes provider: 'my cool company'
    }
}
```
这就会给所有子项目设置上java的插件、使用mavenCentral作为所有子项目的repository以及对JUnit的项目依赖。通常还会在此文件中配置allprojects和configure。
allprojects是父Project的一个属性，该属性会返回该Project对象以及其所有子项目。在父项目的build.gradle脚本里，可以通过给allprojects传一个包含配置信息的闭包，来配置所有项目（包括父项目）的共同设置。通常可以在这里配置IDE的插件，group和version等信息，比如：
```
allprojects {
    repositories {
		mavenCentral()
	}
    group = 'com.ibm.isg'
}
```
这样就会给所有项目（当前项目以及所有子项目）设置mavenCentral仓库以及group信息。注意，在allprojects中配置的信息就没必要同时在subprojects配置了，如上面的repositories信息。
在某些情况下可能会用到configure这个配置。在项目中，并不是所有的子项目都会具有相同的配置，但是会有部分子项目具有相同的配置，比如有的项目里即有web项目，又有standalone的项目。这时候就可以给web项目单独添加war插件。
```
configure(subprojects.findAll {it.name.contains('war')}) {
    apply plugin: 'war'
}
```
configure需要传入一个Project对象的数组，通过查找所有项目名包含war的子项目，并为其设置war插件。
此时，整个项目的目录结构如下所示：
![img](/img/in-post/20170625/gradle/04.png)
至此，我们已经初步认识了一个gradle构建的多模块项目，其结构简单，非常适合理解多模块的概念。我也将其上传到了我的[github仓库](https://github.com/hxwhou/DemoRoot)来供你学习。接下来，我们会用一个实际的例子来阐述多模块项目的构建过程。

## 实际应用
在上文中，我们简单的了解了一个gradle多模块项目的结构与配置。然而实际的多模块项目远比这个复杂，本文就将介绍使用gradle构建多模块的SpringBoot项目。还不明白SpringBoot的同学赶紧上网了解一下吧。
这个项目主要包括以下子模块:

* 1）一个描述核心业务的core模块
* 2）一个用于存放项目配置文件的config模块
* 3）一个提供工具类的tools模块
* 4）两个提供不同服务的web项目（cas-subA和cas-subB)

#### 创建项目
在Eclipse中首先创建一个根项目，项目名为cas，其就是一个普通的gradle project，然后如图所示依次创建其下的五个子project：
![img](/img/in-post/20170625/gradle/05.png)
创建完成后的工程目录结构如下：
![img](/img/in-post/20170625/gradle/06.png)
此时我们查看其在本地磁盘的目录结构如下：
![img](/img/in-post/20170625/gradle/07.png)
#### 配置子模块
在cas根项目下，新建一个settings.gradle文件，将需要加入到项目构建中的子项目配置在这个文件中，其内容如下：
![img](/img/in-post/20170625/gradle/08.png)
#### 共享配置
在大型Java项目中，子项目之间必然具有相同的配置项。我们在编写代码时，要追求代码重用和代码整洁；而在编写Gradle脚本时，同样需要保持代码重用和代码整洁。Gradle提供了不同的方式使不同的项目能够共享配置。打开cas下的build.gradle文件，加入如下内容：
```
apply from: 'dependencyDefinitions.gradle'

buildscript {
	repositories {
	    mavenCentral()
	}
	dependencies {
		classpath("org.springframework.boot:spring-boot-gradle-plugin:1.4.0.RELEASE")
	}
}

allprojects {
    repositories {
        mavenCentral()
    }
    group = 'com.ibm.cas'
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'eclipse'
    apply plugin: 'spring-boot'

    sourceCompatibility = JavaVersion.VERSION_1_8
    [compileJava, compileTestJava]*.options*.encoding = 'UTF-8'

    configurations {
        all*.exclude module: 'spring-boot-starter-logging'
    }

    dependencies {
        testCompile(
            libraries.'spring-boot-starter-test'
        )
    }
}
```
在上面的配置中，可以看到其中有一个buildscript的配置项，其中配置了Maven仓库和依赖的spring-boot-gradle-plugin，同样的Maven仓库配置在allprojects下也有，这是为什么呢？其实道理很简单，buildscript中的声明是gradle脚本自身需要使用的资源。可以声明的资源包括依赖项、第三方插件、maven仓库地址等。而在build.gradle文件中直接声明的依赖项、仓库地址等信息是项目自身需要的资源。因为我们的project是springboot工程，所以引入了spring-boot-gradle-plugin的插件用来构建多模块。allprojects和subprojects的区别在上文中已经阐述过，故这里不再赘述。另外，我们引入了一个dependencyDefinitions.gradle文件，这个文件中会放置整个项目要用到的依赖包。这个很好理解，项目大了以后，依赖的jar包会非常多，为了便于管理和维护，统一放到一个地方进行声明，其他子项目要适应的话直接从此处引用即可，这样既可以保证jar包版本的一致性，又方便维护。dependencyDefinitions.gradle中的内容如下：
```
/**
 * 依赖包的定义。
 *
 * 这种定义方式的优点是在顶级项目目录下引入，在子项目中也可以直接用了。
 *
 */
ext.versions = [
        springBoot        : '1.4.0.RELEASE',
        springBootMybaits : '1.1.1',
        jedis             : '2.9.0',
        commonsLang3      : '3.3.2',
        commonsIo         : '2.4',
        commonsCodec      : '1.10',
        mysql             : '5.1.34',
        dbcp              : '1.4'
]

// 各种可能会用到的jar包
ext.libraries = [
        // Spring Boot
        "spring-boot-starter"                : "org.springframework.boot:spring-boot-starter:${versions.springBoot}",
        "spring-boot-starter-web"            : "org.springframework.boot:spring-boot-starter-web:${versions.springBoot}",
        "spring-boot-starter-aop"            : "org.springframework.boot:spring-boot-starter-aop:${versions.springBoot}",
        "spring-boot-starter-log4j2"         : "org.springframework.boot:spring-boot-starter-log4j2:${versions.springBoot}",
        "spring-boot-mybaits"                : "org.mybatis.spring.boot:mybatis-spring-boot-starter:${versions.springBootMybaits}",
        "spring-boot-starter-test"           : "org.springframework.boot:spring-boot-starter-test:${versions.springBoot}",
        "spring-boot-starter-tomcat"         : "org.springframework.boot:spring-boot-starter-tomcat:${versions.springBoot}",
        "spring-boot-starter-jdbc"           : "org.springframework.boot:spring-boot-starter-jdbc:${versions.springBoot}",
        "spring-boot-starter-data-rest"      : "org.springframework.boot:spring-boot-starter-data-rest:${versions.springBoot}",
        // Commons
        "commons-io"                         : "commons-io:commons-io:${versions.commonsIo}",
        "commons-lang3"                      : "org.apache.commons:commons-lang3:${versions.commonsLang3}",
        "commons-codec"                      : "commons-codec:commons-codec:${versions.commonsCodec}",
        // Cache
        "jedis"                              : "redis.clients:jedis:${versions.jedis}",
        // DB
        "mysql-driven"                       : "mysql:mysql-connector-java:${versions.mysql}",
        "dbcp"                               : "commons-dbcp:commons-dbcp:${versions.dbcp}"
]
```
至此，共享配置部分已经完成，接下来会配置每个子模块的私有配置。

#### 私有配置
在项目中，除了设置共同配置之外， 每个子项目还会有其独有的配置。比如每个子项目具有不同的依赖以及每个子项目特殊的task等。Gradle提供了两种方式来分别为每个子项目设置独有的配置。
在父项目的build.gradle文件中通过project(':sub-project-name')来设置对应的子项目的配置。比如在子项目cas-config需要spring-boot-starter的依赖，可以在父项目的build.gradle文件中添加如下的配置：
```
project(':cas-config') {
    dependencies { 
        compile(
            libraries.'spring-boot-starter'
        )
    }
}
```
注意这里子项目名字前面有一个冒号（:）。 通过这种方式，指定对应的子项目，并对其进行配置。

我们还可以在每个子项目的目录里建立自己的构建脚本。在上例中，可以在子项目cas-config目录下为其建立一个build.gradle文件，并在该构建脚本中配置cas-config子项目所需的所有配置。例如，在该build.gradle文件中添加如下配置：
![img](/img/in-post/20170625/gradle/09.png)
对于子项目少，配置简单的小型项目，推荐使用第一种方式配置，这样就可以把所有的配置信息放在同一个build.gradle文件里。但是，若是对于子项目多，并且配置复杂的大型项目，使用第二种方式对项目进行配置会更好。因为，第二种配置方式将各个项目的配置分别放到单独的build.gradle文件中去，可以方便设置和管理每个子项目的配置信息。
#### cas-config模块
在cas-config模块中主要放置一些共用的属性配置文件，如log4j2.xml和application.properties.在application.properties中配置datasource信息如下：
```
# Spring Production Profile 
spring.profiles.active=production

spring.datasource.url=jdbc:mysql://127.0.0.1:3306/test
spring.datasource.username=root
spring.datasource.password=passw0rd
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

spring.datasource.max-idle=10
spring.datasource.max-wait=10000
spring.datasource.min-idle=5
spring.datasource.initial-size=5
spring.datasource.validation-query=SELECT 1
spring.datasource.test-on-borrow=false
spring.datasource.test-while-idle=true
spring.datasource.time-between-eviction-runs-millis=18800
spring.datasource.jdbc-interceptors=ConnectionState;SlowQueryReport(threshold=10)

```
到这里cas-config基本就配置完成了，具体的代码可以从我的[github仓库](https://github.com/hxwhou/cas)下载。
#### cas-tools模块
在cas-tools子模块中，我们提供了一些工具类的定义，如记录log的LoggerContext工具类，用于声明异常的CASSystemException类，以及一些其它的工具类，具体源码请参考[github仓库](https://github.com/hxwhou/cas)。

#### cas-core模块
在cas-core模块中，我们定义了一些common的Entity，DAO和Service类，用来处理业务逻辑并进行数据的CRUD操作。这部分的代码可能会被用在不同的Web或是客户端项目中，并且其逻辑基本上都是固定的，结构也是经典的SpringMVC式的。具体实现逻辑参考github仓库上的code。在此模块中，会依赖之前完成的两个子模块，除此之外，这个模块要依赖一些私有的jar包，所以在此模块中需要单独引入进来，其具体配置如下：
```
archivesBaseName = 'cas-core'

dependencies {

    compile(
        libraries.'spring-boot-starter-data-jpa',
        libraries.'spring-boot-starter-jdbc',
        libraries.'spring-boot-starter-data-rest',
        libraries.'mysql-driven',
        project(':cas-config'),
        project(':cas-tools')
    )
	
	compile(fileTree(dir:'libs', include:'**/*.jar'))
}

```
#### cas-subA和cas-subB模块
这两个模块都是Web项目，实现了Restful风格的API，其代码逻辑可以参考github上的源码。这里因为是Web项目，在实际的使用过程中，可能就需要将其打成war包部署到server上运行，这时候就需要添加war插件，如cas-subA模块的具体配置如下：
```
archivesBaseName = 'cas-subA'

apply plugin: 'war'

war {
	baseName = 'cas-subA'
}

dependencies {

    compile(
        project(':cas-core')
    )
    
    compile(libraries.'spring-boot-starter-web'){
		exclude module : 'org.springframework.boot:spring-boot-starter-tomcat'
	}
	
	providedRuntime(libraries.'spring-boot-starter-tomcat')
}
```
可以看到我们在这里引入了war插件，并且引入了对cas-core模块的引用，需要注意的是，模块的依赖也是有传递性的，虽然我们在此仅仅引入cas-core模块，但就相当于同时引入了cas-core所依赖的模块，所以可以在此直接使用cas-config和cas-tools中的内容。
至此，整个项目多模块的配置及编码就算完成了，下面开始验证其效果。
## 验证多模块效果
分别运行cas-subA和cas-subB模块下CasSubAApplication和CasSubBApplication中的main方法，等服务启动以后，访问相应的REST API查看效果。
打开浏览器访问如下地址：
http://localhost:8080/rest/cas/v1/greet/sayhello?name=hxwhou
http://localhost:8081/rest/cas/v1/score/list
可以看到API返回的数据，这就证明整个cas多模块项目是可以正常运行的。
![img](/img/in-post/20170625/gradle/10.png)

～over

***完整的示例[下载](https://github.com/hxwhou/cas)*** 