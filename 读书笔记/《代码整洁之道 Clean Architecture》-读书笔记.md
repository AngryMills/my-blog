大家好，我是烤鸭：
关于《代码整洁之道》，记录一下读书笔记。
@[TOC]( 代码整洁之道)

#  第一章 整洁代码
## 整洁代码的艺术
优雅而高效。代码逻辑直接了解，bug无处可藏。尽量减少依赖关系，性能调优。
应用单元测试和验收测试，有意义的命名，明确的定义和提供清晰、尽量少的API。
通过所有测试、没有重复代码，提高表达力。
# 第二章 有意义的命名
## 避免误导
比如 XYZControllerEfficientHandlingOfStrings 和 XYZControllerEfficientStorageOfStrings。
## 有意义的区分
反例：
比如  字母 O和 I 和 数字 0 和 1
比如  getAccount 和  getAccounts 和 getAccountInfo ，不知道这三个方法的区别
## 使用读得出来和可搜索的名字
反例：
genhydhms  一个生成年月日时分秒的方法
应该避免下面这种命名不知道什么意义的代码。
```
for (int i = 0; i < s; i++) {
         s +=1;
}
```
## 避免使用编码
反例：
成员变量前缀加 m_ 比如 m_dsc。
不对接口命名
反例：
比如 IShapeFactory，按照阿里规范，命名提现设计模式，直接叫 ShapeFactory即可。

# 第三章 函数
**函数要短小**(阿里规范80行，作者建议不要超过120行)；
**只做一件事**(单一原则)；
函数语句处在**同一抽象层级且自顶向下**(比如getHtml()和PathParser.render(pagePath)就不是同一抽象层级)；
**switch或if/else** 语句不要做过多的逻辑处理(容易违反单一原则和开闭原则);
**起个好名字**(不怕长，要有描述性)；
**函数参数尽量少**，不要标识参数(需要的话用两个方法表示)，多参数时采用对象，可变参数不要超过三元，动词和关键词(比如write(name)或者writeField(name),可以更好的表示函数和参数的意思)；
**避免函数的副作用**(函数的变量范围过大之类的(成员变量定义为全局变量))，尽量避免输出参数；
**分隔指令与询问**(也是单一原则的体现,把检查和设值放到一个函数的问题，比如if(set("username","unclebob")){...}应换成if(attributeExists(username){set("username","unclebob"))})))；
**使用异常代替错误码**(这个还是要看业务场景,按照阿里规范第三方调用的时候需要返回错误码,如果是内部服务调用可以抛异常)，抽离try catch 代码块，catch finally 不要做其他事，定义错误枚举；
**减少重复代码**；
**遵循结构化编程**，小函数偶尔使用return、break或continue没有坏处；
没人一上来就能按照规范写代码，打磨代码，按照本章规则组装函数。
# 第四章 注释
**好的代码可以代替注释** ，比如 employ.isEligibleForFullBenefits 方法名称即注释
**必要的注释**，1. 法律信息 2. 提供信息 3. 对意图的注释（对返回值的解释） 4. 阐释：a.compareTo(b) == -1 // a < b 5. 给下一个看这个代码的人的提示 6. TODO 7. 强调 8. 公共API的javadoc 
**坏注释**，1. 多余的注释 2. 误导注释 3. 循规（不需要所有的javadoc）4. 日志式注释 5. 废话 6. 位置标记（// Action /////////////////）7. 注释和代码没有隔行 8. 注释的代码 9. HTML、非本地的(代码变量)、信息过多、非公共Javadoc。 这段感觉作者写的就很废话了...
复杂的方法有注释，简单的不需要（函数名称要清晰），需要的地方加注释(作者信息、Javadoc、TODO 等)
# 第五章 格式
**垂直格式** 像报纸学习（名称简单且一目了然）、代码间隔（不同方法中间会有空行）、靠近（有联系的变量间不需要空行）、变量声明的位置（本地变量在函数顶部、循环语句的变量在循环体中声明、实体变量在类顶部声明、相关函数放到一起、概念相关放到一起）

横向注意分隔(函数名和左括号不加空格，函数调用括号中参数逗号分隔)

```
private void measure(String line){
	lineCount++;
	int lineSize = line.length();
	totalChars += lineSize;
	lineWidthHistogram.addLine(lineSize, lineCount);
	recordWidestLine(lineSize);
}
```

不对齐的声明和赋值(变量左对齐)

if、while 要缩进

while或for语句体为空时，不写循环体的结束分号最好换行（阿里规范是空循环也要有{ }）

团队规则很重要（花很小的时间统一团队的代码规范）

# 第六章 对象和数据结构

**数据抽象** 不暴露细节且符合业务场景

**数据、对象的反对称性** 面向过程便于不改变既有数据结构下添加新函数，面向对象便于不改动既有函数前提下添加新类。（过程代码不易添加新数据结构，面向对象代码不易添加新函数）

**得墨忒耳律** 模块不应该了解调用对象的内部情形。比如 类C的方法f只应该调用以下对象的方法：

C、由f创建的对象、作为参数传递给f的对象、由C的实体变量持有的对象，方法不应该调用由任何函数返回的对象的方法。是数据结构还是对象（数据结构无所谓）、一半数据结构一半对象（公共方法导致私有变量公有化，不安全）、隐藏结构（只暴露对象的方法、隐藏内部和实现）

**数据传送对象** DTO（data transfer objects）业务流转对象、VO（展示层对象）、DO（数据层交互对象）不参杂业务逻辑

**小结** 对象暴露行为、隐藏数据，数据结构暴露数据、没有明显的行为。面向对象和面向过程看具体的场景选择。

# 第七章 错误处理

**使用异常而非错误码** （内部服务调用可以使用抛出异常，对外提供的接口最好还是返回错误码）

**先写try-catch-finally语句** 定义范围，方便满足测试

**使用不可控异常** checked exception（可控异常），异常和方法绑定，但是违反了开闭原则。如果在方法中抛出课可控异常，需要在抛出异常处之间的每个方法签名中声明该异常。如果写一套关键代码库，可控异常有用，必须捕获异常。但一般应用开发，依赖成本高于收益。

**给出异常发生的环境说明** 记录错误的堆栈信息

**依调用者需要定义异常类**  封装第三方调用api，确保返回的异常类型，简化代码

**定义常规流程** 特例模式，创建一个类或配置一个对象处理特例，就不用应付异常行为了。比如下面的代码，处理业务逻辑没有餐食消耗的时候，总账加上补贴。

```
try{
	MealExpenses expenses.getTotal; = expenseReportDAO.getMeals();
	m_total = expenses.getTotal;
}catch(MealExpensesNotFound e){
	m_total = getMealPerDiem();
}
```

应该把catch和业务代码分开。定义特例的类。

```
public class PerDiemMealExpenses implements MealExpenses{
	public int getTotal(){
		// renturn the per diem default
	}
}
```

**别返回null值** 避免NPE的最好方式，不返回null，返回空集合或者空对象(实际场景协同开发时很难保证)

**别传递null值** 避免NPE和其他运行时问题

# 第八章 边界

**使用第三方代码** 比如 java.util.Map，有一个对象想用Map做数据结构，还不要暴露细节(隐藏Map)，可以写成下面这样。

```
public class Sensors{
	private Map sensors = new HashMap();
	public Sensor getById(String id){
		return (Sensor)sensors.get(id);
	}
}
```

**浏览和学习边界** 

**学习测试好处不只是免费** 确保第三方程序包正常工作

**使用尚不存在的代码** 先定义API，后完成代码细节

**整洁的边界** 通过引入第三方边界接口的位置管理第三方边界。比如 Map 包装 或者 Adapter 模式。

# 第九章 单元测试

**TDD 三定律** (每天编写数十个测试用例，代码量大，不易管理)

在编写不能通过的单元测试前，不可编写生产代码。

只可编写刚好无法通过的单元测试，不能编译也算不通过。

只可编写刚好足以通过当前失败测试的生产代码。

**保持测试整洁** 测试代码和生产代码一样重要。测试覆盖率越高，越不会出问题。

**整洁的测试** 可读性：明确、简洁、足够的表达力。

构造-操作-检验(build-operate-check)模式。第一个环节 构造测试数据，第二个环节操作测试数据，第三个环节检验操作是否得到期望结果。

针对特定场景封装api符合特定的测试语言。

测试和生产环境代码的不同标准，测试用例可以使用 String += 的方式拼接字符串，不需要考虑性能。

每个测试用例都应该有断言，但要限制数量。（正常开发也推荐断言代替if、else）。每个测试用例要清晰，只测试一个概念，每个概念的断言数量要控制。

整洁的规则 FIRST

- first 快速。
- independent 独立。用例间相互独立，单独运行。
- repeatable 可重复。可在不同环境下重复通过。
- self-validating 自足验证。应该有布尔值输出，成功失败一目了然。
- timely 及时。用例应及时编写。单元测试应该恰好在其通过的生产代码之前编写。

# 第十章 类

**类的组织** 公共静态常量>私有静态常量>实体变量>公共函数>私有函数（符合自顶向下原则）。封装（保持变量和工具函数的私有性）

**类应该短小** 根据职责衡量代码行数。避免一个类拥有太多职责。(类名定义含混，就可能有更多职责，模糊词比如 Processor、Manager、Super)

- 单一职责-SRP(Simple Responsibility Principle)，类或模块有且只有一条加以修改的理由。SRP 是面向对象设计中最为重要的概念之一，也是较为容易理解和遵循的概念之一。系统应该由许多短小的类而不是少量巨大的类组成，每个短小的类封装一个权责，只有一个修改的原因，并与少数其他类一起协同达成期望的系统行为。
- 内聚。类应该只有少量实体变量。尽量使用局部变量，减少成员变量。如果一个类多个变量，考虑分拆成两个类。
- 保持内聚，得到短小的类。这地方作者改写了一段很长的代码，增加了几个类，目的就是为了让原有的类使用更少的变量。

**为了修改而组织**  为了避免修改而破坏SRP原则，将原本一个类的多个方法拆成多个子类，比如 Sql.class 类中的多个方法，如果新增一个 update方法，需要修改原来的Sql类。好的方式是Abstract Class Sql ，一堆继承Class，比如 SelectSql 、InsertSql 、这样新增的时候就不会对原有的产生影响。有点类似设计模式中的工厂模式。

隔离修改，遵循的是依赖倒置原则(Dependency Inversion Principle)。DIP认为类应当依赖于抽象而不是具体细节。比如有一个 Class Profile，一个接口  Stock，和一个Class TokyoStock 实现于 Stock。当想调用 TokyoStock 中的方法时，Profile中代理的对象是Stock，无须知道TokyoStock 的细节。类似设计模式中的代理模式。

# 第十一章 系统

**建造城市** 较高的抽象层级—系统层级—保持整洁。

**系统构造和使用分开** 启动过程和启动之后的运行的逻辑要分离开。比如：

```
public Service getService{
    if(service == null){
        service = new MyServiceImpl(...); 
        return service;
    }
}
```

延迟初始化，或者单例模式都使用这种方式保证返回同一个对象。

- 分解。main和application分开，比如启动的时候执行对象初始化，而使用的时候无需再初始化，只需要使用即可。

- 工厂。使用工厂模式，由工厂决定何时创建对象。隔离构造细节。
- 依赖注入。DI(Dependency Injection)。控制反转(Inversion of Control)是一种应用手段。将对象的管理权交给容器，同时由容器保证单一职责，返回一个该对象的代理对象。可以通过注解、配置文件等方式，最著名的就是spring了。

**扩容** 软件系统应该和物理系统已于，支持递增式地增长，只要切当地切分。

**Java代理** 单独的对象或类中包装方法调用。JDK的动态代理仅能与接口协同工作，代理类的话，需要字节码操作库，CGLIB、ASM、Javassist。代理会有很多相对复杂的代码，不太容易保持整洁。而很多时候需要的是插桩的机制(AOP)。

**纯Java AOP 框架** 解决了代码冗长和耦合性的问题，使用 xml 或者 annotation ，清晰易于维护。

**AspectJ 的方面** 和aop不同，aspectj 更关注切分面。

**测试驱动系统架构** 软件避免先过度设计，有效切分各个关注面。最佳的系统架构由模块化的关注面领域组成，每个关注面均用java(或其他语言)的对象实现。

**优化决策** 模块化和关注面切分成就了分散化管理和决策。模块化降低了决策的复杂性。

**明智使用添加了可论证价值的标准** 建立标准，事半功倍。

**系统需要领域特定语言** 领域特定语言(Domain-Specific language,DSL)，减少不同领域概念的代码"鸿沟"，降低翻译失误的风险。

# 第十二章 迭进

**通过迭进设计达到整洁目的**  Kent Beck 简单设计的四条规则：

- 运行所有测试
- 不可重复
- 表达程序员的意图
- 尽可能减少类和方法数量
- 以上规则按其重要程度排列

现在这个规则在测试驱动或者代码测试用例算基本的了，还需要考虑实际的业务场景、抽象等，甚至会有一个专门用于测试的项目。

**简单设计规则1:运行所有测试**   遵循SRP、DIP，减少耦合

**简单设计规则2~4:重构**   测试消除了对清理代码就会破坏代码的恐惧。减少重复、保证表达力、尽可能减少类和方法的数量。

**不可重复**   抽取重复方法可能违反了SRP原则，可以考虑使用模板方法模式。

**表达力**  代码应当清晰表达作者的意图，函数名称易于理解，单元测试也具有表达性。否则可能连自己都看不懂自己写的代码。

**尽可能少的类和方法**  其实 SRP和DIP 本身是有冲突的，只有找到符合自己业务场景的设计才是最好的。保持短小、消除重复，同时也要避免过度设计。

# 第十三章 并发编程

关于并发编程推荐大家看下 《JAVA并发编程实战》

**为什么要并发** 并发是一种解耦策略。把做什么和何时做分解开。servlet是单线程模式，需要手动对接口开发保证QPS。

关于并发：

- 并发改进性能：看场景，跟锁的级别、系统、内核都有关系等等，有时能改进性能。
- 并发程序需要修改设计：并发利用性能换取时间的方式很可能导致线程安全问题，需要好好设计。
- Web、EJB等容器也需要理解并发
- 并发在性能和编写额外代码增加一些开销
- 正确的并发是复杂的，并发缺陷不易重现
- 并发常常需要对设计策略的根本性修改

**挑战**

```
public class X{
    private int lastIdUsed;
    public int getNextId(){
        return ++lastIdUsed;
    }
}
```

并发场景下的得到 的lastIdUsed 可能是多种结果。

**并发防御原则** 

- 单一权责原则(SRP) ：并发相关代码有自己的开发、修改和调优生命周期。建议分离并发相关代码和其他代码。
- 推论：限制数据作用域：对并发区域加 synchronized 关键字（谨记数据封装；严格限制对可能被共享数据的访问）
- 推论：使用数据副本：多线程复制对象并以只读方式对待
- 推论：线程尽可能独立：尝试将数据分解到独立线程操作

**了解Java库** 使用线程安全的类库(juc)、尽可能使用非锁解决方案

**了解执行模型** 

- 生产者-消费者模型：生产者和消费者之间的队列是一种限定资源(并发环境中有固定尺寸或数量的资源)。例如 数据库连接
- 读者-作者模型：作者线程倾向于长期锁定许多读者线程(读者线程需要更新的共享资源)，会导致吞吐量的问题。
- 宴席哲学家：典型的死锁问题

**警惕同步方法之间的依赖** ：避免使用一个共享对象的多个方法。如果必须使用的话：

- 基于客户端的锁定：客户端代码在调用第一个方法前锁定服务器，确保锁的范围覆盖了调用最后一个方法的代码。
- 基于服务端的锁定：在服务端内创建锁定服务端的方法，调用所有方法，然后解锁。
- 适配服务端：创建执行锁定的中间层。基于服务端锁定的例子，但不修改原始服务端代码。

**保持同步区域微小** 锁会带来延迟和额外开销。尽可能减小同步区域。

**很难编写正确的关闭代码** 平静关闭很难做到。常见问题与死锁有关，线程一直等待永远不会到来的信号。比如生产-消费模型，如果生产者线程收到信号迅速关闭，消费者线程可能还在等待生产者线程发来消息。尽早考虑关闭问题。

**测试线程代码** 编写有潜力暴露问题的测试，在不同的编程配置、系统配置和负载条件下频繁运行。

- 将伪失败看作可能的线程问题：编程代码导致”不可能失败的“。不要将系统错误归咎于偶发事件。
- 先使非线程代码可工作：不要同时追踪非线程缺陷和线程缺陷。
- 编写可插拔的线程代码：目的是不同的配置环境下运行。
- 编写可调整的线程代码：要获得良好的线程平衡，常常需要试错。
- 运行多于处理器数量的线程：任务交换越频繁，越有可能找到错过临界区域或导致死锁的代码。
- 在不同平台上运行：尽早并经常在所有目标平台运行线程代码(不同操作系统有着不同线程策略)
- 装置试错代码：增加 Object.wait()、Object.sleep() 等方法的调用，改变代码执行顺序。两种方法：硬编码、自动化。
- 硬编码：例如。

```
public synchronized String nextUrlOrNull(){
    if(hasNext()){
        String url = urlGenerator.next();
        Thread.yield(); // inserted for testing
        updateHasNext();
        return url;
    }
    return null;
}
```

插入对yield 的调用，将改变代码的执行路径，有可能导致代码在以前未失败的地方失败。如果代码出错，并非是 yield()方法调用(yield 只是当前线程让步，不应该影响过程或结果)。这种方式的弊端：硬编码还得找到合适的调用时机，很可能找不到缺陷。

- 自动化：使用第三方框架装置代码。如 Aspect-Oriented Framework、CGLIB 或 ASM。

```
public synchronized String nextUrlOrNull){
    if(hasNext()){
        ThreadJinglePoing.jiggle();
        String url = urlGenerator.next();
        ThreadJinglePoing.jiggle();
        updateHasNext();
        ThreadJinglePoing.jiggle();
        return url;
    }
    return null;
}
```

ThreadJinglePoing 类可以多种实现，生产环境什么都不做，测试环境加随机数、睡眠或者让步。重点是让不同线程异动。1

# 第十四章 逐步改进

这一章节作者用了一个代码示例讲解了 循序渐进的代码过程，好的代码不是一次就完成的，需要多次改进。不能每次完成需求，就投入到下一个需求，而不考虑当前代码的优雅和可维护性。

这个章节是纯代码优化分析，代码草稿 ->缩短程序(注意命名方式、函数大小和代码格式) -> 派生基类 ->

采用TDD(重构要兼容之前的case) -> 派生变量和类移植 -> 隐藏异常细节 -> 公共方法抽取 -> 清理重复代码 ->

减少原始类中的成员变量 -> 封装异常(ApiException) -> Args 类处理异常(ErrorMessage)是否合理?

可以看一下这个人的博客，有修改后的代码示例。

https://waltyou.github.io/Clean-Code-Practice/

小结：永远保持代码整洁和简单。

# 第十五章 Junit内幕

以 ComparisonCompactor 为例，讲述了优化过程。

重构常会推翻另一次重构，重构是一种不停试错的迭代过程。

junit的源码地址，感兴趣的可以看下：https://github.com/junit-team/junit4

# 第十六章 重构SerialDate

重构 org.jfree.date.SerialDate

抽象工厂—> 完善用例 —> 优化结构(使用枚举) —>  简化函数 —> 修改函数名 —> 抽取公共函数 

源码网上也找不到，这一章跳过。



# 第十七章 味道与启发

**注释** 

- 不恰当的信息： 作者、最后修改时间、SPR等数据不应该在数据中出现(实际业务中，作者还是需要的，最好留下联系方式)
- 废弃的注释 ：过时、无关或者不正确的都属废弃注释。
- 冗余注释：比如 i++; // i 自增
- 糟糕的注释： 使用正确的语法和拼写。(一般会统一注释格式)
- 注释掉的代码：直接删除(和阿里规范一样)

**环境** 

- 需要多步才能实现的构建：构建系统应该是单步的小操作，不需要从源代码一点点迁出代码。比如 maven/gradle/ant 都可以实现。
- 需要多步才能做到的测试：单个指令可以运行全部单元测试。

**函数**

- 过多的函数：函数尽量少，不超过3个。
- 输出参数：如果想修改参数，直接修改对应的对象状态。
- 标识参数：不要使用boolean类型参数
- 死函数：删除永远不会调用的代码

**一般性问题**

- 一个源文件中存在多种语言： 尽量减少源文件中额外语言的数量和范围。(前后端分离了，别再在项目里写jsp和html了)

- 明显的行为未被实现：遵循 "最小惊诧原则"，比如定义日期api的时候，期望字符串Monday翻译为Day.Monday，使用户依靠对函数的直觉，不需要阅读源代码。

- 不正确的边界行为：设立正确的边界，完善单元测试。

- 忽视安全：编译器警告要注意。（规约扫描没有警告的代码才可以提交）

- 重复：DIY（Dont't repeat yourself），重复意味着遗漏了抽象。

- 在错误的抽象层级上的代码：通常抽象类容纳高层级概念，派生类容纳底层级概念。高层级的基类应该对常量、变量或工具函数一无所知。比如下面的代码，percentFull("多满")不应该出现在基类中，可以放到派生接口中。

  ```
  pubilc interfece Stack{
  	Object pop();
  	void push(Object o);
  	double percentFull();
  }
  ```

- 基类依赖于派生类：基类应该对派生类一无所知，也有例外，基类拥有在派生类中选择的代码。（以springboot-data-redis 为例，2.0以前的连接默认使用jedis，2.0以后使用的lettuce。）

- 信息过多：设计良好的模块有非常小的接口，耦合度低。隐藏数据、工具函数、常量、临时变量。不要创建大量方法和大量实体变量的类。不要为子类创建大量受保护变量和函数。尽力保持接口紧凑。通过限制信息来控制耦合。

- 死代码：删除永不执行的代码。

- 垂直分隔：变量和函数在靠近被使用的地方定义。

- 前后不一致：从一而终。比如 特定函数中名为reponse 持有HttpServletResponse对象，那么用到HttpServletResponse的函数也用同样的变量名。

- 混淆视听：没用的变量、函数、注释信息等等，保持文件整洁。

- 人为耦合：不相互依赖的东西不该耦合。应该花点时间研究变量、常量的放置位置。

- 特性依恋(G14)：没有必要在三方类中定义某个类对象，(会暴露类的内部情形)，破坏了面对对象原则。

- 选择算子参数：尽量不使用选择函数的参数，比如 true/false、枚举、整数等，比如 true/fasle 在整个funcation里做条件判断，不如 true，function A/false function B 这样，拆分单独的方法

- 晦涩的意图：短小节凑的代码不一定易懂，比如下面。

  ```
  public int m_otCalc(){
  	return iThsWkd * iThsRte + (int) Math.round(0.5 * iThsRte + Math.max(0, iThsWkd - 400))
  }
  ```

- 位置错误的权责：代码应该放在读者自然而然期待它所在的地方。比如一般写的配置类都会放到 config 包下面。

- 不恰当的静态方法：
  比如 Math.max(double a,double b) 是个很好的静态方法，不在单个实体上操作。通常倾向于选用非静态方法。如果有疑问就用非静态函数。如果使用静态函数，确保不要有多态行为的机会。

-  使用解释性变量：比如下面代码，如果用两个中间变量定义一下，更容易理解。

  ```
  Matcher match = headerPattern.matcher(line);
  if(match.find())
  {
  	String key = match.group(1);
  	String value = match.group(2);
  	headers.put(key,toLowerCase(), value);
  }
  ```

- 函数名称应该表达其行为：比如下面函数 看不出函数行为，应该改成 addDaysTo 或 increaseByDays

  ```
  Date newDate = date.add(5);
  ```

- 理解算法：不仅要通过全部测试，还要找出最佳方案。

- 把逻辑依赖改为物理依赖：比如下面的类是一个统计报表的类，PAGE_SIZE 不应该是该类中的变量，这是一种错误的逻辑依赖，应该改为物理依赖，通过 HourlyReport 中名为 getMaxPageSize()方法物理化这种依赖。

  ```
  public class HourlyReporter{
  	private HourlyReportFormatter formatter;
  	private List<LineItem> page;
  	private final int PAGE_SIZE = 55;
  	
  	public HourlyReporter(HourlyReportFormatter formatter){
  		this.formatter = formatter;
  		page = new ArrayList<LineItem>();
  	}
  	//...
  }
  ```

- 用多态替代 If/Else 或 Switch/Case：
  对于给定的选择类型，不应有多于一个switch 语句。在那个switch语句中的多个case，必须创建多态对象，取代系统中其他类似switch语句。可以使用策略模式。

- 遵循标准约定：团队的编码标准。

- 用命令常量替代魔术数：定义常量代替魔法值

- 准确：比如对待金额的地方，需要明确保留几位(使用 Bigdecimal)

- 结构甚于约定：基类的抽象方法比良好命名的枚举强(抽象方法必须按照基类的实现)

- 封装条件：if(shouldBeDeleted(timer)) 要好于 if(timer.hasExpired()) && !(timer.isRecurrent())

- 避免否定性条件：if(buffer.shouldCompact()) 要好于 if(!buffer.shouldNotCompact())

- 函数只做一件事：一个 “支付函数” 需要做，遍历雇员 -> 检查是否给雇员工资 -> 支付薪水，改成 3 个函数。

- 掩蔽时序耦合：函数的调用顺序和放置的地方相关联。

- 别随意：构建代码需要理由，而且理由应与代码结构相契合。比如 VariableExpandingWidgetRoot 作为公共的工具类，不应该定义在 AliasLinkWidget 类里。

  ```
  public class AliasLinkWidget extends ParentWidget{
  	public static class VariableExpandingWidgetRoot{
  		//...
  	}
  }
  ```

- 封装边界条件：把处理边界条件的代码集中到一处。下面的代码中 level + 1 出现了两次，封装到局部变量。

  ```
  if(level + 1 < tags.length){
  	parts = new Parse(body, tags, level + 1,offset + endTag);
  }
  ```

- 函数应该只在一个抽象层级上：函数中的语句应该在同一抽象层级上，该层级应该是函数名所示操作的下一层。下面方法混杂了至少两个抽象层级，第一个是横线有尺寸，第二个是hr标记自身的语法。

  ```
  public String render(){
  	StringBuffer html = new StringBuffer("<hr");
  	if(size > 0){
  		html.append(" size=\"").append(size + 1).append("\"");
  		html.append(“>”);
  		return html.toString();
  	}
  }
  ```

  重构后：

  ```
  public String render(){
  	HtmlTag hr = new HtmlTag("hr");
  	if (extraDashes > 0){
  		hr.addAttribute("size",hrSize(extraDashes));
  		return hr.html();
  	}
  }
  
  private String hrSize(int height){
  	int hrSize = height + 1;
  	return String.format("%d", hrSize);
  }
  ```

- 在较高层级放置可配置数据：较高抽象层级的默认常量或配置值，不要放到较低层级的函数中。（常量放到类顶部而变量名大写）

- 避免传递浏览：避免 a.getB().getC().doSomething()的代码，应该改为 myCollaborator.doSomething()

**Java**

- 通过使用通配符避免过长的导入清单：同名不同包的类需要指定名称导入。
- 不要继承常量：常量放到静态常量类里。
- 常量 VS 枚举：使用枚举代替常量，枚举能表达更多而且更灵活。

**名称**

- 采用描述性名称：确认名称具有描述性，比如 count、max（业务含义） 代替 i 、j、k。

- 名称应与抽象层级相符：方法名称要体现抽象层级，比如 调制解调器的抽象层级的方法，
   dial 是实现类的方法名称(电话拨号)

  ```
  public interface Modem{
  	boolean dial(String phoneNumber);
  	boolean disconnect;
  }
  ```

  应改为：connect 更能体现层级

  ```
  public interface Modem{
  	boolean connect(String connectionLocator);
  	boolean disconnect;
  }
  ```

- 尽可能使用标准命名法：如果使用装饰模式，类命名时加上 Decorator ，例如 AutoHangupModemDecorator。

- 无歧义的名称：比如说 getData() 应该具体到业务场景 getTreeDataByColor

- 为较大作用范围选用较长名称：下面简短的代码用变量 i ，n 刚好，rollCount 代替 i 反倒混乱

  ```
  private void rollMany(int n, int pins){
  	fori (int i = 0,i < n; i ++){
  		g.roll(pins);
  	}
  }
  ```

- 避免编码：不应在名称中包括类型或作用范围信息。比如 m_ 或者 f_ 完全无用。（redis常用前缀 业务属性： 比如 user: use_key）

- 名称应该说明副作用：名称更好的应该是 createOrReturnOos 。

  ```
  public ObjectOutputStream getOos() throws IOException(){
  	if(m_oos == null){
  		m_oos = new ObjectOutputStream(m_socket.getOutputStream());
  	}
  	return m_oos;
  }
  ```

**测试**

- 测试不足：一套测试应该测到所有可能失败的东西。
- 使用覆盖率工具：覆盖率工具能汇报测试策略中的缺口。
- 别略过小测试：小测试易于编写，文档价值高于编写成本。
- 被忽略的测试就是对不确定事物的疑问：可以使用注释掉的测试或者用 @Ignore 标记测测试来表达我们对于需求的疑问。用哪种方式取决于代码关联代码是否要编译。
- 测试边界条件：算法在中间部分正确但边界判断错误的情形很常见。比如用 new Integer(200) == new Integer(200)
- 全面测试相近的缺陷：趋势倾向于扎堆。某个函数中发现一个缺陷时，最好全面测试那个函数。
- 测试失败的模式有启发性：根据测试用例失败的模式来诊断问题。
- 测试覆盖率的模式有启发性：查看被或未被已通过的测试执行的代码，往往能发现失败的测试为何失败。
- 测试应该快速：慢速的测试是不会被运行的测试。

整洁代码并非遵循一套规则写就。专业性和技艺来自驱动规程的价值观。


最后附上最新的阿里规范github 地址：

https://github.com/alibaba/p3c





