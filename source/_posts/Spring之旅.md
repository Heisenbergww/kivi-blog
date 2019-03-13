---
title: Spring之旅
date: 2019-03-06 17:43:56
tags: Spring Spring实战
---
# Spring之旅

>以下内容来自Craig Walls先生的《Spring实战》(第四版)

>**背景**  
*在诞生之初，创建Spring的主要目得是用来替代更加重量级的企业级Java技术，尤其是EJB。相对于EJB来说，Spring提供了更加轻量级和简单的编程模型。它增强了老式Java对象（Plain Old Java object，POJO）的功能，使其具备了之前只有EJB和其他企业级Java规范才具有的功能。
尽管J2EE能够赶上Spring的步伐，但Spring也继续在其他领域发展，比如移动开发、社交API集成、NoSQL数据库、云计算以及大数据都是Spring正在涉足和创新的领域。*

### 1.1 简化Java开发
虽然Spring用bean或者JavaBean来表示应用组件，但并不意味着Spring组件必须要遵循JavaBean规范。一个Spring组件可以是任何形式的POJO。

为了降低Java开发的复杂性，Spring采取了以下4种关键策略：
- 基于POJO的轻量级和最小侵入性编程；
- 通过依赖注入和面向接口实现松耦合；
- 基于切面和惯例进行声明式编程；
- 通过切面和模板减少样板式代码。

#### 1.1.1 激发POJO的潜能
Spring竭力避免因自身的API而弄乱你的应用代码。Spring不会强迫你实现Spring规范的接口或继承Spring规范的类，相反，在基于Spring构建的应用中，他的类通常没有任何痕迹表明你使用了Spring。最坏的场景时，一个类或许会使用Spring注解，但它依旧是POJO。

#### 1.1.2 依赖注入

**DI功能是如何实现的**

```java
package sia.knights;

public class DamselRescuingKnight implements Knight {

  private RescueDamselQuest quest;

  public DamselRescuingKnight() {
    this.quest = new RescueDamselQuest();
  }

  public void embarkOnQuest() {
    quest.embark();
  }

}
```
`DamselRescuingKnight`在它的构造函数中自行创建了`RescueDamselQuest`。这使得`DamselRescuingKnight`紧密地和`RescueDamselQuest`耦合到一起了，它极大地限制了这个骑士执行探险的能力。
而且，为这个`DamselRescuingKnight`编写单元测试将出奇的困难。在测试中，你必须保证骑士的`embarkOnQuest`方法被调用的时候，探险的`embark()`方法也要被执行。但是没有一个简单明了的方法能够实现这一点。

依赖注入会将所依赖的关系自动交给目标对象，而不是让对象自己去获取依赖。

下面的`BraveKnight`能够挑战任何形式的探险:

```java
package sia.knights;
  
public class BraveKnight implements Knight {

  private Quest quest;

  public BraveKnight(Quest quest) {
    this.quest = quest;
  }

  public void embarkOnQuest() {
    quest.embark();
  }

}
```
与之前的`DamselRescuingKnight`不同，`BraveKnight`没有自行创造探险任务，而是在构造的时候把探险任务作为构造器参数传入。这是依赖注入的方式之一，即构造器注入。
更重要的是，传入的探险类型是`Quest`，也就是所有探险任务都必须实现的一个接口。所以，`BraveKnight`能够响应`RescueDamselQuest`、`SlayDragonQuest`、`MakeRoundTableRounderQuest`等任意的`Quest`实现。

*这里的要点是`BraveKnight`没有与任何特定的`Quest`实现发生耦合。对它来说，被要求挑战的探险任务只要实现了`Quest`接口，那么具体是哪种类型的探险就无关紧要了。*

*这就是DI所带来的最大收益————松耦合。如果一个对象只通过接口（而不是具体实现或初始化过程）来表明依赖关系，那么这种依赖就能够在对象本身毫不知情的情况下，用不同的具体实现进行替换。*

对依赖进行替换的一个最常用的方法就是在测试的时候使用mock实现。
```java
package sia.knights;
import static org.mockito.Mockito.*;

import org.junit.Test;

import sia.knights.BraveKnight;
import sia.knights.Quest;

public class BraveKnightTest {

  @Test
  public void knightShouldEmbarkOnQuest() {
    Quest mockQuest = mock(Quest.class);
    BraveKnight knight = new BraveKnight(mockQuest);
    knight.embarkOnQuest();
    verify(mockQuest, times(1)).embark();
  }

}
```
**将`Quest`注入到`Knight`中**

现在`BraveKnight`类可以接受你传递给他的任意一种`Quest`实现。杀死恶龙的的探险任务：
```java
package sia.knights;

import java.io.PrintStream;

public class SlayDragonQuest implements Quest {

  private PrintStream stream;

  public SlayDragonQuest(PrintStream stream) {
    this.stream = stream;
  }

  public void embark() {
    stream.println("Embarking on quest to slay the dragon!");
  }

}
```
`SlayDragonQuest`实现了`Quest`接口，这样它就适合注入到`BraveKnight`中去了。这里的最大问题在于，我们如何将`SlayDragonQuest`交给`BraveKnight`呢，又如何将`PrintStream`交给`SlayDragonQuest`呢？  
创建应用组件之间协作的行为通常称为装配（wiring）。Spring有多种装配bean的方式，采用XML是很常见的一种装配方式。下面展示了一个简单的Spring配置文件：knights.xml，该文件将`BraveKnight`、`SlayDragonQuest`和`PrintStream`装配到一起：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans 
      http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="knight" class="sia.knights.BraveKnight">
    <constructor-arg ref="quest" />
  </bean>

  <bean id="quest" class="sia.knights.SlayDragonQuest">
    <constructor-arg value="#{T(System).out}" />
  </bean>

</beans>
```
在这里，`BraveKnight`和`SlayDragonQuest`被声明为Spring中的bean。就`BraveKnight`bean来讲，它在构造时传入了对`SlayDragonQuest`bean的引用，将其作为构造器参数。同时`SlayDragonQuest`bean的声明使用了Spring表达式语言（Spring Expression Language），将System.out（这是一个`PrintStream`）传入到了`SlayDragonQuest`的构造器中。

Spring还支持使用Java来描述配置，比如：
```java
package sia.knights.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import sia.knights.BraveKnight;
import sia.knights.Knight;
import sia.knights.Quest;
import sia.knights.SlayDragonQuest;

@Configuration
public class KnightConfig {

  @Bean
  public Knight knight() {
    return new BraveKnight(quest());
  }
  
  @Bean
  public Quest quest() {
    return new SlayDragonQuest(System.out);
  }

}
```

**观察它如何工作**
Spring通过应用上下文（Application Context）装载bean的定义并把它们组装起来。Spring应用上下文全权负责对象的创建和组装。Spring自带了多种应用上下文的实现，他们之间的主要的区别仅仅在于如何加载配置。
因为knights.xml中的bean是使用XML文件进行配置的，所以选择`ClassPathXmlApplicationContext`作为应用上下文相对是比较合适的。
```java
package sia.knights;

import org.springframework.context.support.
                   ClassPathXmlApplicationContext;

public class KnightMain {

  public static void main(String[] args) throws Exception {
    ClassPathXmlApplicationContext context = 
        new ClassPathXmlApplicationContext(
            "META-INF/spring/knights.xml");
    Knight knight = context.getBean(Knight.class);
    knight.embarkOnQuest();
    context.close();
  }

}
```
这里`main()`方法基于knights.xml文件创建了Spring应用上下文。随后它调用该应用上下文获取一个ID为knight的bean。得到Knight对象的引用后，只需简单调用`embarkOnQuest()`方法就可以执行所赋予的探险任务了。  
*注意这个类完全不知道我们的英雄骑士接受了哪种探险任务，问切完全没有意识到这是由`BraveKnight`来执行的。只有knights.xml文件知道那个骑士执行那种探险任务*

#### 1.1.3 应用切面
AOP（aspect-oriented programming，AOP）允许你把遍布应用各处的功能分离出来形成可重用的组件。[关注点分离]
每个组件除了实现自身核心功能还经常承担着额外的职责，诸如日志、事务管理和安全这样的系统服务经常融入到自身具有核心业务逻辑的组件中去，这些系统服务通常被称作*横切关注点*，它们会跨越系统的多个组件。 

如果将这些关注点分散到多个组件中去，会给代码带来双重的复杂性。
- 实现系统关注点功能的代码将会重复出现在多个组件中。这意味着如果你要改变这些关注点的逻辑，必须修改各个模块中的相关实现。
- 组件会因为那些于自身核心业务无关的代码而变得混乱。

AOP能够使这些服务模块化，并以声明的方式将他们应用到他们需要影响的组件中去。这些组件会具有更高的内聚性，并且会更加关注自身的业务，完全不需要了解涉及系统服务所带来的复杂性。总之，AOP能够确保POJOde简单性。

可以把切面想象成为覆盖在很多组件之上的一个外壳。应用是有那些实现各自业务功能的模块组成的。借助AOP,可以使用各种功能层去包裹核心业务层。这些层以声明的方式灵活的应用到系统中，你的核心应用甚至根本不知道他们的存在。  

**AOP**  
假设我们需要使用吟游诗人这个服务类来记载骑士的所有事迹。
```java
package sia.knights;

import java.io.PrintStream;

public class Minstrel {

  private PrintStream stream;
  
  public Minstrel(PrintStream stream) {
    this.stream = stream;
  }

  public void singBeforeQuest() {
    stream.println("Fa la la, the knight is so brave!");
  }

  public void singAfterQuest() {
    stream.println("Tee hee hee, the brave knight " +
    		"did embark on a quest!");
  }

}
```
在骑士执行每一个探险任务之前，`singBeforeQuest`方法会被调用；在骑士完成任务之后`singAfterQuest`方法会被调用。  
下面将`BraveKnight`和`Minstrel`组合起来：
```java
package sia.knights;
  
public class BraveKnight implements Knight {

  private Quest quest;
  private Minstrel minstrel;

  public BraveKnight(Quest quest, Minstrel minstrel) {
    this.quest = quest;
    this.minstrel = minstrel;
  }

  public void embarkOnQuest() throws QuestException {
  	minstrel.singBeforeQuest();
    quest.embark();
    minstrel.singAfterQuest();
  }

}
```
但是，`BraveKnight`管理起来了吟游诗人。吟游诗人应该做他份内的事，根不需要其实命令他这么做。毕竟，用诗歌记载骑士的探险事迹，这是吟游诗人的职责。  
但利用AOP，你可以声明吟游诗人必须歌颂骑士的探险事迹，而骑士本身并不用直接访问`Minstrel`的方法。

要将`Minstrel`抽象为一个切面，你所需要做的事情就是在一个Spring配置文件中声明它。下面是更新后的knights.xml文件，`Minstrel`被声明为一个切面。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:aop="http://www.springframework.org/schema/aop"
  xsi:schemaLocation="http://www.springframework.org/schema/aop 
      http://www.springframework.org/schema/aop/spring-aop.xsd
		http://www.springframework.org/schema/beans 
      http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="knight" class="sia.knights.BraveKnight">
    <constructor-arg ref="quest" />
  </bean>

  <bean id="quest" class="sia.knights.SlayDragonQuest">
    <constructor-arg value="#{T(System).out}" />
  </bean>

  <bean id="minstrel" class="sia.knights.Minstrel">
    <constructor-arg value="#{T(System).out}" />
  </bean>

  <aop:config>
    <aop:aspect ref="minstrel">
      <aop:pointcut id="embark"
          expression="execution(* *.embarkOnQuest(..))"/>
        
      <aop:before pointcut-ref="embark" 
          method="singBeforeQuest"/>

      <aop:after pointcut-ref="embark" 
          method="singAfterQuest"/>
    </aop:aspect>
  </aop:config>
  
</beans>
```

这里使用了Spring的aop配置命名空间把`Minstrel`bean声明为一个切面。首先，需要把`Minstrel`生命为一个bean，然后在`<aop:aspect>`元素中引用该bean。为了进一步定义切面，声明（使用`<aop:before>`）在`embarkOnQuest()`方法执行前调用`Minstrel`的`singBeforeQuest()`方法。这种方式被称为前置通知。同样的后置通知...

在这两种方式中，pointcut-ref属性都引用了名字为embark的嵌入点。该切入点是在前面的`<pointcut>`元素中定义的，并配置expression属性来选择所应用的通知。表达式的语法采用的是AspectJ的切点表达式语言。

*我们可以从这个示例中获得两个重要的观点：*
- `Minstrel`仍然是一个POJO，没有任何代码表明它要被作为一个切面使用。当我们按照上面那样进行配置后，在Spring的上下文中，`Minstrel`实际上已经变成了一个切面。
- 最重要的是`Minstrel`可以被应用到`BraveKnight`中，而`BraveKnight`不需要显式地调用它，实际上，`BraveKnight`完全不知道`Minstrel`的存在。

#### 1.1.4 使用模板消除样板式代码

样板式代码的一个常见范例是使用JDBC访问数据库查询数据。但是JDBC不是产生样板式代码的唯一场景。在许多便能成场景中往往都会导致类似的样板式代码，JMS、JNDI和使用REST服务通常也涉及大量的重复代码。
