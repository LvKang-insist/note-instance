### 前言

**Android 系统对每个app都会有一个最大的内存限制，如果超出这个限制，就会抛出 OOM，也就是Out Of Memory** 。本质上是抛出的一个异常，一般是在内存超出限制之后抛出的。最为常见的 OOM 就是内存泄露(大量的对象无法被释放)导致的 OOM，或者说是需要的内存大小大于可分配的内存大小，例如加载一张非常大的图片，就可能出现 OOM。

### 常见的 OOM 

#### 堆溢出

堆内存溢出是最为常见的 OOM ，**通常是由于堆内存已经满了，并且不能够被垃圾回收器回收**，从而导致 OOM。

#### 线程溢出

不同的手机允许的最大线程数量是不一样的，在有些手机上这个值被修改的非常低，就会比较容易出现线程溢出的问题

#### FD数量溢出

文件描述符溢出，当程序打开或者新建一个文件的时候，系统会返回一个索引值，指向该进程打开文件的记录表，例如当我们用输出流文件打开文件的时候，系统就会返回我们一个FD，FD是可能出现泄露的，例如输入输出流没有关闭的时候，[详细可参考 Android FD泄露问题](https://blog.csdn.net/ws6013480777777/article/details/84594116)

#### 虚拟内存不足

在新建线程的时候，底层需要创建 JNIEnv 对象，并且分配虚拟内存，如果虚拟内存耗尽，会导致创建线程失败，并抛出 OOM。



### Jvm,Dvm,Art的内存区别

Android 中使用的是基于 Java 语言的虚拟机 Dalvik / ART ，而 **Dalvik 和 ART 都是基于 JVM** 的，但是需要注意的是 Android 中的 虚拟器和标准的 JVM 有所不同，因为它们需要运行在 Android 设备上，因此他们具有不同的优化和限制。

在回收方面，Dalvik 仅固定一种回收算法，而 ART 回收算法可在运行期按需选择，并且ART 具备内存整理能力，减少内存空洞。

#### JVM

JVM 是一个虚构出来的计算机，是通过在实际计算机上仿真各种计算机功能来实现的，它有完善的(虚拟)硬件架构，还有相应的指令系统，其指令集基于堆栈结构。使用 `Java虚拟机`就是为了支持系统操作无关，在任何系统中都可以运行的程序。

JVM 将所管理的内存分为以下几个部分:
![image-20230512152005731](/Users/tidycar/Desktop/202305121520882.png)

- 方法区

    **各个线程锁共享的，用于存储已经被虚拟机加载的类信息，常量，静态变量等**，当方法区无法满足内存分配需求时，将会抛出 OutOfMemoryError 异常。

    - 常量池

        常量池也是方法区的一部分，用于存放编译器生成的各种自变量和符号引用,用的最多的就是 String，当 new String 并调用intern 时，就会在常量池查看是否有该字符串，有则返回，没有则创建一个并返回。

- Java 堆

    虚拟机内存中最大的一块内存，**所有通过 new 创建的对象都会在堆内存进行分配，是虚拟机中最大的一块内存，也是gc需要回收的部分，同时OOM也容易发生在这里**。

    从内存回收角度来看，由于现在收集器大都采用分代收集法，所以还可以细分为新生代，老年代等。

    根据 Java 虚拟机规定，Java 堆可以处于物理上不连续的空间，只要逻辑上是连续的就行，如果对中没有可分配内存时，就会出现 OutOfMemoryError 异常

- Java 栈

    **线程私有**，用来存放 java 方法执行时的所有数据，由栈贞组成，一个栈贞就代表一个方法的执行，每个方法的执行就相当于是一个栈贞在虚拟机中从入栈到出栈的过程。栈贞中主要包括，局部变量，栈操作数，动态链接等。

    Java 栈划分为操作数栈，栈帧数据和局部变量数据，方法中分配的局部变量在栈中，同时每一次方法的调用都会在栈中奉陪栈帧，栈的大小是把双刃剑，分配太小可能导致栈溢出，特别是在有递归，大量的循环操作的时候。如果太大就会影响到可创建栈的数量，如果是多线程应用，就会导致内存溢出。

- 本地方法栈

    与 java 栈的效果基本类似，区别只不过是用来服务于 native 方法。

- 程序计数器

    是一块较小的空间，它的作用可以看做是当前线程锁执行字节码的行号指示器，用于记录线程执行的字节码指令地址，使得线程切换时能够恢复到正确的执行位置。

#### DVM

原名 Dalvik 是 Google 公司自己设计用于 Android 平台的虚拟机，**本质上也是一个 JAVA 虚拟机，是 Android 中 Java 程序运行的基础**，其指令基于寄存器架构，执行其特有的文件格式-dex。

##### DVM 运行时堆

DVM 的堆结构和 JVM 的堆结构有所区别，主要体现在**将堆分成了 Active 堆 和 Zygote 堆**。Zygote 是一个虚拟机进程，同时也是一个虚拟机实例孵化器，zygote 堆是 Zygote 进程在启动时预加载的类，资源和对象，除此之外我们在代码中创建的实例，数组等都是存储在 Active 堆中的。

为什么要将 Dalvik 堆分为两块，主要是因为 Android 通过 fork 方法创建一个新的 zygote 进程，为了尽量避免父进程和子进程之间的数据拷贝。

Dalvik 的 **Zygote 对存放的预加载类都是 Android 核心类和 Java 运行时库，这部分很少被修改，大多数情况下子进程和父进程共享这块区域，因此这部分类没有必要进行垃圾回收**，而 Active 作为程序代码中创建的实例对象的堆，是垃圾回收的重点区域，因此需要将两个堆分开。

##### DVM 回收机制

DVM 的垃圾回收策略默认是标记清除算法(mark-and-sweep)，基本流程如下

1. 标记阶段：从根对象开始遍历，标记所有可达对象，将它们标记为非垃圾对象
2. 清楚阶段：遍历整个堆，将所有未被标记的对象清除
3. 压缩阶段(可选)：将所有存货的对象压缩到一起，以便减少内存碎片

> 需要注意的是 DVM 垃圾回收器是基于标记清除算法的，这种算法会产生内存算法，可能会导致内存分配效率降低，因此 DVM 还支持分代回收算法，可以更好的处理内存碎片问题。
>
> 在分代垃圾回收中，内存被分为不同的年代，每个年代使用不同的垃圾回收算法进行处理，年轻代使用标记复制算法，老年代使用标记清除法，这样可以更好的平衡内存分配效率和垃圾回收效率



#### ART

ART 是在 Android 5.0 中引入的虚拟机，与 DVM 相比，**ART 使用的是 AOT(Ahead of Time) 编译技术**，这意味着他将应用程序的字节码转换为本机机器码，而不是在运行时逐条解释字节码，这种编译技术可以提高应用程序的执行效率，减少应用程序启动时间和内存占用量

##### JIT 和 AOT 区别

- Just In Time

    DVM 使用 JIT 编译器，每次应用运行时，它实时的将一部分 dex 字节码翻译成机器码。在程序的执行过程中，更多的代码被编译缓存，由于 JIT 只翻译一部分代码，它消耗更少的内存，占用更少的物理内存空间

- Ahead Of Time

    ART 内置了一个 AOT 编译器，在应用安装期间，她将 dex 字节码编译成机器码存储在设备的存储器上，这个过程旨在应用安装到设备的时候发生，由于不在需要 JIT 编译，代码的执行速度回快很多

##### ART运行时堆

与 DVM 不同的是，ART 采用了多种垃圾收集方案，每个方案会运行不同的垃圾收集器，默认是采用了 CMS (Concurrent Mark-Sweep) 方案，也就是并发标记清除，该方案主要使用了 sticky-CMS 和 partial-CMS。根据不同的方案，ART 运行时堆的空间也会有不同的划分，默认是由四个区域组成的。

分别是 Zygote、Active、Image 和 Large Object 组成的，其中 Zygote 和 Active 的作用越 DVM 中的作用是一样的，Image 区域用来存放一些预加载的类，Large Object 用来分配一下大对象(默认大小为12kb)，其中 Zygote 和 Image 是进程间共享的，

### LMK 内存管理机制

LMK(Low Memory Killer) 是 Android 系统内存管理机制的一部分，LMK 是在内存不足时释放系统中不必要的进程，以保证系统的正常运行。

LMK 机制的底层原理是利用内核 OOM 机制来管理内存，当系统内存不足时，内核会根据各个进程优先级将内存优先分配给重要的进程，同时会结束一下不重要的进程，避免系统崩溃。

LKM 机制的使用场景包括：

- 系统内存不足：LMK 机制帮助系统管理内存，以保证系统正常运行
- 内存泄露：当应用存在内存泄露时，LMK 会将泄露的内存释放掉，以保证系统正常运行
- 进程优化：帮助系统管理进程，以确保系统资源的合理利用

在系统内存紧张的情况下，LMK 机制可以通过结束不重要的进程来释放内存，以保证系统正常运行。

但是如果不当使用，它也可能导致应用程序的不稳定。

### 为什么会出现 OOM？

出现 OOM 是应为 **Android 系统对虚拟机的 heap 做了限制，当申请的空间超过这个限制时，就会抛出 OOM**，这样做的目的是为了让系统能同时让比较多的进程常驻于内存，这样程序启动时就不用每次都重新加载到内存，能够给用户更快的响应

#### Android 获取可分配的内存大小

```kotlin
val manager = getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
manager.memoryClass
```

返回当前设备的近似每个应用程序内存类。这让你知道你应该对应用程序施加多大的内存限制，让整个系统工作得最好。返回值以兆字节为单位; 基线Android内存类为16 (恰好是这些设备的Java堆限制); 一些内存更多的设备可能会返回24甚至更高的数字。

我使用的手机内存是 16 g，调用返回的是 256Mb，

> manager.memoryClass 对应 build.prop 中 dalvik.vm.heapgrowthlimit 

#### 申请更大的堆内存

```kotlin
val manager = getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
manager.largeMemoryClass
```

可分配的最大对内存上限，**需要在 manifest 文件中设置 android:largeHeap="true" 方可启用**

> manager.largeMemoryClass 对应 build.prop 中 dalvik.vm.heapsize

#### Runtime.maxMemory

获取进程最大可获取的内存上限，等于上面这两个值之一



#### /system/build.prop

该目录是Android内存配置相关的文件，里面保存了系统的内存的限制等数据，执行 adb 命令可看到 Android 配置的内存相关信息：

```
adb shell
cat /system/build.prop
```

默认是打不开的，没有权限，需要 root

打开后找到 dalvik.vm 相关的配置

```
dalvik.vm.heapstartsize=5m	#单个应用程序分配的初始内存
dalvik.vm.heapgrowthlimit=48m	#单个应用程序最大内存限制，超过将被Kill，
dalvik.vm.heapsize=256m  #所有情况下（包括设置android:largeHeap="true"的情形）的最大堆内存值，超过直接oom。
```

未设置android:largeHeap="true"的时候，只要申请的内存超过了heapgrowthlimit就会触发oom，而当设置android:largeHeap="true"的时候，只有内存超过了heapsize才会触发oom。heapsize已经是该应用能申请的最大内存（这里不包括native申请的内存）。

### OOM 演示

#### 堆内存分配失败

堆内存分配失败对应的是 /art/runtime/gc/heap.cc ，如下代码

```c++
oid Heap::ThrowOutOfMemoryError(Thread* self, size_t byte_count, AllocatorType allocator_type) {
  // If we're in a stack overflow, do not create a new exception. It would require running the
  // constructor, which will of course still be in a stack overflow.
  if (self->IsHandlingStackOverflow()) {
    self->SetException(
        Runtime::Current()->GetPreAllocatedOutOfMemoryErrorWhenHandlingStackOverflow());
    return;
  }
  //....
  std::ostringstream oss;
  size_t total_bytes_free = GetFreeMemory();
  oss << "Failed to allocate a " << byte_count << " byte allocation with " << total_bytes_free
      << " free bytes and " << PrettySize(GetFreeMemoryUntilOOME()) << " until OOM,"
      << " target footprint " << target_footprint_.load(std::memory_order_relaxed)
      << ", growth limit "
      << growth_limit_;
  
  self->ThrowOutOfMemoryError(oss.str().c_str());
}
```

通过上面的分析，我们也知道系统对每个应用都做了最大内存的约束，超过这个值就会 OOM ，下面通过一段代码来演示一下这种类型的 OOM

```kotlin
fun testOOM() {
    val manager = requireContext().getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
    Timber.e("app maxMemory ${manager.memoryClass} Mb")
    Timber.e("large app maxMemory ${manager.largeMemoryClass} Mb")
    Timber.e("current app maxMemory ${Runtime.getRuntime().maxMemory() / 1024 / 1024} Mb")
    var count = 0
    val list = mutableListOf<ByteArray>()
    while (true) {
        Timber.e("count $count    total ${count * 20}")
        list.add(ByteArray(1024 * 1024 * 20))
        count++
    }
}
```

上面代码中每次申请 20mb，测试分为两种情况，

1. 未开启 largeHeap:

    ```text
     E  app maxMemory 256 Mb
     E  large app maxMemory 512 Mb
     E  current app maxMemory 256 Mb
     E  count 0    total 0
     E  count 1    total 20
     E  count 2    total 40
     E  count 3    total 60
     E  count 4    total 80
     E  count 5    total 100
     E  count 6    total 120
     E  count 7    total 140
     E  count 8    total 160
     E  count 9    total 180
     E  count 10    total 200
     E  count 11    total 220
     E  count 12    total 240
    java.lang.OutOfMemoryError: Failed to allocate a 20971536 byte allocation with 12386992 free bytes and 11MB until OOM, target footprint 268435456, growth limit 268435456
    ......
    ```

    可以看到一共分配了 12次，在第十二次的时候抛出了异常，显示 分配 20 mb 失败，空闲只有 11 mb，

2. 开启 largeHeap

    ```
    app maxMemory 256 Mb                      
    large app maxMemory 512 Mb
    current app maxMemory 512 Mb
    E  count 0    total 0
    E  count 1    total 20
    E  count 2    total 40
    E  count 3    total 60
    E  count 4    total 80
    E  count 5    total 100
    E  count 6    total 120
    E  count 7    total 140
    E  count 8    total 160
    E  count 9    total 180
    E  count 10    total 200
    E  count 11    total 220
    E  count 12    total 240
    E  count 13    total 260
    E  count 14    total 280
    E  count 15    total 300
    E  count 16    total 320
    E  count 17    total 340
    E  count 18    total 360
    E  count 19    total 380
    E  count 20    total 400
    E  count 21    total 420
    E  count 22    total 440
    E  count 23    total 460
    E  count 24    total 480
    E  count 25    total 500
    FATAL EXCEPTION: main
    Process: com.dzl.duanzil, PID: 31874
    java.lang.OutOfMemoryError: Failed to allocate a 20971536 byte allocation with 8127816 free bytes and 7937KB until OOM, target footprint 536870912, growth limit 536870912
    ```

    可以看到分配了25 次，可使用的内存也增加到了 512 mb

#### 创建线程失败

线程创建会消耗大量的内存资源，创建的过程涉及 java 层 和 native 层，本质上是在 native 层完成的，对应的是 /art/runtime/thread.cc ，如下代码

```c++
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  //........
  env->SetLongField(java_peer, WellKnownClasses::java_lang_Thread_nativePeer, 0);
  {
    std::string msg(child_jni_env_ext.get() == nullptr ?
        StringPrintf("Could not allocate JNI Env: %s", error_msg.c_str()) :
        StringPrintf("pthread_create (%s stack) failed: %s",
                                 PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    ScopedObjectAccess soa(env);
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());
  }
}
```

这里借用网上的一张照片来看一下创建线程的流程

<img src="/Users/tidycar/Desktop/202305221618788.jpg" alt="2841684743512_.pic" style="zoom:50%;" />

根据上图可以看到主要有两部分，分别是创建 JNI Env 和 创建线程

##### 创建 JNI Env 失败

1. FD 溢出导致 JNIEnv 创建失败

    ```kotlin
    E/art: ashmem_create_region failed for 'indirect ref table': Too many open files java.lang.OutOfMemoryError:Could not allocate JNI Env at java.lang.Thread.nativeCreate(Native Method) at java.lang.Thread.start(Thread.java:730)
    ```

2. 虚拟内存不足导致 JNIEnv 创建失败

    ```kotlin
    E OOM_TEST: create thread : 1104
    W com.demo: Throwing OutOfMemoryError "Could not allocate JNI Env: Failed anonymous mmap(0x0, 8192, 0x3, 0x22, -1, 0): Operation not permitted. See process maps in the log." (VmSize 2865432 kB)
    E InputEventSender: Exception dispatching finished signal.
    E MessageQueue-JNI: Exception in MessageQueue callback: handleReceiveCallback
    MessageQueue-JNI: java.lang.OutOfMemoryError: Could not allocate JNI Env: Failed anonymous mmap(0x0, 8192, 0x3, 0x22, -1, 0): Operation not permitted. See process maps in the log.
    E MessageQueue-JNI:      at java.lang.Thread.nativeCreate(Native Method)
    E MessageQueue-JNI:      at java.lang.Thread.start(Thread.java:887)
    
    E AndroidRuntime: FATAL EXCEPTION: main
    E AndroidRuntime: Process: com.demo, PID: 3533
    E AndroidRuntime: java.lang.OutOfMemoryError: Could not allocate JNI Env: Failed anonymous mmap(0x0, 8192, 0x3, 0x22, -1, 0): Operation not permitted. See process maps in the log.
    E AndroidRuntime:        at java.lang.Thread.nativeCreate(Native Method)
    E AndroidRuntime:        at java.lang.Thread.start(Thread.java:887)
    ```

##### 创建线程失败

1. 虚拟机内存不足导致失败

    native 通过 FixStackSize 设置线程大小

    ```c++
    static size_t FixStackSize(size_t stack_size) {
      if (stack_size == 0) {
        stack_size = Runtime::Current()->GetDefaultStackSize();
      }
      stack_size += 1 * MB;
      if (kMemoryToolIsAvailable) {
        stack_size = std::max(2 * MB, stack_size);
      }  if (stack_size < PTHREAD_STACK_MIN) {
        stack_size = PTHREAD_STACK_MIN;
      }
      if (Runtime::Current()->ExplicitStackOverflowChecks()) {
        stack_size += GetStackOverflowReservedBytes(kRuntimeISA);
      } else {
        stack_size += Thread::kStackOverflowImplicitCheckSize +
            GetStackOverflowReservedBytes(kRuntimeISA);
      }
      stack_size = RoundUp(stack_size, kPageSize);
      return stack_size;
    }
    ```

    ```kotlin
    W/libc: pthread_create failed: couldn't allocate 1073152-bytes mapped space: Out of memory
    W/tch.crowdsourc: Throwing OutOfMemoryError with VmSize  4191668 kB "pthread_create (1040KB stack) failed: Try again"
    java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Try again
            at java.lang.Thread.nativeCreate(Native Method)
            at java.lang.Thread.start(Thread.java:753)
    ```

2. 线程数量超过限制

    用一段简单的代码来测试一下

    ```kotlin
    fun testOOM() {
        var count = 0
        while (true) {
            val thread = Thread(Runnable {
                Thread.sleep(1000000000)
            })
            thread.start()
            count++
            Timber.e("current thread count $count")
        }
    }
    ```

    通过打印日志发现，一共创建了 2473 个线程，当然这些线程都是没有任务的线程，报错信息如下所示

    ```kotlin
    pthread_create failed: couldn't allocate 1085440-bytes mapped space: Out of memory
    Throwing OutOfMemoryError "pthread_create (1040KB stack) failed: Try again" (VmSize 4192344 kB)
    
    FATAL EXCEPTION: main
    Process: com.dzl.duanzil, PID: 18085
    java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Try again
    	at java.lang.Thread.nativeCreate(Native Method)
    ```

    通过测试可以看出来，具体的原因也是内存不足引起的，而不是线程数量超过限制，可能是。

    

### OOM 监控

我们都知道，OOM 的出现就是大部分原因都是由于内存泄露，导致内存无法释放，才出现了 OOM，所以监控主要监控的是内存泄露，现在市面上对于内存泄露检查这方面已经非常成熟了，我们来看几个常用的监控方式

#### LeakCanary

使用非常简单，只需要添加依赖后就可以直接使用，无需手动初始化，就能实现内存泄露检测，当内存发生泄露后，会自动发出一个通知，点击就可以查看具体的泄露堆栈信息

LeakCannary 只能在 debug 环境使用，因为他是在当前进程 dump 内存快照，会冻结当前进程一段时间，所以不适于在正式环境使用。

#### Android Profile

可以以图像的方式直观的查看内存使用情况，并且可以直接 capture heap dump，或者抓取原生内存(C/C++)  以及 Java/Kotlin 内存分配。只能在线下使用，功能非常强大，可是吧内存泄露，抖动，强制 GC 等。

#### ResourceCanary

ResourceCanary 属于 Matrix 的一个子模块，它将原本难以发现的 Acivity 泄露和 Activity 泄露和重复创建的沉余的 Bitmap 暴露出来，并提供引用链等信息帮助排查这些问题

ResourceCanary 将检测和分析分离，客户端只负责检测和dump内存镜像文件，并且对检查部分生成的 Hprof 文件进行了裁剪，移除了大部分无用数据。也增加了 Bitmap 对象检测，方便通过减少沉余 Bitmap 数量，降低内存消耗。

使用可查看 [Matrix](https://github.com/Tencent/matrix#matrix_android_cn)

#### KOOM

上面的两者都只能在线下使用，而 KOOM 可以再线上使用，KOOM 是快手出的一套完整的解决方案，可以实现 Java，native 和 thread 的泄露监控

使用可查看 [KOOM](https://github.com/KwaiAppTeam/KOOM/blob/master/README.zh-CN.md)

### 优化方向

#### 图片优化

图片所占用的内存其实和图片的大小是没有太大关系的，主要取决于图片的宽高以及图片的加载方式，例如 ARGB__8888 就比 RGB_565 所占用的内存大了一倍，要计算图片占用了多大内存，可查看这篇文章 **[计算图片占用内存大小](https://juejin.cn/post/7085372918602924068)**，下面介绍几种图片的优化方式

- 统一图片库

    在项目中，应该避免使用多种图片库，多种图片库会导致图片的重复缓存等一系列的问题

- 图片压缩

    对于一些图片资源文件，可以再添加到项目中的时候对图片进行压缩处理，这里推荐一个插件 **[CodeLocator](https://github.com/bytedance/CodeLocator)** ，该插件在前一段时间好像用不了了，这里在推荐一个插件 **[McImage](https://github.com/smallSohoSolo/McImage)**，在打包的时候回对图片进行压缩处理。

    使用方式如下

    ```groovy
    buildscript {
        repositories {
            mavenCentral()
        }
        dependencies {
            classpath 'com.smallsoho.mobcase:McImage:1.5.1'
        }
    }
    ```

    然后在你想要压缩的Module的build.gradle中应用这个插件，注意如果你有多个Module，请在每个Module的build.gradle文件中apply插件

    ```groovy
    apply plugin: 'McImage'
    ```

    最后将我代码中的mctools文件夹放到项目根目录，此文件在[这里下载](https://github.com/Mobcase/McImage/releases)

    ```
    mctools
    ```

    配置

    你可以在build.gradle中配置插件的几个属性，如果不设置，所有的属性都使用默认值

    ```groovy
    McImageConfig {
            isCheckSize true //是否检测图片大小，默认为true
            optimizeType "Compress" //优化类型，可选"ConvertWebp"，"Compress"，转换为webp或原图压缩，默认Compress，使用ConvertWep需要min sdk >= 18.但是压缩效果更好
            maxSize 1*1024*1024 //大图片阈值，default 1MB
            enableWhenDebug false //debug下是否可用，default true
            isCheckPixels true // 是否检测大像素图片，default true
            maxWidth 1000 //default 1000 如果开启图片宽高检查，默认的最大宽度
            maxHeight 1000 //default 1000 如果开启图片宽高检查，默认的最大高度
            whiteList = [ //默认为空，如果添加，对图片不进行任何处理
    
            ]
            mctoolsDir "$rootDir"
            isSupportAlphaWebp false  //是否支持带有透明度的webp，default false,带有透明图的图片会进行压缩
            multiThread true  //是否开启多线程处理图片，default true
            bigImageWhiteList = [
                    "launch_bg.png"
            ] //默认为空，如果添加，大图检测将跳过这些图片
    }
    ```

- 图片监控

    通过对 app 中所有的图片监控，实时查看图片占用的内存大小，从而发现图片大小是否合适，是否存在泄露等，推荐一个分析工具 **[AndroidBitmapMonitor](https://github.com/shixinzhang/AndroidBitmapMonitor/blob/master/README_CN.md)**，该工具可获取内存中图片的数量及占用大小，并且可以获取 Bitmap 的创建堆栈等。

    另外，也可以通过自定义实现监控，通过监听 setImageDrawable 等方法，获取图片的大小以及 ImageView 本身的大小，然后再判断提示是否需要修改即可，具体方案如下：

    1. 自定义 ImageView，重写 setImageResource ，setBackground 等方法，在里面检测图片的大小
    2. 使用自定义的 ImageView，如果挨个替换肯定不现实，这里提供两种方式，第一种，在编译期修改字节码，将所有的 ImageView 都替换为自定义的 ImageView，第二种，重写 Layoutinflat.Fractory2，在创建 View 的时候将 ImageVIew 替换为自定义的 ImageView，可封装一下，一键式替换。
    3. 检查的时候可以异步检测，或者在主线程空闲的时候检查。

- 优化加载方式

    一般情况下，我们加载图片都使用的是 Glide 或者别的图片加载库，就比如 Glide 默认的图片加载格式是 ARGB_8888，对于这种格式，每个像素需要占用四个字节，对于一些低端机型，为了降低内存的占有率，可以修改图片的加载格式为 RGB_565 ，相比于 ARGB_8888 每个像素只有两个字节，故可以节省一半的内存，代价就是少了透明通道，对于不需要透明通道的图片来说，使用这种方式加载无疑是更好的选择。

    ```kotlin
    fun loadRGB565Image(context: Context, url: String?, imageView: ImageView) {
        Glide.with(context)
            .load(url)
            .apply(RequestOptions().format(DecodeFormat.PREFER_RGB_565))
            .into(imageView)
    }
    ```

    也可以指定 Bitmap 的格式

    ```kotlin
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inPreferredConfig = Bitmap.Config.RGB_565;
    ```

- 设置图片采样率

    ```cpp
    Options options = new BitmapFactory.Options();
    options.inSampleSize = 5; // 原图的五分之一，设置为2则为二分之一
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(),
                           R.id.myimage, options);
    ```

    inSampleSize 值越大，图片质量越差，需谨慎设置

#### 内存泄露优化

如果内存在规定的生命周期内没有被释放掉，就会被认为是内存泄露，而内存泄露的次数多了，就会占用很大一部分内存，从而导致新对象申请不到可用的内存，最终就会出现 OOM。

如何观测内存泄露在上文中已经提出了解决的办法，这里主要说一下在日常开发过程中，如何规避一些常见的内存泄露

1. 内存抖动

    内存抖动主要说的是在一段时间内频繁的创建对象，而回收的速度更不上创建的速度，并且导致了很多不连续的内存片。

    通过 Profiler 观察，如果频繁的出现 GC ，内存曲线呈锯齿状，就极有可能发生内存抖动，如下图所示：

    ![image-20230526152913242](/Users/tidycar/Desktop/202305261529368.png)

​		出现这种情况肯定就是频繁的创建对象而导致的，例如：在 onDraw 中频繁的创建重复对象，在循环中不断创建局部变量等等，这些都是可以在开发中直接避免的问题，以及使用缓存池，手动释放缓存池中的对象，多进行复用等等。

2. 各种常见的泄露方式

    - 集合泄露

    - 单例泄露

    - 匿名内部类泄露

        静态匿名内部类 和外部类的关系：**如果没有传入参数就没有引用关系，被调用是不需要外部类的实例**，不能调用外部类的方法和变量，拥有自主的生命周期

        非静态匿名内部类 和外部类的关系，**自动获取外部类的强引用，被调用时需要外部类实例**，可以调用外部类的方法和变量，依赖于外部类，甚至比外部类更长。

    - 资源文件未关闭造成的泄露

        1. 主序广播
        2. 关闭输入输出流
        3. 回收 Bitmap
        4. 停止销毁动画
        5. 销毁 WebView
        6. 及时注销 eventBus 以及需要注销的组件等

    - Handler 早餐内存泄露

        如果 Handler 中有延时任务或者等待的任务队列过长，都有可能因为 Handler 的继续执行从而导致内存泄露

        解决方法：静态内部类+弱引用

        ```java
        private static class MyHalder extends Handler {
        
        		private WeakReference<Activity> mWeakReference;
        
        		public MyHalder(Activity activity) {
        			mWeakReference = new WeakReference<Activity>(activity);
        		}
        
        		@Override
        		public void handleMessage(Message msg) {
        			super.handleMessage(msg);
        			//...
        		}
        	}
        	最后在Activity退出时，移除所有信息
        	移除信息后，Handler 将会跟Activity生命周期同步
        	
        	@Override
        	protected void onDestroy() {
        		super.onDestroy();
        		mHandler.removeCallbacksAndMessages(null);
        	}
        }
        ```

    - 多线程导致内存泄露
        1. 使用匿名内部类启动的线程默认会持有外部对象的引用
        2. 线程数量超过限制导致泄露，这种情况可以使用线程池。



#### 内存监控及兜底策略

使用上面介绍的几种方式来实现在线上/线下对内存的监控。

也可以自己起一个线程，定时的去监控实时的内存使用情况，如果内存告急，可以清理 Glide 等一些占用内存较大的缓存来救急

使用 Activity 兜底策略，在 BaseActivity 的 onDestory 做一些 View 的监听移除，背景置 null 等等。





### 最后

本文到此结束，文章中的内容也是通过积累和查看阅读资料找到的，当然本文所讲的优化方式也是最基础最简单的，如果有任何问题可直接留言或者私信。

都看到这了，就动动发财的小手点个赞呗！



### 参考链接

[【性能优化】大厂OOM优化和监控方案](https://juejin.cn/post/7074762489736478757)

[深入探索 Android 内存优化上](https://juejin.cn/post/6844904099998089230)

[深入探索 Android 内存优化下](https://juejin.cn/post/6872919545728729095)

[DVM和ART原理初探](https://juejin.cn/post/6844903480520359944)

[Android OOM 问题探究](https://www.cnblogs.com/roger-yu/p/16599262.html)

[内存优化 · 基础论 · 初识Android内存优化](https://juejin.cn/post/7198826344582037562)÷

....
