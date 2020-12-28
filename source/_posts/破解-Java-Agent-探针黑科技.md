title: 破解 Java Agent 探针黑科技
date: 2020-12-28 15:54:50
tags: [Java]
---

### 什么是 Java Agent

笼统地来讲，Java Agent 是一个统称，该功能是 Java 虚拟机提供的一整套后门。通过这套后门可以对虚拟机方方面面进行监控与分析。甚至干预虚拟机的运行。

Java Agent 又叫做 Java 探针，Java Agent 是在 JDK1.5 引入的，是一种可以动态修改 Java 字节码的技术。Java 类编译之后形成字节码被 JVM 执行，在 JVM 在执行这些字节码之前获取这些字节码信息，并且通过字节码转换器对这些字节码进行修改，来完成一些额外的功能，这种就是 Java Agent 技术。

从用户使用层面来看，Java Agent 一般通过在应用启动参数中添加 -javaagent 参数添加 ClassFileTransformer 字节码转换器。 在 Java 虚拟机启动时，执 行main() 函数之前，Java  虚拟机会先找到 -javaagent 命令指定 jar 包，然后执行 premain-class 中的 premain() 方法。用一句概括其功能的话就是：main() 函数之前的一个拦截器。

<!-- more -->
### Java Agent 可以实现什么样的功能

从上面提到的字节码转换器的两种执行方式来看可以实现如下功能：

- Java Agent 能够在加载 Java 字节码之前进行拦截并对字节码进行修改; 
- 在 Jvm 运行期间修改已经加载的字节码;

因此，通过以上两点即可实现在一些框架或是技术的采集点进行字节码修改，对应用进行监控（比如通过JVM CPU Profiler 从CPU、Memory、Thread、Classes、GC等多个方面对程序进行动态分析），或是对执行指定方法或接口时做一些额外操作，比如打印日志、打印方法执行时间、采集方法的入参和结果等；

基于前面对 Java Agent 大致机制的描述，我们不难猜到，能够干预 Java JVM 虚拟机的运行，那么就可以解决不限于如下的问题：

- 使用 JVMTI 对 class 文件加密：有时一些涉及到关键技术的 class 文件或者 jar 包我们不希望对外暴露，因而需要进行加密。使用一些常规的手段（例如使用混淆器或者自定义类加载器）来对 class 文件进行加密很容易被反编译。反编译后的代码虽然增加了阅读的难度，但花费一些功夫也是可以读懂的。
- 使用 JVMTI 我们可以将解密的代码封装成 .dll, 或 .so  文件。这些文件想要反编译就很麻烦了，另外还能加壳。解密代码不能被破解，从而也就保护了我们想要加密的 class 文件。使用 JVMTI 实现应用性能监控(APM) 在微服务大行其道的环境下，分布式系统的逻辑结构变得越来越复杂。这给系统性能分析和问题定位带来了非常大的挑战。基于JVMTI的APM能够解决分布式架构和微服务带来的监控和运维上的挑战。APM通过汇聚业务系统各处理环节的实时数据，分析业务系统各事务处理的交易路径和处理时间，实现对应用的全链路性能监测。开源的Skywalking、Pinpoint,、ZipKin、 Hawkular, 商业的 AppDynamics、OneAPM、Google Dapper等都是个中好手。

另外来看看 Github 上有哪些开源工具、项目使用到了 Agent 技术：

- 阿里巴巴开源的 Java 诊断工具—— Arthas，深受开发者喜爱。在线排查问题，无需重启；动态跟踪 Java 代码；实时监控 JVM 状态。
- Apache Skywalking 的 Java Agent 则针对服务的调用链路、JVM 基础监控信息进行采集。
- Uber/jvm-profiler: 通过 Java Agent 采集 JVM CPU、Memory、IO等指标并发送给 Kafka、Console 以及可以自定义的发送器。

### Java Agent 的实现原理

从 JVM 类加载流程来看，字节码转换器的执行方式有两种：一种是在 main 方法执行之前，通过 premain 来实现，另一种是在程序运行中，通过 Attach Api 来实现。

对于 JVM 内部的 Attach 实现，是通过 tools.jar 这个包中的 com.sun.tools.attach.VirtualMachine 以及  VirtualMachine.attach(pid) 这种方式来实现的。底层则是通过 JVMTI 在运行前或者运行时，将自定义的 Agent 加载并和 VM 进行通信。

了解 Java Agent 的实现原理就必须先了解 Java 的类加载机制（这里不做过多介绍），这个是了解 Java Agent 的前提。JVM 在类加载时触发 JVMTI_EVENT_CLASS_FILE_LOAD_HOOK 事件调用添加的字节码转换器完成字节码转换，该过程时序如下：

![java-agent](/images/java-agent1.jpg)

Java Agent 所使用的 Instrumentation 依赖 JVMTI 实现，当然也可以绕过 Instrumentation 直接使用 JVMTI 实现 Agent。因此，JVMTI 与 JDI 组成了 Java 平台调试体系（JPDA）的主要能力。

如果想要深入了解 Java Agent,就得需要了解 JVMTI 以及 JVMTIAgent，下面分别介绍下：

#### JVMTI

JVMTI 是JVM Tool Interface 的缩写，是 JVM 暴露出来给用户扩展使用的接口集合，JVMTI 是基于事件驱动的，JVM每执行一定的逻辑就会调用一些事件的回调接口，这些接口可以给用户自行扩展来实现自己的逻辑。JVMTI是实现 Debugger、Profiler、Monitor、Thread Analyser 等工具的统一基础，在主流 Java 虚拟机中都有实现。

### JVMTIAgent

JVMTI 并不一定在所有的 Java 虚拟机上都有实现，不同的虚拟机的实现也不尽相同。不过在一些主流的虚拟机中，比如 Sun 和 IBM，以及一些开源的如 Apache Harmony DRLVM 中，都提供了标准 JVMTI 实现。

JVMTI 是一套本地代码接口，因此使用 JVMTI 需要我们与 C/C++ 以及 JNI 打交道。事实上，开发时一般采用建立一个 Agent 的方式来使用 JVMTI，它使用 JVMTI 函数，设置一些回调函数，并从 Java 虚拟机中得到当前的运行态信息，并作出自己的判断，最后还可能操作虚拟机的运行态。把 Agent 编译成一个动态链接库之后，我们就可以在 Java 程序启动的时候来加载它（启动加载模式），也可以在 Java 5 之后使用运行时加载（活动加载模式）。

```
-agentlib:agent-lib-name=options
-agentpath:path-to-agent=options
```

JVMTIAgent主要有三个方法，

- Agent_OnLoad 方法，如果 agent 在启动时加载，就执行这个方法 
- Agent_OnAttach方法，如果agent不是在启动的时候加载的，是我们先attach到目标线程上，然后对对应的目标进程发送load命令来加载agent，在加载过程中调用Agent_OnAttach函数 
- Agent_OnUnload 方法，在 agent 做卸载掉时候调用 Instrument Agent说到 javaagent，必须要讲的是一个叫做 instrument 的 JVMTIAgent（Linux下对应的动态库是 libinstrument.so） 

#### instrument agent 

实现了上面 Agent_OnLoad 方法和 Agent_OnAttach 方法，也就是即能在启动的时候加载 agent，也可以在运行期来加动态加载 agent，运行期动态加载 agent 依赖 JVM 的 attach 机制实现，通过发送 load 命令来加载 agent

#### JVM Attach 机制

Jvm attach 机制是指 JVM 提供的一种 JVM 进程间通信的功能，能让一个进程传命令给另一个进程，并进行一些内部的操作，比如进行线程 dump，那么就需要执行 jstack 进行，然后把 pid 等参数传递给需要 dump 的线程来执行，这就是一种 java attach。

### 可以实现 Java Agent 的技术框架有哪些？

原理了解清楚了就需要实现，Java Agent 从实现上来看主要涉及到字节码增强的过程，其到过程大概是：

- 修改字节码
- 加载新的字节码
- 替换旧的字节码

通过上面对 Java Agent 介绍之后，是不是发现，我想要实现一个 Java Agent 还得去深入学习那么多东西吗？

当然不用，这里就介绍几个常用的字节码增强工具：

- ASM：对于需要手动操纵字节码的需求，可以使用 ASM，它可以直接生成 .class 字节码文件，也可以在类被加载入 JVM 之前动态修改类行为。

![java-agent](/images/java-agent2.jpg)

- Javassist： ASM 是在指令层次上操作字节码的，我们的直观感受是在指令层次上操作字节码的框架实现起来比较晦涩。故除此之外，再简单介绍另外一类框架：强调源代码层次操作字节码的框架 Javassist。利用 Javassist 实现字节码增强时，可以无须关注字节码刻板的结构，其优点就在于编程简单。直接使用 Java 编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构或者动态生成类。
- Instrument：Instrument 是 JVM 提供的一个可以修改已加载类的类库，专门为 Java 语言编写的插桩服务提供支持。它需要依赖 JVMTI 的 Attach API 机制实现。
- Byte Buddy：ByteBuddy 是一个开源 Java 库，其主要功能是帮助用户屏蔽字节码操作，以及复杂的 InstrumentationAPI。ByteBuddy 提供了一套类型安全的API和注解，我们可以直接使用这些 API 和注解轻松实现复杂的字节码操作。另外，Byte Buddy 提供了针对 Java Agent 的额外 API，帮助开发人员在 Java Agent 场景轻松增强已有代码。

本文引自[知乎](https://zhuanlan.zhihu.com/p/135872794)


