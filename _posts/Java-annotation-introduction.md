---
title: Java注解简介
date: 2016-06-18 16:35:45
tags: [Java]
categories: Java
link_title: java-annotation-introduction
thumbnailImage: https://i.loli.net/2019/09/24/4kPOL3i5aAqcfNs.png
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://i.loli.net/2019/09/24/4kPOL3i5aAqcfNs.png)
> 有必要对JDK 5.0新增的注解（Annotation）技术进行简单的学习，因为spring 支持@AspectJ，而@AspectJ本身就是基于JDK 5.0的注解技术。所以学习JDK 5.0的注解知识有助于我们更好地理解和掌握Spring的AOP技术。

# 了解注解

对于Java开发人员来说，在编写代码时，除了源程序以外，我们还会使用Javadoc标签对类、方法或成员变量进行注释，以便使用Javadoc工具生成和源代码配套的Javadoc文档。这些@param、@return等Javadoc标签就是注解标签，它们为第三方工具提供了描述程序代码的注释信息。使用过Xdoclet的朋友，对此将更有感触，像Struts、hibernate都提供了Xdoclet标签，使用它们可以快速地生成对应程序代码的配置文件。



JDK5.0注解可以看成是Javadoc标签和Xdoclet标签的延伸和发展。在JDK5.0中，我们可以自定义这些标签，并通过Java语言的反射机制中获取类中标注的注解，完成特定的功能。 
注解是代码的附属信息，它遵循一个基本原则：注解不能直接干扰程序代码的运行，无论增加或删除注解，代码都能够正常运行。Java语言解释器会忽略这些注解，而由第三方工具负责对注解进行处理。第三方工具可以利用代码中的注解间接控制程序代码的运行，它们通过Java反射机制读取注解的信息，并根据这些信息更改目标程序的逻辑，而这正是Spring AOP对@AspectJ提供支持所采取的方法。

> 很多东西的设计都必须遵循最基本的原则，为了防止机器人伤害人类，科幻作家阿西莫夫于1940年提出了“机器人三原则”：第一，机器人不能伤害人类；第二，机器人应遵守人类的命令，与第一条违背的命令除外；第三，机器人应能保护自己，与第一条违背的命令除外。这是给机器人赋予的伦理性纲领，机器人学术界一直将这三条原则作为机器人开发的准则。

# 一个简单的注解类
通常情况下，第三方工具不但负责处理特定的注解，本身还提供了这些注解的定义，所以我们通常仅需关注如何使用注解就可以了。但定义注解类本身并不困难，Java提供了定义注解的语法。下面，我们马上着手编写一个简单的注解类，如代码清单7-1所示：

代码清单7-1 NeedTest注解类

```java
package com.baobaotao.aspectj.anno;  
import java.lang.annotation.ElementType;  
import java.lang.annotation.Retention;  
import java.lang.annotation.RetentionPolicy;  
import java.lang.annotation.Target;  

@Retention(RetentionPolicy.RUNTIME) //①声明注解的保留期限  
@Target(ElementType.METHOD)//②声明可以使用该注解的目标类型  
public @interface NeedTest {//③定义注解  
    boolean value() default true;//④声明注解成员  
}  
```
Java新语法规定使用@interface修饰符定义注解类，如③所示，一个注解可以拥有多个成员，成员声明和接口方法声明类似，这里，我们仅定义了一个成员，如④所示。成员的声明有以下几点限制：

成员以无入参无抛出异常的方式声明，如boolean value(String str)、boolean value() throws Exception等方式是非法的； 
可以通过default为成员指定一个默认值，如String level() default “LOW_LEVEL”、int high() default 2是合法的，当然也可以不指定默认值； 
成员类型是受限的，合法的类型包括原始类型及其封装类、String、Class、enums、注解类型，以及上述类型的数组类型。如ForumService value()、List foo()是非法的。

在①和②处，我们所看到的注解是Java预定义的注解，称为元注解（Meta-Annotation），它们被Java编译器使用，会对注解类的行为产生影响。@Retention(RetentionPolicy. RUNTIME)表示NeedTest这个注解可以在运行期被JVM读取，注解的保留期限类型在java.lang.annotation.Retention类中定义，介绍如下：

SOURCE：注解信息仅保留在目标类代码的源码文件中，但对应的字节码文件将不再保留； 
CLASS：注解信息将进入目标类代码的字节码文件中，但类加载器加载字节码文件时不会将注解加载到JVM中，也即运行期不能获取注解信息； 
RUNTIME：注解信息在目标类加载到JVM后依然保留，在运行期可以通过反射机制读取类中注解信息。 
Target(ElementType.METHOD)表示NeedTest这个注解只能应用到目标类的方法上，注解的应用目标在java.lang.annotation.ElementType类中定义： 
TYPE：类、接口、注解类、Enum声明处，相应的注解称为类型注解； 
FIELD：类成员变量或常量声明处，相应的注解称为域值注解； 
METHOD：方法声明处，相应的注解称为方法注解； 
PARAMETER：参数声明处，相应的注解称为参数注解； 
CONSTRUCTOR：构造函数声明处，相应的注解称为构造函数注解； 
LOCAL_VARIABLE：局部变量声明处，相应的注解称为局域变量注解； 
ANNOTATION_TYPE：注解类声明处，相应的注解称为注解类注解，ElementType. TYPE包括ElementType.ANNOTATION_TYPE； 
PACKAGE：包声明处，相应的注解称为包注解。

如果注解只有一个成员，则成员名必须取名为value()，在使用时可以忽略成员名和赋值号（=），如@NeedTest(true)。注解类拥有多个成员时，如果仅对value成员进行赋值则也可不使用赋值号，如果同时对多个成员进行赋值，则必须使用赋值号，如DeclareParents (value = “NaiveWaiter”, defaultImpl = SmartSeller.class)。注解类可以没有成员，没有成员的注解称为标识注解，解释程序以标识注解存在与否进行相应的处理；此外，所有的注解类都隐式继承于java.lang.annotation.Annotation，但注解不允许显式继承于其他的接口。

我们希望使用NeedTest注解对业务类的方法进行标注，以便测试工具可以根据注解情况激活或关闭对业务类的测试。在编写好NeedTest注解类后，就可以在其他类中使用它了。

# 使用注解
我们在ForumService中使用NeedTest注解，标注业务方法是否需要测试，如代码清单7-2所示：

代码清单7-2 ForumService：使用注解
```java
package com.baobaotao.aspectj.anno;  
public class ForumService {  
    @NeedTest(value=true) ①  
    public void deleteForum(int forumId){  
        System.out.println("删除论坛模块："+forumId);  
    }  
    @NeedTest(value=false) ②  
    public void deleteTopic(int postId){  
        System.out.println("删除论坛主题："+postId);  
    }     
}  
```
如果注解类和目标类不在同一个包中，需要通过import引用的注解类。在①和②处，我们使用NeedTest分别对deleteForum()和deleteTopic()方法进行标注。在标注注解时，可以通过以下格式对注解成员进行赋值：

> <注解名>(<成员名1>=<成员值1>,<成员名1>=<成员值1>,…)

如果成员是数组类型，可以通过{}进行赋值，如boolean数组的成员可以设置为{true,false,true}。下面是几个注解标注的例子：

示例1，多成员的注解：
```java
@AnnoExample(id= 2868724, synopsis = "Enable time-travel",  
engineer = "Mr. Peabody",date = "4/1/2007")  
```

示例2，一个成员的注解，成员名为value。可以省略成员名和赋值符号：

```java
@Copyright("2011 bookegou.com All Right Reserved")  
```

示例3，无成员的注解：
```java
@Override  
```

示例4，成员为字符串数组的注解：
```java
@SuppressWarnings(value={"unchecked","fallthrough"})  
```
示例5，成员为注解数组类型的注解：
```java
@Reviews({@Review(grade=Review.Grade.EXCELLENT,reviewer="df"),        
           @Review(grade=Review.Grade.UNSATISFACTORY,reviewer="eg",              comment="This method needs an @Override annotation")})  
```

Reviews注解拥有一个@Review注解数组类型的成员，@Review注解类型有三个成员，其中reviewer、comment都是String类型，但comment有默认值，grade是枚举类型的成员。
由于NeedTest注解的保留限期是RetentionPolicy.RUNTIME类型，因此当ForumService被加载到JVM时，仍就可通过反射机制访问到ForumService各方法的注解信息。

# 访问注解
前面提到过，注解不会直接影响程序的运行，但是第三方程序或工具可以利用代码中的注解完成特殊的任务，间接控制程序的运行。对于RetentionPolicy.RUNTIME保留期限的注解，我们可以通过反射机制访问类中的注解。

在JDK5.0里，Package、Class、Constructor、Method以及Field等反射对象都新增了访问注解信息的方法：T getAnnotation(Class annotationClass)，该方法支持通过泛型直接返回注解对象。

下面，我们就通过反射来访问注解，得出ForumService 类中通过@NeedTest注解所承载的测试需求，如代码清单7-3所示：

代码清单7-3 TestTool：访问代码中的注解
```java
package com.baobaotao.aspectj.anno;  
import java.lang.reflect.Method;  
public class TestTool {  
    public static void main(String[] args) {  

               //①得到ForumService对应的Class对象  
        Class clazz = ForumService.class;   

                //②得到ForumSerivce对应的Method数组  
        Method[] methods = clazz.getDeclaredMethods();   

        System.out.println(methods.length);  
        for (Method method : methods) {  

                        //③获取方法上所标注的注解对象  
            NeedTest nt = method.getAnnotation(NeedTest. class);  
            if (nt != null) {  
                if (nt.value()) {  
                    System.out.println(method.getName() + "()需要测试");  
                } else {  
                    System.out.println(method.getName() + "()不需要测试");  
                }  
            }  
        }  
    }  
}  
```

在③处，通过方法的反射对象，我们获取了方法上所标注的NeedTest注解对象，接着就可以访问注解对象的成员，从而得到ForumService类方法的测试需求。运行以上代码，输出以下的信息：

> deleteForum()需要测试 
  deleteTopic()不需要测试