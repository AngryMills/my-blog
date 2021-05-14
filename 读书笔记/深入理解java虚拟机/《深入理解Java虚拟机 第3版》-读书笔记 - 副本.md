# 《深入理解Java虚拟机》-读书笔记（第一、第二部分）

大家好，我是烤鸭：
	《深入理解Java虚拟机》-读书笔记。

## 第一部分：走进Java
### 第1章 走进Java
#### 1.1 概述

摆脱了硬件平台的束缚，实现了“**一次编写，到处运行**”的理想；它提供了一种相对安全的内存管理和访问机制，避免了绝大部分内存泄漏和指针越界问题；它实现了热点代码检测和运行时编译及优化，这使得Java应用能随着运行时间的增长而获得更高的性能；它有一套完善的应用程序接口，还有无数来自商业机构和开源社区的第三方类库来帮助用户实现各种各样的功能。

#### 1.2 Java技术体系

从广义上讲，Kotlin、Clojure、JRuby、Groovy等运行于Java虚拟机上的编程语言及其相关的程序都属于Java技术体系中的一员。

![1](.\1.png)

#### 1.3 Java发展史

现在是 2021.3 , JDK 前几天刚发布 16

![2](.\2.png)

#### 1.4 Java虚拟机家族

虚拟机始祖：Sun Classic/Exact VM ：JDK 1.2 之前唯一的虚拟机

武林盟主：**HotSpot VM：目前使用最多的**

小家碧玉：Mobile/Embedded VM：移动和嵌入式市场

天下第二：BEA JRockit/IBM J9 VM：IBM J9虚拟机的职责分离与模块化做得比HotSpot更优秀

软硬合璧：BEA Liquid VM/Azul VM：与特定硬件平台绑定、软硬件配合工作的专有虚拟机，Azul公司为它编写了新的垃圾收集器，也修改了HotSpot内的许多实现细节，在要求低延迟、快速预热等场景中，Zing VM都要比HotSpot表现得更好。

挑战者：Apache Harmony/Google Android Dalvik VM：曾经是Android平台的核心组成部分之一

没有成功，但并非失败：Microsoft JVM及其他：Windows XP高级产品经理Jim Cullinan称：“我们花费了三年的时间和Sun公司打官司，当时他们试图阻止我们在Windows中支持Java，现在我们这样做了，可他们又在抱怨，这太具有讽刺意味了。”

#### 1.5 展望Java技术的未来

##### 无语言倾向

Graal VM才是真正意义上与物理计算机相对应的高级语言虚拟机，理由是它与物理硬件的指令集一样，做到了只与机器特性相关而不与某种高级语言特性相关。如果Java语言或者 HotSpot虚拟机真的有被取代的一天，那从现在看来Graal VM是希望最大的一个候选项。

##### 新一代即时编译器

HotSpot虚拟机中含有两个即时编译器，分别是编译耗时短但输出代码优化程度较低的客户端编译器（简称为C1）以及编译耗时长但输出代码优化质量也更高的服务端编译器（简称为C2）。自JDK 10起，HotSpot中又加入了一个全新的即时编译器：Graal编译器，看名字就可以联想到它是来自于前一节提到的Graal VM。Graal编译器是以C2编译器替代者的身份登场的。

（使用 -XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler 参数来启用Graal编译器）

##### 向native迈进

Substrate VM补全了Graal VM“Run Programs Faster Anywhere”愿景蓝图里的最后一块拼图，让Graal VM支持其他语言时不会有重量级的运行负担。譬如运行JavaScript代码，Node.js的V8引擎执行效率非常高，但即使是最简单的HelloWorld，它也要使用约20MB的内存，而运行在Substrate VM上的Graal.js，跑一个HelloWorld则只需要4.2MB内存，且运行速度与V8持平。

##### 灵活的胖子

JDK1.4 (Java Virtual Machine Profiler Interface，JVMPI 和 Java Virtual Machine Debug Interface，JVMDI)

JDK5 (Java Virtual Machine Tool Interface，JVMTI)

JDK9(Java Virtual Machine Compiler Interface，JVMCI)

JDK 10，HotSpot又重构了Java虚拟机的垃圾收集器接口(Java Virtual Machine Compiler Interface)

##### 语言语法持续增强

JDK 7的Coins项目结束以后，Java社区又创建了另外一个新的语言特性改进项目Amber，JDK 10至13里面提供的新语法改进基本都来自于这个项目

https://openjdk.java.net/projects/amber/

#### 1.6 实战

http://openjdk.java.net/

笔者下载的OpenJDK 12源码包大小为 171MB，解压之后约为579MB。 

尽量在Linux或者MacOS上构建OpenJDK，在官方 文档上要求编译OpenJDK至少需要2～4GB的内存空间（CPU核心数越多，需要的内存越大），而且 至少要6～8GB的空闲磁盘空间，不要看OpenJDK源码的大小只有不到600MB，要完成编译，过程 会产生大量的中间文件，并且编译出不同优化级别（Product、FastDebug、SlowDebug）的HotSpot虚拟机可能要重复生成这些中间文件，这都会占用大量磁盘空间。

后续编译和IDE中源码调试还是看书吧。

#### 1.7 小结

## 第二部分 自动内存管理

### 第2章 Java内存区域与内存溢出异常

#### 2.1 概述

C、C++的开发人员，既拥有每一个对象的“所有权”，又担负每一个对象从开始到终结的维护责任。而Java是把控制内存的权力交给了Java虚拟机。

#### 2.2 运行时数据区域

![2](.\3.png)

##### 2.2.1 程序计数器

程序计数器（Program Counter Register）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令。由于Java虚拟机的多线程是通过线程轮流切换、分配处理器执行时间的方式来实现的，在任何一 个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。为了线程切换后能恢复到正确的执行位置，**每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存**。

如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地 址；如果正在执行的是本地（Native）方法，这个计数器值则应为空（Undefined）。此内存区域是唯一一个在《Java虚拟机规范》中没有规定任何OutOfMemoryError情况的区域。

##### 2.2.2 Java 虚拟机栈

与程序计数器一样，**Java虚拟机栈（Java Virtual Machine Stack）也是线程私有的，它的生命周期 与线程相同**。虚拟机栈描述的是Java方法执行的线程内存模型：**每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态连接、方法出口等信息**。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。 

局部变量表存放了编译期可知的各种Java虚拟机**基本数据类型**（boolean、byte、char、short、int、 float、long、double）、**对象引用**（reference类型，它并不等同于对象本身，可能是一个指向对象起始 地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和**returnAddress 类型**（指向了一条字节码指令的地址）。 

这些数据类型在局部变量表中的存储空间以**局部变量槽（Slot）**来表示，其中64位长度的long和 double类型的数据会占用两个变量槽，其余的数据类型只占用一个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。请读者注意，这里说的“大小”是指变量槽的数量，虚拟机真正使用多大的内存空间（譬如按照1个变量槽占用32个比特、64个比特，或者更多）来实现一个变量槽，这是完全由具体的虚拟机实现自行决定的事情。 

在《Java虚拟机规范》中，对这个内存区域规定了两类异常状况：**如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常**；如果Java虚拟机栈容量可以动态扩展，当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError异常。 在HotSpot虚拟机上是不会由于虚拟机栈无法扩展而导致OutOfMemoryError异常——**只要线程申请栈空间成功了就不会有OOM**，但是如果申请时就失败，仍然是会出现OOM异常的

##### 2.2.3 本地方法栈

本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别只是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而**本地方法栈则是为虚拟机使用到的本地（Native 方法服务**。

##### 2.2.4 Java堆

Java堆（Java Heap）是虚拟机所管理的内存中最大的一块。Java堆是被**所有线程共享的一块内存区域，在虚拟机启动时创建**。此内存区域的唯一目的就是存放对象实例，Java 世界里“几乎”所有的对象实例都在这里分配内存。在《Java虚拟机规范》中对Java堆的描述是：“**所有的对象实例以及数组都应当在堆上分配**“。

根据《Java虚拟机规范》的规定，**Java堆可以处于物理上不连续的内存空间中**，但在逻辑上它应该被视为连续的，这点就像我们用磁盘空间去存储文件一样，并不要求每个文件都连续存放。**但对于大对象（典型的如数组对象），多数虚拟机实现出于实现简单、存储高效的考虑，很可能会要求连续的内存空间**。 

通过参数-Xmx和-Xms设定 堆大小。

##### 2.2.4 方法区

在JDK 8以前，很多人都更愿意把方法区称呼为“永久代”。本质上这两者并不是等价的，因为仅仅是当时的HotSpot虚拟机设计团队选择把收集器的分代设计扩展至方法区，或者说**使用永久代来实现方法区而已，这样使得HotSpot的垃圾收集器能够像管理Java堆一样管理这部分内存，省去专门为方法区编写内存管理代码的工作**。

到了**JDK 7的HotSpot，已经把原本放在永久代的字符串常量池、静态变量等移出**，而到了**JDK 8，终于完全废弃了永久代**的概念，改用与JRockit、J9一样在本地内存中实现的元空间（Meta- space）来代替，**把JDK 7中永久代还剩余的内容（主要是类型信息）全部移到元空间中**。

**方法区的内存回收目标主要是针对常量池的回收和对类型的卸载**，一般来说这个区域的回收效果比较难令人满意，尤其是类型的卸载，条件相当苛刻，但是这部分区域的回收有时又确实是必要的。

根据《Java虚拟机规范》的规定，如果方法区无法满足新的内存分配需求时，将抛出OutOfMemoryError异常。

##### 2.2.6 运行时常量池

**运行时常量池（Runtime Constant Pool）是方法区的一部分**。**Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table）**，用于存放**编译期生成的各种字面量与符号引用**，这部分内容将在类加载后存放到方法区的运行时常量池中。 

除了保存Class文件中描述的符号引用外，还会把由符号引用翻译出来的直接引用也存储在运行时常量池中。

Class文件常量池的另外一个重要特征是具备动态性，并非预置入Class文件中常量池的内容才能进入方法区运行时常 量池，**运行期间也可以将新的常量放入池中**，这种特性被开发人员利用得比较多的便是**String类的 intern()方法**。

##### 2.2.7 直接内存

**直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分**，也不是《Java虚拟机规范》中定义的内存区域。但是这部分内存也被频繁地使用，而且**也可能导致OutOfMemoryError异常出现**，所以我们放到这里一起讲解。 

在JDK 1.4中新加入了**NIO（New Input/Output）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作**。这样能在一些场景中显著提高性能，因为**避免了在Java堆和Native堆中来回复制数据**。 

**本机直接内存的分配不会受到Java堆大小的限制**，但是，既然是内存，则肯定还是**会受到本机总内存（包括物理内存、SWAP分区或者分页文件）大小以及处理器寻址空间的限制**，一般服务器管理员配置虚拟机参数时，会根据实际内存去设置-Xmx等参数信息，但经常忽略掉直接内存，使得 各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现OutOfMemoryError异常。

#### 2.3 HotSpot虚拟机对象探秘
##### 2.3.1 对象的创建

当Java虚拟机遇到一条字节码new指令时，首先将去检查这个指令的参数是否能在常量池中定位到 一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。

在类加载检查通过后，接下来虚拟机将为新生对象分配内存。假设Java堆中内存是绝对规整的，所有被使用过的内存都被放在一边，空闲的内存被放在另一边，中间放着一个指针作为分界点的指示器，**那所分配内存就仅仅是把那 个指针向空闲空间方向挪动一段与对象大小相等的距离，这种分配方式称为“指针碰撞”（Bump The Pointer）**。但如果Java堆中的内存并不是规整的，已被使用的内存和空闲的内存相互交错在一起，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，**在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“空闲列表”（Free List）**。当使用**Serial、ParNew等带压缩整理过程的收集器**时，系统采用的分配算法是**指针碰撞，既简单又高效**；而当使用**CMS这种基于清除（Sweep）算法的收集器**时，理论上就只能采用**较为复杂的空闲列表来分配内存**。

并发情况下的对象创建：一种是对分配内存空间的动作进行同步处理——实际上虚拟机是采用**CAS配上失败重试的方式保证更新操作的原子性**；另外一种是把内存分配的动作按照线程划分在不同的空间之中进 行，即每个线程在Java堆中预先分配一小块内存，称为**本地线程分配缓冲（Thread Local Allocation Buffer，TLAB）**，哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有**本地缓冲区用完了，分配新的缓存区时才需要同步锁定**。虚拟机是否使用TLAB，可以通过-XX：+/-UseTLAB参数来 设定。

内存分配完成之后，内存空间（不包括对象头）的初始化为零值。

对象头信息：对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码（实际上对象的哈希码会延后到真正调用Object::hashCode()方法时才计算）、对象的GC分代年龄等信息，是否启用偏向锁等。

一般来说（由字节码流中new指令后面是否跟随invokespecial指令所决定，Java编译器会在遇到new关键字的地方同时生成这两条字节码指令，但如果直接通过其他方式产生的则不一定如此），new 指令之后会接着执行 <init>()，这样才算完成对象的构造。

下面是HotSpot虚拟机字节码解释器（bytecodeInterpreter.cpp）中的代码片段：

```
// 确保常量池中存放的是已解释的类 
if (!constants->tag_at(index).is_unresolved_klass()) { 
    // 断言确保是klassOop和instanceKlassOop（这部分下一节介绍）
    oop entry = (klassOop) *constants->obj_at_addr(index); 
    assert(entry->is_klass(), "Should be resolved klass"); 
    klassOop k_entry = (klassOop) entry; 
    assert(k_entry->klass_part()->oop_is_instance(), "Should be instanceKlass"); 
    instanceKlass* ik = (instanceKlass*) k_entry->klass_part(); 
    // 确保对象所属类型已经经过初始化阶段 
    if ( ik->is_initialized() && ik->can_be_fastpath_allocated() ) { 
        // 取对象长度 
        size_t obj_size = ik->size_helper(); 
        oop result = NULL; 
        // 记录是否需要将对象所有字段置零值 
        bool need_zero = !ZeroTLAB; 
        // 是否在TLAB中分配对象 
        if (UseTLAB) { 
            result = (oop) THREAD->tlab().allocate(obj_size); 
        }
        if (result == NULL) { need_zero = true; 
        // 直接在eden中分配对象 
        retry: HeapWord* compare_to = *Universe::heap()->top_addr(); 
        HeapWord* new_top = compare_to + obj_size; 
        // cmpxchg是x86中的CAS指令，这里是一个C++方法，通过CAS方式分配空间，并发失败的 话，转到retry中重试直至成功分配为止 
        if (new_top <= *Universe::heap()->end_addr()) { 
            if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) { 
                goto retry; 
            }
            result = (oop) compare_to; } 
        }
        if (result != NULL) { 
            // 如果需要，为对象初始化零值 
            if (need_zero ) { 
                HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize; 
                obj_size -= sizeof(oopDesc) / oopSize; if (obj_size > 0 ) { 
                    memset(to_zero, 0, obj_size * HeapWordSize); 
                } 
            }
            // 根据是否启用偏向锁，设置对象头信息 
        if (UseBiasedLocking) { 
            result->set_mark(ik->prototype_header()); 
        } else {
            result->set_mark(markOopDesc::prototype()); 
        }
        result->set_klass_gap(0); 
        result->set_klass(k_entry); 
        // 将对象引用入栈，继续执行下一条指令 
        SET_STACK_OBJECT(result, 0);
        UPDATE_PC_AND_TOS_AND_CONTINUE(3,1);
        }
    }
}
```

##### 2.3.2 对象的内存布局

对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）

HotSpot虚拟机对象的**对象头部分包括两类信息。第一类是用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，这部分数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32个比特和64个比特，官方称它为“Mark Word”**。对象需要存储的运行时数据很多，其实已经超出了32、64位Bitmap结构所能记录的 最大限度，但对象头里的信息是与对象自身定义的数据无关的额外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个有着动态定义的数据结构，以便在极小的空间内存储尽量多的数据，根 据对象的状态复用自己的存储空间。例如在32位的HotSpot虚拟机中，如对象未被同步锁锁定的状态 下，**Mark Word的32个比特存储空间中的25个比特用于存储对象哈希码，4个比特用于存储对象分代年龄，2个比特用于存储锁标志位，1个比特固定为0**，在其他状态（轻量级锁定、重量级锁定、GC标记、可偏向）下对象的存储内容如表2-1所示。

![2](.\4.png)

对象头的另外一部分是**类型指针，即对象指向它的类型元数据的指针，Java虚拟机通过这个指针来确定该对象是哪个类的实例**。并不是所有的虚拟机实现都必须在对象数据上保留类型指针，换句话说，查找对象的元数据信息并不一定要经过对象本身，这点我们会在下一节具体讨论。此外，**如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据**，因为虚拟机可以通过普通 java对象的元数据信息确定Java对象的大小，但是如果数组的长度是不确定的，将无法通过元数据中的信息推断出数组的大小。

**实例数据部分是对象真正存储的有效信息，即我们在程序代码里面所定义的各种类型的字段内容**，无论是从父类继承下来的，还是在子类中定义的字段都必须记录起来。这部分的**存储顺序会受到虚拟机分配策略参数（-XX：FieldsAllocationStyle参数）和字段在Java源码中定义顺序的影响**。HotSpot虚拟机默认的分配顺序为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers，OOPs），从以上默认的分配策略中可以看到，相同宽度的字段总是被分配到一起存放，在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前。如果HotSpot虚拟机的+XX：CompactFields参数值为true（默认就为true），那子类之中较窄的变量也允许插入父类变量的空隙之中，以节省出一点点空间。 

对象的第三部分是**对齐填充**，这并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。由于HotSpot虚拟机的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说就是**任何对象的大小都必须是8字节的整数倍**。对象头部分已经被精心设计成正好是**8字节的倍数（1倍或者2倍）**，因此，如果对象实例数据部分没有对齐的话，就需要**通过对齐填充来补全**。 

##### 2.3.3 对象的访问定位

创建对象自然是为了后续使用该对象，我们的Java程序会通过**栈上的reference数据来操作堆上的具体对象**。由于reference类型在《Java虚拟机规范》里面只规定了它是**一个指向对象的引用**，并没有定义这个引用应该通过什么方式去定位、访问到堆中对象的具体位置，所以对象访问方式也是由虚拟机实现而定的，主流的访问方式主要有使用**句柄和直接指针两种**： 

- 如果使用**句柄**访问的话，Java堆中将可能会划分出一块内存来作为句柄池，**reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自具体的地址信息**，其结构如图2-2所示。
  ![2](.\5.png)

- 如果使用**直接指针**访问的话，Java堆中对象的内存布局就必须考虑如何放置访问类型数据的相关信息，**reference中存储的直接就是对象地址**，如果只是访问对象本身的话，就不需要多一次间接访问的开销，如图2-3所示。 
  ![2](.\6.png)

这两种对象访问方式各有优势，使用**句柄来访问的最大好处就是reference中存储的是稳定句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要被修改**。 

使用**直接指针来访问最大的好处就是速度更快，它节省了一次指针定位的时间开销**，由于对象访问在Java中非常频繁，因此这类开销积少成多也是一项极为可观的执行成本，就本书讨论的主要**虚拟机HotSpot而言，它主要使用第二种方式进行对象访问**（有例外情况，如果使用了Shenandoah收集器的 话也会有一次额外的转发，具体可参见第3章），但从整个软件开发的范围来看，在各种语言、框架中使用句柄来访问的情况也十分常见。

#### 2.4 实战：OutOfMemoryError异常
##### 2.4.1 Java堆溢出

```
/**
 * @author zzm
 * @Description VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {
    static class OOMObject {
    }
    
    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        while (true) {
            list.add(new OOMObject());
        }
    }
}
```

运行结果：

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid19628.hprof ...
Heap dump file created [28651733 bytes in 0.077 secs]
```

出现Java堆内存溢出时，异常堆栈信息“java.lang.OutOfMemoryError”会跟随进一步提示“Java heap space”。

使用内存分析工具，比如 JProfiler 或者 Eclipse Memory Analyzer）对Dump出来的堆转储快照进行分析，如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链，找到泄漏对象是通过怎 样的引用路径、与哪些GC Roots相关联，才导致垃圾收集器无法回收它们，**根据泄漏对象的类型信息以及它到GC Roots引用链的信息，一般可以比较准确地定位到这些对象创建的位置，进而找出产生内存泄漏的代码的具体位置**。

![2](.\7.png)

如果不是内存泄漏，换句话说就是内存中的对象确实都是必须存活的，那就应当**检查Java虚拟机的堆参数（-Xmx与-Xms）设置，与机器的内存对比，看看是否还有向上调整的空间**。再从代码上检查是否存在某些对象生命周期过长、持有状态时间过长、存储结构设计不合理等情况，尽量减少程序运行期的内存消耗。

##### 2.4.2 虚拟机栈和本地方法栈溢出

由于HotSpot虚拟机中并不区分虚拟机栈和本地方法栈，因此对于HotSpot来说，-Xoss参数（设置本地方法栈大小）虽然存在，但实际上是没有任何效果的，**栈容量只能由-Xss参数来设定**。关于虚拟机栈和本地方法栈，在《Java虚拟机规范》中描述了两种异常： 

- 如果**线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常**。 

- 如果**虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出OutOfMemoryError异常**。 

  - 使用-Xss参数减少栈内存容量。 

    结果：抛出StackOverflowError异常，异常出现时输出的堆栈深度相应缩小。 

  
      package src.jvm.oom;
      
      import java.util.ArrayList;
      import java.util.List;
      
      /**
       * @author zzm
       * @Description VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
       */
      public class HeapOOM {
          static class OOMObject {
          }
          
          public static void main(String[] args) {
              List<OOMObject> list = new ArrayList<OOMObject>();
              while (true) {
                  list.add(new OOMObject());
              }
          }
      }
  运行结果：
  
  ```
  stack length:984
    Exception in thread "main" java.lang.StackOverflowError
    	at src.jvm.oom.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:8)
    	at src.jvm.oom.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:9)
    	at src.jvm.oom.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:9)
    	at src.jvm.oom.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:9)
  ```

  - 定义了大量的本地变量，增大此方法帧中本地变量表的长度。 

    结果：抛出StackOverflowError异常，异常出现时输出的堆栈深度相应缩小。 
  
    ```
    package src.jvm.oom;
    /*** @author zzm */
    public class JavaVMStackSOF_2 {
    	private static int stackLength = 0;
        public static void test() {
            long unused1, unused2, unused3, unused4, unused5, unused6, unused7, unused8, unused9, unused10, unused11, unused12, unused13, unused14, unused15, unused16, unused17, unused18, unused19, unused20, unused21, unused22, unused23, unused24, unused25, unused26, unused27, unused28, unused29, unused30, unused31, unused32, unused33, unused34, unused35, unused36, unused37, unused38, unused39, unused40, unused41, unused42, unused43, unused44, unused45, unused46, unused47, unused48, unused49, unused50, unused51, unused52, unused53, unused54, unused55, unused56, unused57, unused58, unused59, unused60, unused61, unused62, unused63, unused64, unused65, unused66, unused67, unused68, unused69, unused70, unused71, unused72, unused73, unused74, unused75, unused76, unused77, unused78, unused79, unused80, unused81, unused82, unused83, unused84, unused85, unused86, unused87, unused88, unused89, unused90, unused91, unused92, unused93, unused94, unused95, unused96, unused97, unused98, unused99, unused100;
            stackLength++;
            test();
            unused1 = unused2 = unused3 = unused4 = unused5 = unused6 = unused7 = unused8 = unused9 = unused10 = unused11 = unused12 = unused13 = unused14 = unused15 = unused16 = unused17 = unused18 = unused19 = unused20 = unused21 = unused22 = unused23 = unused24 = unused25 =
                    unused26 = unused27 = unused28 = unused29 = unused30 = unused31 = unused32 = unused33 = unused34 = unused35 = unused36 = unused37 = unused38 = unused39 = unused40 = unused41 = unused42 = unused43 = unused44 = unused45 = unused46 = unused47 = unused48 = unused49 = unused50 = unused51 = unused52 = unused53 = unused54 = unused55 = unused56 = unused57 = unused58 = unused59 = unused60 = unused61 = unused62 = unused63 = unused64 = unused65 = unused66 = unused67 = unused68 = unused69 = unused70 = unused71 = unused72 = unused73 = unused74 = unused75 = unused76 = unused77 = unused78 = unused79 = unused80 = unused81 = unused82 = unused83 = unused84 = unused85 = unused86 = unused87 = unused88 = unused89 = unused90 = unused91 = unused92 = unused93 = unused94 = unused95 = unused96 = unused97 = unused98 = unused99 = unused100 = 0;
        }
        
        public static void main(String[] args) {
            try {
                test();
            } catch (Error e) {
                System.out.println("stack length:" + stackLength);
                throw e;
            }
        }
    }
            
    ```



  运行结果：

  ```
  stack length:6752
  Exception in thread "main" java.lang.StackOverflowError
  	at src.jvm.oom.JavaVMStackSOF_2.test(JavaVMStackSOF_2.java:10)
  	at src.jvm.oom.JavaVMStackSOF_2.test(JavaVMStackSOF_2.java:10)
  	at src.jvm.oom.JavaVMStackSOF_2.test(JavaVMStackSOF_2.java:10)
  	at src.jvm.oom.JavaVMStackSOF_2.test(JavaVMStackSOF_2.java:10)
  ```

实验结果表明：无论是由于栈帧太大还是虚拟机栈容量太小，当新的栈帧内存无法分配的时候，HotSpot虚拟机抛出的都是StackOverflowError异常。

**线程创建也可能出现内存溢出（每个线程分配到的栈内存越大，建立线程数量自然就越少，线程创建可能耗尽内存）**

我的事64位系统，用了作者的demo，改小了栈内存，死机了两次，还是没能复现内存溢出...

在32位操作系统下的运行结果： 

```
Exception in thread "main" java.lang.OutOfMemoryError: unable to create native thread
```

##### 2.4.3 方法区和运行时常量池溢出

```
package src.jvm.oom;

public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        String str1 = new StringBuilder("计算机").append("软件").toString();
        System.out.println(str1.intern() == str1);
        String str2 = new StringBuilder("ja").append("va").toString();
        System.out.println(str2.intern() == str2);
    }
}
```

自JDK 7起，原本存放在永久代的**字符串常量池被移至Java堆**之中，intern()方法实现就不需要再拷贝字符串的实例到永久代了，既然字符串常量池已经移到Java堆中，那**只需要在常量池里记录一下首次出现的实例引用即可**，因此intern()返回的引用和由StringBuilder创建的那个字符串实例就是同一个。而对str2比较返回false，这是因为**“java”这个字符串在执行String-Builder.toString()之前就已经出现过了，字符串常量池中已经有它的引用**，不符合intern()方法要求“首次遇到”的原则，**“计算机软件”这个字符串则是首次出现的，因此结果返回true**。 

Metaspace的设置：

- -XX：MaxMetaspaceSize：**设置元空间最大值**，默认是-1，即不限制，或者说只受限于本地内存大小。

- -XX：MetaspaceSize：**指定元空间的初始空间大小，以字节为单位**，达到该值就会触发垃圾收集进行类型卸载，同时收集器会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过-XX：MaxMetaspaceSize（如果设置了的话）的情况下，适当提高该值。

- -XX：MinMetaspaceFreeRatio：**作用是在垃圾收集之后控制最小的元空间剩余容量的百分比，可减少因为元空间不足导致的垃圾收集的频率**。类似的还有-XX：Max-MetaspaceFreeRatio，用于控制最大的元空间剩余容量的百分比。 

##### 2.4.4 本机直接内存溢出

直接内存（Direct Memory）的容量大小可通过-XX：MaxDirectMemorySize参数来指定，如果不去指定，则默认与Java堆最大值（由-Xmx指定）一致。

```
package src.jvm.oom;

import sun.misc.Unsafe;

import java.lang.reflect.Field;

/*** VM Args：-Xmx20M -XX:MaxDirectMemorySize=10M * @author zzm */
public class DirectMemoryOOM {
    private static final int _1MB = 1024 * 1024;
    
    public static void main(String[] args) throws Exception {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }
}
```

运行结果：

```
Connected to the target VM, address: '127.0.0.1:52128', transport: 'socket'
Exception in thread "main" java.lang.OutOfMemoryError
	at sun.misc.Unsafe.allocateMemory(Native Method)
	at src.jvm.oom.DirectMemoryOOM.main(DirectMemoryOOM.java:16)
```

如果发现内存溢出而且Heap Dump文件很小，就有可能是直接内存溢出，是否直接或间接使用了DirectMemory(NIO)等

#### 2.5 本章小结

虚拟机内存划分，哪部分区域和代码操作可能导致内存溢出异常。

### 第3章 垃圾收集器与内存分配策略
#### 3.1 概述

垃圾收集（Garbage Collection，下文简称GC）诞生时提出的三大问题：哪些内存需要回收？什么时候回收？ 如何回收？ 如今技术成熟，我们关心的是：当需要**排查各种内存溢出、内存泄漏问题时，当垃圾收集成为系统达到更高并发量的瓶颈**时，我们就必须对这些“自动化”的技术实施必要的监控和调节。 

#### 3.2 对象已死？
##### 3.2.1 引用计数算法

当前对象内置一个计数器，每当有一个地方引用当前对象，计数器就加一，反之就减一。

原理简单，判断效率高。对应复杂场景需要考虑的情况比较多，比如下面的循环引用。

objA 和 objB 互被对方引用(实际连个对象都已经是null了)，如果按照引用计数法，就无法回收。

```
package src.jvm.gc;

/*** Vm args: -XX:+PrintGCDetails testGC()方法执行后，objA和objB会不会被GC呢？ * @author zzm */
public class ReferenceCountingGC {
    public Object instance = null;
    private static final int _1MB = 1024 * 1024;
    /*** 这个成员属性的唯一意义就是占点内存，以便能在GC日志中看清楚是否有回收过 */
    private byte[] bigSize = new byte[2 * _1MB];
    
    public static void main(String[] args) {
        testGC();
    }
    
    public static void testGC() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;
        objA = null;
        objB = null; // 假设在这行发生GC，objA和objB是否能被回收？ 
        System.gc();
    }
}
```

结果如下，Java的gc回收是可以回收这两个对象（8009K->855K），说明使用的不是引用计数算法。

```
[GC (System.gc()) [PSYoungGen: 8009K->847K(75776K)] 8009K->855K(249344K), 0.0010770 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 847K->0K(75776K)] [ParOldGen: 8K->696K(173568K)] 855K->696K(249344K), [Metaspace: 3074K->3074K(1056768K)], 0.0047179 secs] [Times: user=0.11 sys=0.00, real=0.01 secs] 
Heap
 PSYoungGen      total 75776K, used 1951K [0x000000076ba00000, 0x0000000770e80000, 0x00000007c0000000)
  eden space 65024K, 3% used [0x000000076ba00000,0x000000076bbe7c68,0x000000076f980000)
  from space 10752K, 0% used [0x000000076f980000,0x000000076f980000,0x0000000770400000)
  to   space 10752K, 0% used [0x0000000770400000,0x0000000770400000,0x0000000770e80000)
 ParOldGen       total 173568K, used 696K [0x00000006c2e00000, 0x00000006cd780000, 0x000000076ba00000)
  object space 173568K, 0% used [0x00000006c2e00000,0x00000006c2eae250,0x00000006cd780000)
 Metaspace       used 3081K, capacity 4592K, committed 4864K, reserved 1056768K
  class space    used 326K, capacity 424K, committed 512K, reserved 1048576K
```

##### 3.2.2 可达性分析算法

当前主流商用程序语言的内存管理都是通过可达性分析来判定对象是否存活。这个算法的基本思路就是通过一系列称为**“GC Roots”的根对象作为起始节点集**，从这些节点开始，根据引用关系向下搜索，**搜索过程所走过的路径称为“引用链”（Reference Chain）**，如果**某个对象到GC Roots间没有任何引用链相连，或者用图论的话来说就是从GC Roots到这个对象不可达时**，则证明此对象是不可能再被使用的。

![2](.\10.png)

在Java技术体系里面，固定可作为GC Roots的对象包括以下几种： 

- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等。 

- 在方法区中类静态属性引用的对象，譬如Java类的引用类型静态变量。 

- 在方法区中常量引用的对象，譬如字符串常量池（String Table）里的引用。·在本地方法栈中JNI（即通常所说的Native方法）引用的对象。 

- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（比如 NullPointExcepiton、OutOfMemoryError）等，还有系统类加载器。 

- 所有被同步锁（synchronized关键字）持有的对象。 

- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。 

GCRoots也要考虑跨代收集的情况。

##### 3.2.3 再谈引用

引用分为**强引用（Strongly Re-ference）、软引用（Soft Reference）、弱引用（Weak Reference）和虚引用（Phantom Reference）**4种，这4种引用强度依次逐渐减弱。 

- 强引用是最传统的“引用”的定义，是指在程序代码之中普遍存在的引用赋值，即类似“Object obj=new Object()”这种引用关系。无论任何情况下，**只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象**

- 软引用是用来描述一些还有用，但非必须的对象。**只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常**。在JDK 1.2版之后提供了SoftReference类来实现软引用。
- 弱引用也是用来描述那些非必须对象，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生为止。**当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象**。在JDK 1.2版之后提供了WeakReference类来实现弱引用。
- 虚引用也称为“幽灵引用”或者“幻影引用”，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也**无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知**。在JDK 1.2版之后提供了PhantomReference类来实现虚引用。

##### 3.2.4 生存还是死亡？

即使在可达性分析算法中判定为不可达的对象，也不是“非死不可”的，这时候它们暂时还处于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程：**如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记，随后进行一次筛选，筛选的条件是此对象是 否有必要执行finalize()方法**。假如对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，那么虚拟机将这两种情况都视为“没有必要执行”。

**如果这个对象被判定为确有必要执行finalize()方法，那么该对象将会被放置在一个名为F-Queue的队列之中，并在稍后由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行它们的finalize() 方法**。这里所说的“执行”是指虚拟机会触发这个方法开始运行，但并不承诺一定会等待它运行结束。这样做的原因是，如果某个对象的finalize()方法执行缓慢，或者更极端地发生了死循环，将很可能导 致F-Queue队列中的其他对象永久处于等待，甚至导致整个内存回收子系统的崩溃。finalize()方法是对 象逃脱死亡命运的最后一次机会，稍后收集器将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移出“即将回收”的集 合；如果对象这时候还没有逃脱，那基本上它就真的要被回收了。

##### 3.2.5 回收方法区

方法区的垃圾收集主要回收两部分内容：废弃的常量和不再使用的类型。

没有任何字符串对象引用 常量池中的“java”常量，且虚拟机中也没有其他地方引用这个字面量，系统回收时会回收这个“java”常量。

判定一个类型是否属于“不再被使用的类”，需要同时满足下面三个条件：

- 该类所有的实例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的实例。 
- 加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换类加载器的场景，如 OSGi、JSP的重加载等，否则通常是很难达成的。 
- 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

Java虚拟机被允许对满足上述三个条件的无用类进行回收，这里说的仅仅是“被允许”，而并不是和对象一样，没有引用了就必然会回收。

#### 3.3 垃圾收集算法

##### 3.3.1 分代收集理论

- 部分收集（Partial GC）：指目标不是完整收集整个Java堆的垃圾收集，其中又分为：
  - 新生代收集（Minor GC/Young GC）：指目标只是新生代的垃圾收集。 
  - 老年代收集（Major GC/Old GC）：指目标只是老年代的垃圾收集。目前只有CMS收集器会有单独收集老年代的行为。另外请注意“Major GC”这个说法现在有点混淆，在不同资料上常有不同所指，读者需按上下文区分到底是指老年代的收集还是整堆收集。
  - 混合收集（Mixed GC）：指目标是收集整个新生代以及部分老年代的垃圾收集。目前只有G1收集器会有这种行为。 

- 整堆收集（Full GC）：收集整个Java堆和方法区的垃圾收集。 

##### 3.3.2 标记-清除算法

最早出现也是最基础的垃圾收集算法是“标记-清除”（Mark-Sweep）算法，分为“标记”和“清除”两个阶段。

主要缺点有两个：第一个是**执行效率不稳定**，如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时必须进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低；第二个是**内存空间的碎片化问题**，标记、清除之后会产生大 量不连续的内存碎片，空间碎片太多可能会导致当以后在程序运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

![2](.\11.png)

##### 3.3.3 标记-复制算法

标记-复制算法常被简称为复制算法。算法需要复制的就是占少数的存活对象，而且每次都是针对整个半区进行内存回收，分配内存时也就不用考虑有空间碎片的复杂情况，只要移动堆顶指针，按顺序分配即可。确实点浪费空间。

**HotSpot虚拟机默认Eden和Survivor的大小比例是8∶1：1，发生垃圾搜集时，将Eden和Survivor中仍然存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉Eden和已用过的那块Survivor空间。**

内存的**分配担保**也一样，如果**另外一块Survivor空间没有足够空间存放上一次新生代收集下来的存活对象**，这些对象便将**通过分配担保机制直接进入老年代**，这对虚拟机来说就是安全的。

##### 3.3.4 标记-整理算法

标记-复制算法在对象存活率较高时就要进行较多的复制操作，效率将会降低。

标记-清除算法与标记-整理算法的本质差异在于前者是一种非移动式的回收算法，而后者是移动式的。

是否移动对象都存在弊端，移动则内存回收时会更复杂，不移动则内存分配时会更复杂。从垃圾收集的停顿时间来看，不移动对象停顿时间会更短，甚至可以不需要停顿，但是从整个程序的吞吐量来看，移动对象会更划算。

**HotSpot虚拟机里面关注吞吐量的Parallel Scavenge收集器是基于标记-整理算法的，而关注延迟的CMS收集器则是基于标记-清除算法的。**

CMS  平时多数时间都采用标记-清除算法，直到内存空间的碎片化程度已经大到影响对象分配时，再采用标记-整理算法收集一次，以获得规整的内存空间。

#### 3.4 HotSpot的算法细节实现
##### 3.4.1 根节点枚举

所有收集器在根节点枚举这一步骤时都是必须暂停用户线程("Stop The World")的。

在HotSpot的解决方案里，是使用一组称为OopMap的数据结构来达到这个目的。**一旦类加载动作完成的时候，HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来，在即时编译（见第11章）过程中，也会在特定的位置记录下栈里和寄存器里哪些位置是引用**。这样收集器在扫描时就可以直接得知这些信息了，并不需要真正一个不漏地从方法区等GC Roots开始查找。 

##### 3.4.2 安全点

实际上HotSpot也的确没有为每条指令都生成OopMap，前面已经提到，只是在“特定的位置”记录 了这些信息，这些位置被称为**安全点（Safepoint）**。有了安全点的设定，也就决定了用户程序执行时并非在代码指令流的任意位置都能够停顿下来开始**垃圾收集，而是强制要求必须执行到达安全点后才能够暂停**。因此，安全点的选定既不能太少以至于让收集器等待时间过长，也不能太过频繁以至于过 分增大运行时的内存负荷。安全点位置的**选取基本上是以“是否具有让程序长时间执行的特征”为标准进行选定的**，因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这样的原因而长时间执行，**“长时间执行”的最明显特征就是指令序列的复用，例如方法调用、循环跳转、异常跳转等都属于指令序列复用，所以只有具有这些功能的指令才会产生安全点**。 

用户线程到达安全点的方式：

- 抢先式中断（Preemptive Suspension）：垃圾收集发生时，系统首先把所有用户线程全部中断，如果发现有用户线程中断的地方不在安全点上，就恢复这条线程执行，让它一会再重新中断，直到跑到安全点上
- 主动式中断（Voluntary Suspension）：不直接对线程操作，仅仅简单地设置一个标志位，各个线程执行过程时会不停地主动去轮询这个标志，一旦发现中断标志为真时就自己在最近的安全点上主动中断挂起。

##### 3.4.3 安全区域

安全区域可以看作是安全点的延伸，在线程"不执行"(Sleep或者Blocked)状态发生。

**线程执行到安全区域的代码会进行标记，垃圾收集会过滤被标记的线程，如果线程要离开安全区域时，它要检查虚拟机是否已经完成了根节点枚举（或者垃圾收集过程中其他需要暂停用户线程的阶段）**，如果完成了，那线程就当作没事发生过，继续执行；否则它就必须一直等待，直到收到可以离开安全区域的信号为止。

##### 3.4.4 记忆集与卡表

为解决对象跨代引用所带来的问题，**垃圾收集器在新生代中建立了名为记忆集（Remembered Set）的数据结构，用以避免把整个老年代加进GC Roots扫描范围**。

记忆集是一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构。

记忆集的记录精度：

- 字长精度：每个记录精确到一个机器字长（就是处理器的寻址位数，如常见的32位或64位，这个精度决定了机器访问物理内存地址的指针长度），该字包含跨代指针。 
- 对象精度：每个记录精确到一个对象，该对象里有字段含有跨代指针。 
- 卡精度：每个记录精确到一块内存区域，该区域内有对象含有跨代指针。

第三种"卡精度"指的是用"卡表"的方式实现记忆集，卡表和记忆集的关系类似HashMap和Map。

卡表最简单的形式可以只是一个字节数组，HotSpot默认的卡表标记逻辑如下：

```
CARD_TABLE [this address >> 9] = 0;
```

**CARD_TABLE 中每一个元素都对应标识内存区域中的一块特定大小的内存块，被称为"卡页"(Card Page)**。

**一个卡页的内存中通常包含不止一个对象，只要卡页内有一个（或更多）对象的字段存在着跨代指针，那就将对应卡表的数组元素的值标识为1，称为这个元素变脏（Dirty），没有则标识为0。**

##### 3.4.5 写屏障

在HotSpot虚拟机里是通过写屏障（Write Barrier）技术维护卡表状态的。

写屏障可以看作在虚拟机层面对“引用类型字段赋值”这个动作的AOP切面，在引用对象赋值时会产生一个环形（Around）通知，供程序执行额外的动作，也就是说赋值的前后都在写屏障的覆盖范畴内。**在赋值前的部分的写屏障叫作写前屏障（Pre-Write Barrier），在赋值 后的则叫作写后屏障（Post-Write Barrier）**。HotSpot虚拟机的许多收集器中都有使用到写屏障，但直至G1收集器出现之前，其他收集器都只用到了写后屏障。

写后屏障更新卡表 ：

```
void oop_field_store(oop*field, oop new_value) { 
	// 引用字段赋值操作 
	*field = new_value; 
	// 写后屏障，在这里完成卡表状态更新 
	post_write_barrier(field, new_value); 
}
```

-XX：+UseCondCardMark 解决 "伪共享"问题（现代中央处理器的缓存系统中是以缓存行（Cache Line）为单位存储的，当多线程修改互相独立的变量时，如果这些变量恰好共享同一个缓存行，就会彼此影响（写回、无效化或者同步）而导致性能降低，这就是伪共享问题）。

启用参数后，先检查卡表标记，只有当该卡表元素未被标记过时才将其标记为变脏。

```
if (CARD_TABLE [this address >> 9] != 0) 
    CARD_TABLE [this address >> 9] = 0;
```

##### 3.4.6 并发的可达性分析

三色标记（Tri-color Marking）:

- 白色：**表示对象尚未被垃圾收集器访问过**。显然在可达性分析刚刚开始的阶段，所有的对象都是白色的，若在分析结束的阶段，仍然是白色的对象，即代表不可达。 

- 黑色：**表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经扫描过**。黑色的对象代表已经扫描过，它是安全存活的，如果有其他对象引用指向了黑色对象，无须重新扫描一遍。黑色对象不可能直接（不经过灰色对象）指向某个白色对象。 

- 灰色：**表示对象已经被垃圾收集器访问过，但这个对象上至少存在一个引用还没有被扫描过**。

可达性分析的扫描过程，可以看做是从黑—>灰(波峰)—>白 的过程。

当且仅当以下两个条件同时满足时，会产生“对象消失”的问题，即原本应该是黑色的对象被误标为白色：

- 赋值器插入了一条或多条从黑色对象到白色对象的新引用； 

- 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。

解决并发扫描时的对象消失的解决方案：增量更新（Incremental Update）和原始快照（Snapshot At The Beginning， SATB）。

- 增量更新破坏的第一个条件：黑色对象一旦新插入了指向白色对象的引用之后，它就变回灰色对象 

  了（CMS）

- 原始快照要破坏的是第二个条件：无论引用关系删除与否，都会按照刚刚开始扫描那一刻的对象图快照来 进行搜索（G1、Shenandoah）

#### 3.5 经典垃圾收集器
##### 3.5.1 Serial收集器

单线程垃圾收集，已经过时了。

##### 3.5.2 ParNew收集器

Serial收集器的多线程版本，和CMS(old gc)组合使用的多(ParNew young gc)

##### 3.5.3 Parallel Scavenge收集器(JDK8 默认)

基于标记-复制算法的多线程收集器，和cms的不同的是，cms关注尽可能缩短stp的时间，而ps的目标是达到可控制的吞吐量。

吞吐量 = 运行用户代码时间 / 运行用户代码时间 + 运行垃圾收集时间

- -XX：MaxGCPauseMillis参数允许的值是一个大于0的毫秒数，收集器将尽力保证内存回收花费的时间不超过用户设定值。不是越小越好，越小gc越频繁。

- -XX：GCTimeRatio参数的值则应当是一个大于0小于100的整数，也就是垃圾收集时间占总时间的比率，相当于吞吐量的倒数。默认值99，即允许最大 1%（1/(1+99)）的垃圾收集时间。

- -XX：+UseAdaptiveSizePolicy，这是一个开关参数，当这个参数被激活之后，就不需要人工指定新生代的大小（-Xmn）、Eden与Survivor区 的比例（-XX：SurvivorRatio）、晋升老年代对象大小（-XX：PretenureSizeThreshold）等细节参数 了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量。这种调节方式称为垃圾收集的自适应的调节策略（GC Ergonomics）。

##### 3.5.4 Serial Old收集器

单线程收集器，使用标记-整理算法。JDK5以前与Parallel Scavenge收集器搭配使用，另外一种是作为CMS并发收集失败时的后备方案。

##### 3.5.5 Parallel Old收集器(JDK8 默认)

Parallel Old是Parallel Scavenge收集器的老年代版本，基于标记-整理算法。在**注重吞吐量或者处理器资源较为稀缺**的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器这个组合

##### 3.5.6 CMS收集器

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，是基于标记-清除算法实现的，整个过程分为四个步骤：

- 初始标记（CMS initial mark）
- 并发标记（CMS concurrent mark）
- 重新标记（CMS remark） 
- 并发清除（CMS concurrent sweep）

初始标记和重新标记需要 STP，初始标记很快，只是标记一下GC ROOTS能直接关联到的对象，速度很快；并发标记是从GC ROOTS的直接关联对象开始遍历整个对象图的过程。重新标记是为了修正并发标记期间可能产生的增量数据，这个阶段耗时介于初始标记和并发标记之间。并发清除，清理判断已经死亡的对象。

优点：并发收集、低停顿

缺点：

- 占用处理器资源(CMS默认启动的回收线程数是 (核数+3)/4)，四核以上的处理器占用不超过25%，四核以下的，占用的就比较多了。
- 无法处理"浮动垃圾"(在CMS并发标记和并发清除阶段产生的垃圾对象)，只能下一次再收集。老年代必须预留一部分空间供并发收集使用，目前CMS 启动阈值默认是92%，如果内存无法满足程序分配新对象的需求，就会触发"并发失败"，使用Serial Old进行Old GC
- "标记清除"会产生大量内存碎片，当无法保证足够大的连续空间来分配对象，会提前触发Full GC。

##### 3.5.7 Garbage First收集器（JDK9默认）

G1 可以面向堆内任何部分来组成回收集(Collection Set)进行回收，Mixed GC 模式。

G1 把连续的Java堆划分为多个大小相等的独立区域(Region)，Region中还有一类特殊的Humongous区域，专门用来存储大对象。大小超过一半Region空间即可判断为大对象，每个Region大小可以通过参数 数-XX：G1HeapRegionSize设定，取值范围为1MB～32MB，且应为2的N次幂。

跨Region引用如何解决：**每个Region维护自己的记忆集，会记录下别的Region指向自己的指针，并标记这些指针分别在哪些卡页的范围之内**。G1的记忆集在存储结构的本质上试一种哈希表，Key是别的Region的起始地址，Value是一个集合，里面存储的是元素是卡表的索引号。这种"双向"的卡表结构实现更复杂，同时Region数量比传统收集器的分代数量明显要多得多，会占用更高的内存(相当于堆内容的10%-20%的额外内存)。

并发标记阶段如何保证收集线程与用户线程互不干扰：CMS采用增量更新算法实现，G1采用原始快照(SATB)算法实现。**G1为每个Region设计了两个名为TAMS(Top at Mark Start)的指针，把Region中的一部分空间划分出来用来并发回收过程中的新对象分配，并发回收时新分配的对象地址都必须要在这两个指针位置上**。G1 收集器默认在这个地址以上的对象是被隐式标记过的，即默认是存活的，不纳入回收范围。与CMS类似，如果内存回收速度赶不上内存分配速度，也会触发Full GC。

怎么样建立可靠的停顿预测模型：通过 -XX：MaxGCPauseMillis参数指定停顿时间。G1收集器的停顿预测模型是以衰减均值(Decaying Average)为理论基础来实现的，在垃圾收集过程中，G1收集器会记录每个Region的回收耗时、每个Region记忆集里的脏卡数量等各个可测量的步骤花费的成本，并分析得出平均值、标准偏差、置信度等统计信息。Region的统计状态越新越能决定其回收的价值。然后通过这些信息预测由哪些Region组成的回收集才可以在不超过期望停顿时间的约束下获得最高的收益。

G1收集器的运作过程大致可划分为以下四个步骤： 

- 初始标记（Initial Marking）：仅仅只是标记一下GC Roots能直接关联到的对象，并且**修改TAMS指针的值**，让下一阶段用户线程并发运行时，能正确地在可用的Region中分配新对象。**这个阶段需要停顿线程，但耗时很短，而且是借用进行Minor GC的时候同步完成的**，所以G1收集器在这个阶段实际并没有额外的停顿。
- 并发标记（Concurrent Marking）：**从GC Root开始对堆中对象进行可达性分析**，递归扫描整个堆 里的对象图，找出要回收的对象，这阶段耗时较长，但可**与用户程序并发执行**。当对象图扫描完成以后，还要**重新处理SATB记录下的在并发时有引用变动的对象**。
- 最终标记（Final Marking）：用户线程做另一个短暂的暂停，用于处理并发阶段结束后仍遗留 下来的最后那少量的SATB记录。 
- 筛选回收（Live Data Counting and Evacuation）：负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个Region构成回收集，然后把决定**回收的那一部分Region的存活对象复制到空的Region中，再清理掉整个旧Region的全部空间**。这里的操作涉及存活对象的移动，是必须暂停用户线程，由多条收集器线程并行完成的。

G1收集器除了并发标记外，其余阶段也是要完全暂停用户线程。

虽然G1和CMS都使用卡表处理跨代指针，但**G1的卡表实现更为复杂，而且堆中的每个Region都有自己的卡表，导致G1的记忆集可能会占堆容量的20%甚至更多的内存空间**。而CMS的卡表只有一份，而且只需要处理老年代到新生代的引用，省下了新生代到年老代的引用的维护开销。

**CMS用写后屏障维护卡表，而G1不仅用了写后屏障，还用到了写前屏障跟踪并发时的指针变化情况。相比起增量更新(CMS)，原始快照搜索(G1)能够减少并发标记和重新标记阶段的消耗，避免CMS在最终标记阶段停顿时间过长的缺点，但是在用户程序运行过程中确实会产生由跟踪引用变化带来的额外负担。**

目前在小内存应用上CMS的表现大概率仍然要会优于G1，而在大内存应用上G1则大多能发挥其优势，这个优劣势的Java堆容量平衡点通常在6GB至8GB之间。

#### 3.6 低延迟垃圾收集器

衡量指标：内存占用、吞吐量和延迟。

![2](.\12.png)

CMS和G1分别使用增量更新和原始快照技术，实现标记阶段的并发。标记之后的处理，CMS使用标记-清除算法，产生内存碎片。G1颗粒度更小，但还是无法避免STW。

Shenandoah和ZGC，几乎整个工作过程全部都是并发的，只有初始标记、最终标记这些阶段有短暂的停顿，这部分停顿的时间基本上是固定的。

##### 3.6.1 Shenandoah收集器 

OpenJDK有，而OracleJDK没有的垃圾收集器。

摒弃了在G1中耗费大量内存和计算资源去维护的记忆集，改用名为“连接矩阵”的全局数据结构来记录跨Region的引用关系，降低了处理跨代指针时的记忆集维护消耗，也降低了伪共享问题（见3.4.4节）的发生概率。

- 初始标记：与G1一样，首先标记与GC Roots直接关联的对象，这个阶段仍是“Stop The World”的，但**停顿时间与堆大小无关，只与GC Roots的数量相关**。 
- 并发标记：与G1一样，遍历对象图，标记出全部可达的对象，这个阶段是与用户线程一起并发的，**时间长短取决于堆中存活对象的数量以及对象图的结构复杂程度**。
- 最终标记：与G1一样，处理剩余的SATB扫描，并在这个阶段统计出回收价值最高的Region，将这些Region构成一组回收集（Collection Set）。**最终标记阶段也会有一小段短暂的停顿**。
- 并发清理：这个阶段用于清理那些整个区域内连一个存活对象都没有找到的Region（这类Region被称为Immediate Garbage Region）。
- 并发回收：在这个阶段，Shenandoah要把回收集里面的存活对象先复制一份到其他未被使用的Region之 中(不暂停线程的并发执行)。将会通过读屏障和被称为“Brooks Pointers”的转发指针来解决。**并发回收阶段运行的时间长短取决于回收集的大小**。 
- 初始引用更新：并发回收阶段复制对象结束后，还需要把堆中所有指向旧对象的引用修正到复制后的新地址，初始引用更新时间很短，**会产生一个非常短暂的停顿**。
- 并发引用更新：**引用更新，与用户线程一起并发，时间长短取决于内存涉及的引用数量**。与并发标记不同，它不再需要沿着对象图来搜索，只需要按照内存物理地址的顺序，线性地搜索出引用类型，把旧值改为新值即可 。
- 最终引用更新：解决了堆中的引用更新后，还要修正存在于GC Roots中的引用。这个阶段是Shenandoah的最后一次停顿，**停顿时间只与GC Roots的数量相关**。 
- 并发清理：并发回收region的内存空间。

"Brooks Pointers"提出的转发指针（Forwarding Pointer，也常被称为Indirection Pointer）来实现对象移动与用户程序并发的一种解决方案。在原有对象布局结构的最前面统一增加一个新的引用字段，在正常不处于并发移动的情况下，该引用指向对象自己。

Shenandoah在读、写屏障中都加入了额外的转发处理，尤其是使用读屏障的代价，这是比写屏障更大的。

##### 3.6.2 ZGC收集器

JDK11 新加入的垃圾收集器。

**ZGC收集器是一款基于Region内存布局的，（暂时） 不设分代的，使用了读屏障、染色指针和内存多重映射等技术来实现可并发的标记-整理算法的，以低延迟为首要目标的一款垃圾收集器**。

ZGC的Region：

- 小型Region：容量固定为2MB，用于存放小于256KB的小对象。
- 中型Region：容量固定为32MB，用于放置大于等于256KB但小于4MB的对象。
- 大型Region：容量不固定，必须是2MB的整数倍。用于放置4MB以上的对象。

ZGC的并发整理算法：

染色指针技术(Tag Pointer或Version Pointer)。以前如果想在对象存储一些额外数据，通常会放到对象头(比如对象的哈希码、分代年龄、锁记录)。而染色指针是直接把标记信息记载引用对象的指针上。

Linux 64位高18位不能用来寻址，将剩余的46位中高4位提取出来存储标志信息。通过这些标志位，虚拟机可以直接从指针中看到其引用对象的三色标记状态、是否进入了重分配集（即被移动过）、是否只能通过finalize()方法才能被访问到。

缺点：染色指针有4TB的内存限制，不支持32位平台，不支持压缩指针等。

优点：

- **只要有一个空闲Region，就能完成收集**。而Shenandoah需要等到引用更新阶段结束以后才能释放回收集中的Region，这意味着堆中几乎所有对象都存活的极端情况，需要1∶1复制对象到新Region的话，就必须要有一半的空闲Region来完成收集。
- 大幅减少在垃圾收集过程中内存屏障的使用数量，**ZGC未使用任何写屏障，只用到了读屏障（一部分是染色指针的功劳，一部分是ZGC现在还不支持分代收集，天然就没有跨代引用的问题）**。
- **染色指针可以作为一种可扩展的存储结构**，Linux下的64位指针还有前18位并未使用，它们虽然不能用来寻址，却可以通过其他手段用于信息记录。如果开发了这18位，既可以腾出已用的4个标志位，将ZGC可支持的最大堆内存从4TB拓展到64TB，也可以利用其余位置再存储更多的标志。

ZGC过程：

![2](.\13.png)

- 并发标记：与G1、Shenandoah一样，并发标记是遍历对象图做可达性分析的阶段，也会经历类似于G1、Shenandoah的初始标记、最终标记（STOP THE WORLD），**ZGC的标记是在指针上而不是在对象上进行的，标记阶段会更新染色指针中的Marked 0、Marked 1标志位**。
- 并发预备重分配：这个阶段需要根据特定的查询条件统计得出本次收集过程要清理哪些Region。**ZGC划分Region的目的并非为了像G1那样做收益优先的增量回收。相反，ZGC每次回收都会扫描所有的Region，用范围更大的扫描成本换取省去G1中记忆集的维护成本。ZGC的重分配集只是决定了里面的存活对象会被重新复制到其他的Region中，里面的Region会被释放**，而并不能说回收行为就只是针对这个集合里面的Region进行，因为标记过程是针对全堆的。
- 并发重分配：**重分配是ZGC执行过程中的核心阶段，这个过程要把重分配集中的存活对象复制到新的Region上，并为重分配集中的每个Region维护一个转发表（Forward Table），记录从旧对象到新对象的转向关系。得益于染色指针的支持，ZGC收集器能仅从引用上就明确得知一个对象是否处于重分配集之中，如果用户线程此时并发访问了位于重分配集中的对象，这次访问将会被预置的内存屏障所截获，然后立即根据Region上的转发表记录将访问转发到新复制的对象上，并同时修正更新该引用的值，使其直接指向新对象，ZGC将这种行为称为指针的“自愈”（Self-Healing）能力**。
- 并发重映射：**修正整个堆中指向重分配集中旧对象的所有引用**。由于旧引用有"自愈"能力，所以此过程不需要太及时，被合并到了并发标记阶段完成。

G1通过写屏障维护记忆集，才能处理跨代指针，得以实现Region的增量回收。记忆集占用大量的内存空间，写屏障也对正常程序运行造成额外负担。

ZGC没有分代和记忆集，但也限制了能承受的对象分配速率不会太快。(如果想保证对象分配速率，还是需要引入分代收集，让新生对象在一个专门的区域中创建，然后专门针对这个区域进行更频繁、更快的收集)。

ZGC支持“NUMA-Aware”（Non-Uniform Memory Access，非统一内存访问架构）的内存分配。ZGC收集器会优先尝试在请求线程当前所处的处理器的本地内存上分配对象，以保证高效内存访问。Parallel Scavenge也是支持NUMA的(JEP345：G1 以后也会支持)。

#### 3.7 选择合适的垃圾收集器

##### 3.7.1 Epsilon收集器

适用于移动平台或桌面平台的垃圾收集器（自动内存管理子系统）。

##### 3.7.2 收集器的权衡

- 应用程序的主要关注点：如果目标是尽快算出结果选吞吐量。如果是SLA应用，停顿时间影响服务质量，选低延迟，如果是客户端或者嵌入式应用，选合理的内存占用。
- 基础设施：硬件规格，处理器数量，分配内存大小，操作系统等。
- JDK：发行商和版本号，对应《虚拟机规范》的哪个版本。

一般来说，新系统尝试ZGC，其次试试Shenandoah，老系统6G以下的堆内存考虑CMS，大的考虑G1。

##### 3.7.3 虚拟机及垃圾收集器日志 

JDK9+，HotSpot所有功能的日志都收归到了“-Xlog”参数上。

```
-Xlog[:[selector][:[output][:[decorators][:output-options]]]]
```

命令行中最关键的参数是选择器（Selector），它由标签（Tag）和日志级别（Level）共同组成。 

| 目标                                       | JDK9之前                                                     | JDK9+                 |
| ------------------------------------------ | ------------------------------------------------------------ | --------------------- |
| 查看GC基本信息                             | -XX：+PrintGC                                                | -Xlog：gc             |
| 查看GC详细信息                             | -XX：+PrintGCDetails                                         | -X-log：gc*           |
| 查看GC前后的堆、方法区可用容量变化         | -XX：+PrintHeapAtGC                                          | -Xlog：gc+heap=debug  |
| 查看GC过程中用户线程并发时间以及停顿的时间 | -XX：+Print-GCApplicationConcurrentTime 及-XX：+PrintGCApplicationStoppedTime | -Xlog： safepoint     |
| 查看收集器Ergonomics机制                   | -XX：+PrintAdaptive-SizePolicy                               | -Xlog：gc+ergo*=trace |
| 查看熬过收集后剩余对象的年龄分布信息       | -XX：+PrintTenuring-Distribution                             | -Xlog：gc+age=trace   |

 ![2](.\14.png)

 ![2](.\15.png)

 ![2](.\16.png)

##### 3.7.4 垃圾收集器参数总结

垃圾收集相关的常用参数

| UseSerialGC                    | 虚拟机运行在Client模式下的默认值，打开此开关后，使用Serial + Serial Old的收集器组合进行内存回收 |
| ------------------------------ | ------------------------------------------------------------ |
| UseParNewGC                    | 打开此开关后，使用ParNew + Serial Old的收集器组合进行内存回收，在JDK 9后不再支持 |
| UseConcMarkSweepGC             | 打开此开关后，使用ParNew + CMS + Serial Old的收集器组合进行内存回收。Serial Old收集器将作为CMS收集器出现Concurrent Mode Failure失败后的后备收集器使用 |
| UseParallelGC                  | 虚拟机运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge + Serial Old（PS MarkSweep）的收集器组合进行内存回收 |
| UseParallelOldGC               | 打开此开关后，使用Parallel Scavenge + Parallel Old 的收集器组合进行内存回收 |
| SurvivorRatio                  | 新生代中Eden区域与Survivor区域的容量比值，默认为8，代表Eden : Survivor=8:1 |
| PretenureSizeThreshold         | 直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象将直接在老年代分配 |
| MaxTenuringThreshold           | 晋升到老年代的对象年龄。每个对象在坚持过一次Minor GC之后，年龄就增加1，当超过这个参数值时就进入老年代 |
| UseAdaptiveSizePolicy          | 动态调整Java堆中各个区域的大小以及进入老年代的年龄           |
| HandlePromotionFailure         | 是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden和Survivor区的所有对象都存活的极端情况 |
| ParallelGCThreads              | 设置并行GC时进行内存回收的线程数                             |
| GCTimeRatio                    | GC时间占总时间的比率，默认值为99，即允许1%的GC时间。仅使用Parallel Scavenge收集器时生效 |
| MaxGCPauseMillis               | 设置GC的最大停顿时间。仅在使用Parallel Scavenge收集器生效    |
| CMSInitiatingOccupancyFraction | 设置CMS收集器在老年代空间被使用多少后触发垃圾收集，默认值为68%，仅使用CMS收集器时生效 |
| UseCMSCompactAtFullCollection  | 设置CMS收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅在使用CMS收集器时生效，此参数从JDK 9开始废弃 |
| CMSFullGCsBeforeCompaction     | 设置CMS收集器在进行若干次垃圾收集后再启动一次内存碎片整理。仅在使用CMS收集器时生效，此参数从JDK 9开始废弃 |
| UseG1GC                        | 使用G1收集器，这个事JDK 9后的 Server 模式默认值              |
| G1HeapRegionSize=n             | 设置Region的大小，并非最终值                                 |
| MaxGCPauseMills                | 设置G1收集过程目标时间，默认值是200ms，不是硬性条件          |
| G1NewSizePercent               | 新生代最小值，默认值5%                                       |
| G1MaxNewSizePercent            | 新生代最大值，默认值60%                                      |
| ParallelGCThreads              | 用户线程冻结期间并行执行的收集器线程数                       |
| ConcGCThreads=n                | 并发标记、并发整理的执行线程数，对不同的收集器，根据其能够并发的阶段，有不同的含义 |
| InitiatingHeapOccupancyPercent | 设置触发标记周期的Java堆占用率阈值。默认值45%。这里的java堆占比指的是 non_young_capacity_bytes，包括 old + humongous |
| UseShenandoahGC                | Shenandoah何时启动一次GC过程，其可选值有adaptive、static、compact、passive、aggressive |
| UseZGC                         | 使用ZGC收集器，目前仍然要配合 -XX：+UnlockExperimentalVMOptions 使用 |
| UseNUMA                        | 启用NUMA内存分配支持，目前只有Parallel和ZGC支持，以后G1收集器可能也会支持该选项 |

#### 3.8 实战：内存分配与回收策略

##### 3.8.1 对象优先在Eden分配

优先Eden，Eden触发MinorGC，复制到 Survivor,如果Survivor不够，通过分配担保机制提前转移到老年代。

##### 3.8.2 大对象直接进入老年代

-XX：PretenureSizeThreshold  可以设置，大于这个值的参数直接在老年代分配。（只对Serial和ParNew两款新生代收集器有效）

##### 3.8.3 长期存活的对象将进入老年代

-XX： MaxTenuringThreshold设置，超过设置值，就会被晋升到老年代。（默认为15）

用的JDK 8测试的，感觉看不出来...

```
/*** VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails 
 -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 */
public class TestTenuringThreshold {
    public Object instance = null;
    private static final int _1MB = 1024 * 1024;
    
    public static void main(String[] args) {
        testTenuringThreshold();
    }
    
    public static void testTenuringThreshold() {
        byte[] allocation1, allocation2, allocation3;
        allocation1 = new byte[_1MB / 4];
        // 什么时候进入老年代决定于XX:MaxTenuring- Threshold设置 
        allocation2 = new byte[4 * _1MB];
        allocation3 = new byte[4 * _1MB];
        allocation3 = null;
        allocation3 = new byte[4 * _1MB];
    }
}
```

Threshold = 1 的结果：

```
Heap
 PSYoungGen      total 9216K, used 7002K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 85% used [0x00000000ff600000,0x00000000ffcd6b08,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 8192K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 80% used [0x00000000fec00000,0x00000000ff400020,0x00000000ff600000)
 Metaspace       used 3081K, capacity 4592K, committed 4864K, reserved 1056768K
  class space    used 326K, capacity 424K, committed 512K, reserved 1048576K
```

Threshold = 15 的结果：

```
Heap
 PSYoungGen      total 9216K, used 6515K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 79% used [0x00000000ff600000,0x00000000ffc5cea8,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 8192K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 80% used [0x00000000fec00000,0x00000000ff400020,0x00000000ff600000)
 Metaspace       used 3301K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 359K, capacity 388K, committed 512K, reserved 1048576K
```

##### 3.8.4 动态对象年龄判定

并不是只有对象的年龄必须达到- XX：MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。

##### 3.8.5 空间分配担保

在发生Minor GC之前，虚拟机必须**先检查老年代最大可用的连续空间是否大于新生代所有对象总空间**，如果这个条件成立，那这一次Minor GC可以确保是安全的。如果不成立，则虚拟机会**先查看-XX：HandlePromotionFailure参数（JDK 6 Update 24之前）的设置值是否允许担保失败（Handle Promotion Failure）**；如果允许，那会继续**检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小**，如果大于，将尝试进行一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者-XX：HandlePromotionFailure设置不允许冒险，那这时就要改为进行一次Full GC。

取历史平均值来比较其实仍然是一种赌概率的解决办法，也就是说假如某次Minor GC存活后的对象突增，远远高于历史平均值的话，依然会导致担保失败。**如果出现了担保失败，那就只好老老实实地重新发起一次Full GC**，这样停顿时间就很长了。

**JDK 6 Update 24之后的规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小，就会进行Minor GC，否则将进行Full GC**。 

#### 3.9 本章小结

垃圾收集器的选择和使用场景相关，也有很多参数可以调节。只有根据实际应用需求、实现方式选择最优的收集 方式才能获取最好的性能。

