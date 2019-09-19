# C11-atomic-operations-in-detail
C11标准的原子操作详解

<br />

当前现代处理器的发展在性能上已经很难跟得上摩尔定律了，各大处理器设计商都在多核心上做文章，从而多核多线程编程则成为了高性能计算所必要的手段。那么在多核多线程环境下与经典的单处理器多线程环境下的多线程同步有何区别呢？

## 传统单处理器多线程的多线程同步

在传统单处理器环境下，由于处理器就一个核心在运转，因此一次只能执行一个线程，然后通过**时间片轮询（Round-Robin）**与**优先级抢占（Priority Preemption）**方式做线程调度。无论是哪种方式，操作系统对多线程的调度都是基于**中断机制**执行上下文切换，然后选择当前优先级高的、排在**就绪**队列中最前面的线程调度执行。因此，我们看到经典的操作系统**同步原语（Synchronization Primitive）**，诸如**互斥体（Mutex）**、**信号量（Semaphore）**、**消息队列（Message Queue）** 等等均是基于抑制软硬件中断或上下文切换来做到多线程同步的。

<br />

## 基于多核处理器的多线程同步

在多核处理器或多个处理器的环境下，多线程同步则更为复杂。因为对于一般通用操作系统而言，核心A是不知道核心B正在处理啥的，尽管对于某些嵌入式实时操作系统（RTOS）而言，一个核心可以给另一个核心发送信号以通知某些事件，但这些均用于特定应用场景，而通用应用级操作系统不会采用这些同步手段，代价太高。正因为如此，在多核多处理器环境下，如果我们要对多线程所共享的一个对象的修改做到数据一致性，那么只能通过原子操作指令！

为何原子操作能起作用？因为原子操作指令通过锁总线等手段，确保多线程对同一共享对象的**读-修改-写**是原子的。所谓“原子（atomic）”就是指，一组操作在执行时作为一个整体进行，而不会被打断。这里举一个简单的例子来说明对某一共享对象的修改采用原子操作与不采用原子操作的差别。
![non-atomic-vs-atomic](https://github.com/zenny-chen/C11-atomic-operations-in-detail/blob/master/non-atomic-vs-atomic.png)
上图中，假定线程A与线程B在不同的核心上并行执行。第一段采用的是普通读写方式对共享存储单元进行修改的，而第二种则使用了原子的“读-修改-写”操作。第一段所采用的普通读写方式会出什么问题呢？从时序图上我们可以看到，假定线程A与线程B几乎同时先对共享存储单元进行加载（读取），然后再几乎同时进行计算。由于线程A计算速度比较快，它先将计算好的结果存储（写入）到了共享存储单元中；而此后，线程B才计算好，再将它计算好的结果存储到该共享单元中，这就导致了数据不一致！因为线程B最终所写回的结果是基于一开始线程A对该共享存储单元修改之前的，它这么一改就把线程A对共享存储单元的操作给抹掉了。因此这里正确的做法应该是线程B必须基于线程A所修改完的结果再对共享存储单元进行操作。
第二段采用原子操作则不会有数据不一致问题，无论是线程A先对共享存储单元修改完还是线程B先修改完，最终结果都是经过这两个线程的计算操作的。CPU系统总线仲裁器会去裁决哪个先执行，哪个后执行，而后执行的那个会被阻塞（等待）直到先前的操作完成。

因此，对于基于多核多处理器的多线程同步而言，原子操作不是有没有必要用的问题，而是必须得用！上述所提到的那些个经典的同步原语在多核多处理器环境下都用到了原子操作。

<br />

## 原子操作的种类

原子操作有许多种，有纯粹用于做同步的（即作为锁的用途），有用于做基本运算操作的，也有可将这两者相结合的。
对于早期的多核处理器，有不少提供了**数据交换（swap）**、**标志测试与设置（flag test and set）** 等基于“ **锁** ”的原子操作。比如8086上的XCHG指令，ARMv7架构之前的SWP指令，这些都属于SWAP原子操作；而Blackfin 561 Duo-Core DSP上则提供了flag test and set原子操作……这些原子操作的实现比较简单，不过都是基于“**锁**”，也就是说如果你要用原子操作来同步某一共享存储对象，那么必须先针对它定义一个原子变量作为锁去同步。这些原子操作所引发的最大问题就是如果某一线程在利用这些锁做“自旋等待”，而另一个线程在释放该原子锁之前就被销毁了，那么等待该锁的线程就倒霉成为僵尸线程了～尽管一般对于应用层来说，我们不会轻易自己去杀线程，但对于操作系统层还有学术界而言，这是一个必须解决的问题。目前一般常见的解决方案就是添加一个重试次数，如果重试了比如1000次，这个原子锁还没有被释放，那么就强行解锁，或者抛出异常等。所以这里要提醒各位的是，用了原子锁之后，后续相关的操作得尽量快，然后马上释放锁，否则的话宁可直接调用系统所提供的同步原语API。下面给出一份使用SWAP原子操作流程的伪代码，当然flag test and set跟这个流程其实也差不多～
```c
// 在执行多任务前，原子锁对象初始值为0
volatile int g_atomic_lock = 0;

// 这里是多个线程可能共同的代码
void AtomicModify(void)
{
    // SWAP的第一个参数指向某一原子对象的地址。
    // SWAP操作一般是将第二个实参的值写入到原子对象中去，
    // 然后返回该原子对象在SWAP操作之前的值。
    while(SWAP(&g_atomic_lock, 1) == 1)
    {
        // 如果SWAP操作返回1，说明之前已经有线程对该对象上了锁，
        // 此时只能等待该原子对象重新变为0。
        CPU_PAUSE();    // 这里暗示CPU可以做对其他线程的调度
    }

    // SWAP操作成功之后，g_atomic_lock的值变为1了，
    // 此时对多个线程所共享的对象进行操作
    DoModificationToSharedObject();

    // 对共享对象操作完之后，释放原子锁
    SWAP(&g_atomic_lock, 0);
}
```

随着科技的进步，现代多核处理器纷纷引入了**无锁**（***Lock-Free***）原子操作，比如**比较与交换（Compare and Swap，简称CAS）**，**加载时锁定/有条件存储（Load-locked Store-conditional，简称LL-SC）**。x86处理器使用前者，Alpha、Power、MIPS、RISC-V、ARMv7、ARMv8等则使用后者，而从ARMv8.1开始也引入了CAS原子操作。LL-SC形式上虽然与CAS有些不同，但逻辑都是互通的。其主要思路就是在当前线程先加载共享对象的值，然后对它做任意修改操作，最后写的时候是原子的——先判定之前所加载的共享对象的值与当前共享对象的值是否完全相同，如果相同，则把新修改的值写回去；否则交换失败，程序可以做循环从而再次从该共享对象中加载值。这种无锁机制可以把之前的原子锁去掉，而直接对目标共享对象进行操作。其实践用法如以下代码所示：
```c
void AtomicSumOfSquares(volatile int *atom, int b)
{
    int orgValue, dstValue;
    do
    {
        // 先加载原子对象的当前值
        orgValue = *atom;

        // 再对原始值做平方和计算，作为最终写入的结果
        dstValue = orgValue * orgValue + b * b;
    }
    // 这里的CAS函数，第一个参数指向一个原子对象；
    // 第二个参数指向之前加载该原子对象的变量；
    // 第三个参数则是比较成功后，最后写入到该原子对象的值。
    // CAS先将原子对象的内容与之前加载它的变量的值进行比较，两者是否相同；
    // 如果相同，说明在此操作期间没有其他线程对该原子进行修改，
    // 则将第三个参数的值写到该原子对象中去，然后返回true；
    // 如果两者不同，说明这期间已经有某个线程对该原子对象进行了修改，返回false。
    while(!CAS(atom, &orgValue, dstValue);

    return dstValue;
}
```
我们看到，使用“无锁”的CAS原子操作也用了一个do-while循环，它跟之前的AtomicModify中的SWAP过程相比，到底无锁在什么地方呢？不都有循环吗？
各位同学请注意！CAS是直接对一个原子对象进行修改，而不是将它作为一个“自旋锁（Spin-Lock）”那样使用。SWAP原子操作需要等待其操作的原子对象处于一个“可用”的状态，如果它一直不处于可用状态，则会一直等下去，或者采用有限次尝试机制而做“异常处理”。而CAS对它所操作的原子对象是“瞬时”的，只要没有其他线程对该原子对象进行修改，那么CAS操作就一定成功，而CAS操作一成功，整个流程结束！因此前者SWAP机制是需要等原子对象的某一状态；而后者CAS机制则是去避免“数据冲突”情况的发生，所以它是无锁的。

我们理解了有锁的SWAP原子操作与无锁的CAS原子操作之后，下面将正式进入正题，我们来谈谈C11标准（ISO/IEC 9899:2011）中的原子操作标准库。

<br />

## C11标准中的原子操作标准库

首先，原子操作库在C11标准中属于**可选**库，对于一些低端的处理器，尤其像8位的单片机MCU那种完全不具备原子操作指令的系统环境，则可以不提供此库。因此C11标准引入了`__STDC_NO_ATOMICS__`这个预定义宏来指示当前系统环境下的C11编译器实现是否提供了原子操作标准库。如果定义了这个宏，则说明当前环境下的C11编译器实现没有支持原子操作的标准库。如果支持原子操作标准库，我们就能引入`<stdatomic.h>`这个头文件，这里面声明了所以能被当前C11编译器支持的原子类型以及原子操作函数接口。

<br />

#### C11中能支持的原子操作类型

上面已经提到过，由于当前处理器技术非常成熟，各种架构、各种配置的处理器百花齐放，有专门用于低功耗、高续航领域的，也有追求高性能领域的……因此这些处理器硬件本身对原子操作指令的支持也可能大相径庭！比如，有些处理器只支持SWAP或flag test and set等基于锁的原子操作；而有些能支持基于无锁版本的原子操作；还有一些能支持基于64位整数的无锁原子操作……为了能给程序员提供当前编译环境能支持何种原子操作，C11标准列出了以下这些与定义宏：
```c
// 指示当前编译器能支持 _Bool 类型的无锁原子操作
ATOMIC_BOOL_LOCK_FREE

// 指示当前编译器能支持 signed char、unsigned char、以及char类型的无锁原子操作
ATOMIC_CHAR_LOCK_FREE

// 指示当前编译器能支持 char16_t 类型的无锁原子操作
ATOMIC_CHAR16_T_LOCK_FREE

// 指示当前编译器能支持 char32_t 类型的无锁原子操作
ATOMIC_CHAR32_T_LOCK_FREE

// 指示当前编译器能支持 wchar_t 类型的无锁原子操作
ATOMIC_WCHAR_T_LOCK_FREE

// 指示当前编译器能支持 short、unsigned short 类型的无锁原子操作
ATOMIC_SHORT_LOCK_FREE

// 指示当前编译器能支持 int、unsigned int 类型的无锁原子操作
ATOMIC_INT_LOCK_FREE

// 指示当前编译器能支持 long、unsigned long 类型的无锁原子操作
ATOMIC_LONG_LOCK_FREE

// 指示当前编译器能支持 long long、unsigned long long 类型的无锁原子操作
ATOMIC_LLONG_LOCK_FREE

// 指示当前编译器能支持 ptrdiff_t、intptr_t、uintptr_t 类型的无锁原子操作
ATOMIC_POINTER_LOCK_FREE
```

有了这些预定义宏之后，我们就能判定在当前编译环境下可用那些原子操作了。当然，如果当前编译环境能支持原子操作的话，那么它至少应该能支持 **`atomic_flag`** 类型。该类型是一个原子标志对象，用于flag test and set原子操作。如果当前的CPU不支持flag test and set，但支持SWAP，那么也可以用SWAP来实现该操作。此外，其他lock-free的原子操作也都能实现flag test and set，包括原子加法、原子逻辑操作、CAS等。因此**原子标志操作**属于整个原子操作中最最基本的操作方式。我们稍后将会对这些原子操作做具体介绍。

对于其他整数类型，如果当前编译环境能支持的话（可根据上面列出的预定义宏来判断），都有其相对应的原子类型。尽管C11标准引入了 **`_Atomic`** 宏，我们可用它将一个基本整数类型转换为对应的原子类型，比如：
```c
char  =>  _Atomic(char)
short => _Atomic(short)
int   => Atomic(int)
```
这里的 `( )` 可省略，但类型与 `_Atomic`之间必须至少有一个空白符；不过笔者仍然建议各位在使用原子类型的时候，先包含 **`<stdatomic.h>`** ，然后使用该头文件所列出的原子类型。当前C11标准所提供支持的整数原子类型如下表所示：
```c
typedef _Atomic(_Bool)              atomic_bool;
typedef _Atomic(char)               atomic_char;
typedef _Atomic(signed char)        atomic_schar;
typedef _Atomic(unsigned char)      atomic_uchar;
typedef _Atomic(short)              atomic_short;
typedef _Atomic(unsigned short)     atomic_ushort;
typedef _Atomic(int)                atomic_int;
typedef _Atomic(unsigned int)       atomic_uint;
typedef _Atomic(long)               atomic_long;
typedef _Atomic(unsigned long)      atomic_ulong;
typedef _Atomic(long long)          atomic_llong;
typedef _Atomic(unsigned long long) atomic_ullong;

// 对于没有定义过char16_t的编译环境，
// 也可能会用 _Atomic(uint_least16_t) 类型来定义其相应的原子类型
typedef _Atomic(char16_t)            atomic_char16_t;

// 对于没有定义过char32_t的编译环境，
// 也可能会用 _Atomic(uint_least32_t) 类型来定义其相应的原子类型
typedef _Atomic(char32_t)            atomic_char32_t;

typedef _Atomic(wchar_t)            atomic_wchar_t;
typedef _Atomic(int_least8_t)       atomic_int_least8_t;
typedef _Atomic(uint_least8_t)      atomic_uint_least8_t;
typedef _Atomic(int_least16_t)      atomic_int_least16_t;
typedef _Atomic(uint_least16_t)     atomic_uint_least16_t;
typedef _Atomic(int_least32_t)      atomic_int_least32_t;
typedef _Atomic(uint_least32_t)     atomic_uint_least32_t;
typedef _Atomic(int_least64_t)      atomic_int_least64_t;
typedef _Atomic(uint_least64_t)     atomic_uint_least64_t;
typedef _Atomic(int_fast8_t)        atomic_int_fast8_t;
typedef _Atomic(uint_fast8_t)       atomic_uint_fast8_t;
typedef _Atomic(int_fast16_t)       atomic_int_fast16_t;
typedef _Atomic(uint_fast16_t)      atomic_uint_fast16_t;
typedef _Atomic(int_fast32_t)       atomic_int_fast32_t;
typedef _Atomic(uint_fast32_t)      atomic_uint_fast32_t;
typedef _Atomic(int_fast64_t)       atomic_int_fast64_t;
typedef _Atomic(uint_fast64_t)      atomic_uint_fast64_t;
typedef _Atomic(intptr_t)           atomic_intptr_t;
typedef _Atomic(uintptr_t)          atomic_uintptr_t;
typedef _Atomic(size_t)             atomic_size_t;
typedef _Atomic(ptrdiff_t)          atomic_ptrdiff_t;
typedef _Atomic(intmax_t)           atomic_intmax_t;
typedef _Atomic(uintmax_t)          atomic_uintmax_t;
```

由于绝大部分处理器都没有提供针对浮点数的原子操作指令，因此C11标准也没有提供任何针对浮点数的原子类型！这里请各位务必注意。

<br />

#### C11中所提供的原子操作函数接口

C11所提供的原子操作函数接口一般都有两个版本。第一个版本是不带有存储器次序参数的，第二个是带有存储器次序参数的，并且函数名也以 `_explicit` 结尾。我们再下一章将会详细描述存储器次序，这是一个比较复杂也比较高级的话题，对于初学者而言，我们先把不带存储器次序的搞明白就哦了～

<br />

##### 一、原子标志相关操作：

对于原子标志的相关操作，C11标准提供了初始化、标志测试与设置、标志清除这三个接口。原子标志 **`atomic_flag`** 对象本身只有两种状态（即只有两种取值），**设置状态**（编译器实现一般用 **1** 或 **`true`** 来表示）以及**清零状态**（编译器实现一般用 **0** 或 **`false`** 来表示）。

C11标准为原子标志类型提供了一个用于初始化的宏—— **`ATOMIC_FLAG_INIT`** ，我们应该用这个宏对一个原子标志对象进行初始化，如以下代码所示。
```c
volatile atomic_flag g_flag = ATOMIC_FLAG_INIT;

int main(void)
{
    // 这里展示了如何在函数内对已声明的g_flag进行初始化。
    // 由于atomic_flag通常被定义为一种结构体形式，
    // 而ATOMIC_FLAG_INIT则通常被定义为针对结构体atomic_flag的初始化器，
    // 即：{ ... } 的形式。
    // 因此我们这里使用C99标准所引入的匿名结构体对象的表示语法
    // 对g_flag进行初始化
    g_flag = (atomic_flag)ATOMIC_FLAG_INIT;
}
```
初始化之后，原子标志对象即处于“清零状态”。

原子标志的测试与设置操作函数有explicit版本，其原型为：
```c
_Bool atomic_flag_test_and_set(volatile atomic_flag *object);

_Bool atomic_flag_test_and_set_explicit(volatile atomic_flag *object, memory_order order);
```
该操作函数的语义为：对原子标志对象 **object** 进行设置，并返回该原子标志对象做设置操作之前的值。也就是说，如果原子标志对象  **`*object`** 之前为“清零状态”，那么经过此操作之后，该原子标志对象的状态变为了“设置状态”，并且返回 **`false`** 。如果原子标志对象  **`*object`** 之前为“设置状态”，那么经过此操作之后，它仍然为“设置状态”，并且返回 **`true`** 。所以，当我们将原子标志对象作为一个“锁”来用的话，就看这个函数接口返回的是啥，如果返回 `false`，则说明锁成功，可以对多线程共享对象做相关的修改操作；如果返回的是 `true`，则说明该原子标志已经被其他线程占用了，需要等待释放。

最后再谈谈原子标志的清除操作函数接口。它也有explicit版本，其原型为：
```c
void atomic_flag_clear(volatile atomic_flag *object);
         
void atomic_flag_clear_explicit(volatile atomic_flag *object, memory_order order);
```
该操作函数的语义为：对指定的原子标志对象 **object** 进行清零操作。如果我们将原子标志对象用作“锁”的话，那么执行此操作就相当于释放锁。

作为C11标准里最最基本的原子类型，原子标志的适用性还是非常广的。尤其要对一些较大的共享资源进行操作的话，无锁原子操作也难以胜任，此时就可以用原子标志或甚至是系统自带的同步原语进行同步操作了。下面笔者将用原子标志操作来举一个例子🌰，描述如何对一个共享浮点数对象做递增的原子操作。
```c
#include <stdio.h>
#include <stdbool.h>
#include <stdint.h>
#include <stddef.h>
#include <stdalign.h>
#include <stdatomic.h>

#include <pthread.h>

// 为了避免CPU直接死等，
// 这里定义了常用处理器架构的释放当前线程的暗示指令操作
#if defined(__x86__) || defined(__x86_64__) || defined(__i386__)

#define CPU_PAUSE()     asm("pause")

#elif defined(__arm__) || defined(__aarch64__)

#define CPU_PAUSE()     asm("yield")

#else

#define CPU_PAUSE()

#endif


/// 为了通用性，这里用一个结构体封装了原子标志与普通的浮点数对象
struct MyAtomicFloat
{
    volatile atomic_flag atomFlag;
    
    // 出于性能上考虑，我们应该尽量让原子对象与普通对象之间留有些空间
    int padding;
    
    // 如果当前编译器能支持C11的alignas的话，
    // 那么我们也能使用alignas来做字节对齐
    alignas(16) float value;
};

/// 定义一个将被多线程共享的原子浮点数对象
static volatile struct MyAtomicFloat sAtomicFLoatObject;

/// 对多线程共享的原子对象进行求递增操作
/// @param nLoops 指定对共享原子对象操作几次
static void AtomicValueInc(int nLoops)
{
    // 这里对共享原子对象操作nLoops次
    for(int loop = 0; loop < nLoops; loop++)
    {
        // 先进行上锁
        while(atomic_flag_test_and_set(&sAtomicFLoatObject.atomFlag))
            CPU_PAUSE();
        
        // 对共享数据做递增操作
        sAtomicFLoatObject.value += 1.0f;
        
        // 最后释放锁
        atomic_flag_clear(&sAtomicFLoatObject.atomFlag);
    }
}

/// 线程处理函数
static void* ThreadProc(void *args)
{
    // 在用户线程中执行10000次
    AtomicValueInc(10000);
    
    return NULL;
}

int main(int argc, const char * argv[])
{
    printf("The size is: %zu, `value` offset is: %zu\n",
           sizeof(sAtomicFLoatObject), offsetof(struct MyAtomicFloat, value));
    
    // 对原子浮点数对象先进行初始化
    sAtomicFLoatObject.atomFlag = (atomic_flag)ATOMIC_FLAG_INIT;
    sAtomicFLoatObject.value = 0.0f;
    
    pthread_t threadID;
    // 创建线程并调度执行
    if(pthread_create(&threadID, NULL, ThreadProc, NULL) != 0)
    {
        puts("Failed to create a thread!");
        return 0;
    }
    
    // 在主线程中也执行10000次
    AtomicValueInc(10000);
    
    // 等待线程执行完毕
    pthread_join(threadID, NULL);
    
    // 输出最终结果
    printf("The final result is: %f\n", sAtomicFLoatObject.value);
}
```
上述代码以及后面的代码出于可跨平台考虑吧，用到了pthread库，因此如果各位在Linux环境下编译运行的话需要添加 `-pthread` 编译选项，macOS、iOS等Apple系统环境则不需要，pthread是被默认连接的。此外，上述代码以及后续代码都要用到C11标准，所以各位所使用的编译器如果稍旧的话（比如GCC 4.8，Clang 3.6），那么必须显式地加上 `-std=gnu11` 编译选项。
Windows系统下MSVC没有提供原子操作的库，笔者这里为Windows平台的开发者封装了一个，可供使用：https://github.com/zenny-chen/simple-stdatomic-for-VS-Clang

上述代码配合注释讲解后，各位应该能很容易地看明白。各位如果感兴趣的话，可以尝试一下将   `while(atomic_flag_test_and_set(&sAtomicFLoatObject.atomFlag))` 这条语句注释掉，然后观察一下输出结果是否正确。

以上描述的是关于 **`atomic_flag`** 类型的相关原子操作描述。下面要介绍的原子操作函数都是基于整数原子类型的（包括布尔与字符类型）。
此外，C11标准为了简化针对整数原子类型的原子操作接口，将所有针对整数原子类型的操作函数以“**泛型**”的形式给出，因此我们在某些编译器实现中查看其自带的 **`<stdatomic.h>`** 头文件时，能发现里面的许多原子操作函数都用宏定义成了编译器自带的**内建函数**（***intrinsic functions***）形式了。这样能简化函数接口，毕竟要为每种整数原子类型写一种原子操作，太过冗长了。我们可以参考上面列出来的那么多整数原子类型。而且，要让程序员再自己去判定用哪种类型的原子操作，也将是一件很麻烦的事情。

<br />

##### 二、整数原子类型对象的初始化：

C11标准为了简化对整数原子对象的初始化，引入了一种宏函数——**`ATOMIC_VAR_INIT`** ，当我们声明一个整数原子对象，然后立即对它初始化时，可以使用这个宏函数。该宏函数接收一个参数，用于指定该原子对象的初始值。当然，我们必须注意的是，初始值的类型要与原子对象的整数类型相兼容。我们可以看以下例子。

```c
#include <stdatomic.h>
#include <stdbool.h>

// 将sIntAtom原子对象初始化为10
static volatile atomic_int sIntAtom = ATOMIC_VAR_INIT(10);

// 将sBoolAtom原子对象初始化为true
static volatile atomic_bool sBoolAtom = ATOMIC_VAR_INIT(true);

// 将sCharAtom原子对象初始化为'c'
static volatile atomic_char sCharAtom = ATOMIC_VAR_INIT('c');

int main(int argc, const char * argv[])
{
    // 将intAtom原子对象初始化为10
    volatile atomic_int intAtom = ATOMIC_VAR_INIT(10);
    
    // 将boolAtom原子对象初始化为true
    volatile atomic_bool boolAtom = ATOMIC_VAR_INIT(true);
    
    // 将charAtom原子对象初始化为'c'
    volatile atomic_char charAtom = ATOMIC_VAR_INIT('c');
}
```

还有一种初始化方法是调用 **`void atomic_init(volatile A *atom, C value);`** 函数。它有两个参数，第一个参数指向一个整数原子类型的对象；第二个参数是为该原子对象指定的初始值。当我们先声明了某个整数原子类型的对象，之后再为它初始化时，就可以调用这个函数，我们可以看以下代码例子。

```c
#include <stdatomic.h>
#include <stdbool.h>

int main(int argc, const char * argv[])
{
    volatile atomic_int intAtom;
    volatile atomic_bool boolAtom;
    volatile atomic_char charAtom;
    
    // 将intAtom原子对象初始化为10
    atomic_init(&intAtom, 10);
    
    // 将boolAtom原子对象初始化为false
    atomic_init(&boolAtom, false);
    
    // 将charAtom原子对象初始化为'c'
    atomic_init(&charAtom, 'c');
}
```

这里大家还需要注意的是，对原子对象的初始化操作并非是原子的！因此我们往往在做多线程操作之前先对原子对象做必要的初始化。此外，对原子对象的初始化应该使用上述所提到的初始化方式，而不是用下面将描述的原子存储与加载操作。下面即将描述的所有原子操作都应该基于已初始化完的原子对象！

<br />

##### 三、整数原子类型对象的存储与加载：

整数原子类型的存储与加载均有两种模式，一种是默认存储器次序模式，还有一种则是显式指定存储器次序的模式。我们先列出整数原子类型加载操作。

```c
C atomic_load(volatile A *object); 

C atomic_load_explicit(volatile A *object, memory_order order);
```

对于默认存储器次序的整数类型原子加载操作而言，它有一个参数，此参数指向一个整数原子类型的对象，然后返回该原子对象的当前值。下面我们来看些例子：

```c
#include <stdio.h>
#include <stdatomic.h>
#include <stdbool.h>

int main(int argc, const char * argv[])
{
    volatile atomic_int intAtom = ATOMIC_VAR_INIT(10);
    volatile atomic_bool boolAtom = ATOMIC_VAR_INIT(true);
    volatile atomic_char charAtom = ATOMIC_VAR_INIT('a');
    
    // 加载intAtom原子对象的值
    int i = atomic_load(&intAtom);
    
    // 加载boolAtom原子对象的值
    bool b = atomic_load(&boolAtom);
    
    // 加载charAtom原子对象的值
    char c = atomic_load(&charAtom);
    
    printf("i = %d, b = %d, c = %c\n", i, b, c);
}
```

整数原子类型的存储操作也有两种版本，一个是默认存储器次序的，另一个是显式指定存储器次序的。

```c
void atomic_store(volatile A *object, C desired); 

void atomic_store_explicit(volatile A *object, C desired, memory_order order);
```

对于默认存储器次序的版本，该操作函数具有两个参数，第一个参数指向一个整数原子类型的对象，第二个参数用于指定所要存储的值。下面来看些例子：

```c
#include <stdio.h>
#include <stdatomic.h>
#include <stdbool.h>

int main(int argc, const char * argv[])
{
    volatile atomic_int intAtom = ATOMIC_VAR_INIT(0);
    volatile atomic_bool boolAtom = ATOMIC_VAR_INIT(false);
    volatile atomic_char charAtom = ATOMIC_VAR_INIT('\0');
    
    // 用-100来存储intAtom原子对象的值
    atomic_store(&intAtom, -100);
    
    // 用true来存储boolAtom原子对象的值
    atomic_store(&boolAtom, true);
    
    // 用'c'来存储charAtom的值
    atomic_store(&charAtom, 'c');
    
    // 加载intAtom原子对象的值
    int i = atomic_load(&intAtom);
    
    // 加载boolAtom原子对象的值
    bool b = atomic_load(&boolAtom);
    
    // 加载charAtom原子对象的值
    char c = atomic_load(&charAtom);
    
    printf("i = %d, b = %d, c = %c\n", i, b, c);
}
```

<br />

##### 四、整数原子类型对象的交换操作：

C11标准中的整数原子类型的交换操作其实就对应了本文一开始所提到的SWAP原子操作。C11标准中给出了两个交换操作版本，一个是默认存储器次序的，另一个是显式指定存储器次序的。

```c
C atomic_exchange(volatile A *object, C desired);

C atomic_exchange_explicit(volatile A *object, C desired, memory_order order);
```

对于默认存储器次序的版本，交换操作函数提供了两个参数，第一个参数指向某个整数原子类型的对象；第二个参数指定了想要存储到该原子对象中的值。该函数返回指定原子对象在执行此操作之前的值。下面我们给出一些例子。

```c
#include <stdio.h>
#include <stdatomic.h>

int main(int argc, const char * argv[])
{
    volatile atomic_int atom = ATOMIC_VAR_INIT(0);
    
    // 使用atomic_exchange操作将1写入到atom原子对象，
    // 然后返回atom原先的值——0
    int value = atomic_exchange(&atom, 1);
    printf("value = %d, atom = %d\n", value, atomic_load(&atom));
    
    // 我们可以再来一遍
    value = atomic_exchange(&atom, 2);
    
    // 这里输出：value = 1, atom = 2
    printf("value = %d, atom = %d\n", value, atomic_load(&atom));
}
```

我们可以自己尝试一下，用 `atomic_exchange` 原子操作来实现 `atomic_flag_test_and_set` 操作的语义，若有不太明白的地方欢迎留言。

<br />

##### 五、整数原子类型对象的比较与交换操作：

C11标准中的整数原子类型对象的比较与交换操作其实就对应了本文一开始所提到的CAS原子操作。C11标准中给出了四个原子比较与交换操作的版本，两个是默认存储器次序的，另外两个是显式指定存储器次序的。

```c
_Bool atomic_compare_exchange_strong(volatile A *object, C *expected, C desired);

_Bool atomic_compare_exchange_strong_explicit(volatile A *object, C *expected, C desired,
memory_order success, memory_order failure); 

_Bool atomic_compare_exchange_weak(volatile A *object, C *expected, C desired);

_Bool atomic_compare_exchange_weak_explicit(volatile A *object, C *expected, C desired, memory_order success, memory_order failure);
```

这里有strong版本与weak版本。它们的语义都差不多，均实现了之前提到的CAS语义逻辑。对于默认存储器次序的操作而言，strong与weak版本都提供了三个参数，第一个参数指向某个整数原子类型对象；第二个参数指向要进行比较的对象，并且如果比较失败，那么该操作会将原子对象的当前值拷贝到该参数所指向的对象中；第三个参数指定存储到原子对象中的值。
如果比较成功，那么desire值会被存放到原子对象中，并且返回 `true`；如果比较失败，那么当前原子对象的值会被拷贝到expected所指向的对象中，并且返回 `false`。

strong版本与CAS的语义完全一致，而weak版本则有些区别。weak版本可能在当前比较成功的情况下，也会被判定为失败。C11标准之所以加入weak语义是为了能使更多的原子操作机制来实现CAS功能，比如通过LL-SC机制来实现CAS原子操作的话，weak版本会更好一些。
那么我们应该如何去选择呢？C11标准建议，如果我们采用像之前提到的，通过循环去测试CAS比较是否成功的话，那么使用weak版本在某些平台上能获得更好的性能；如果我们只是单独对某个原子对象做一次CAS操作，而当前不管这次操作是否成功的话，那么用strong版本更好一些。下面我们来举些例子。

```c
#include <stdio.h>
#include <stdatomic.h>
#include <stdbool.h>

int main(int argc, const char * argv[])
{
    volatile atomic_int atom = ATOMIC_VAR_INIT(0);
    
    // 我们先从atom原子对象加载其值
    int expected = atomic_load(&atom);
    
    // 我们就对atom原子对象操作一次，因此这里用strong版本。
    // 如果比较成功，就将1存储到atom原子对象中
    bool equal = atomic_compare_exchange_strong(&atom, &expected, 1);
    
    // 这里输出：Is equal? 1, atom value is: 1
    printf("Is equal? %d, atom value is: %d\n", equal, atomic_load(&atom));
    
    // 我们再次加载atom的值
    expected = atomic_load(&atom);
    
    // 我们对expected进行了修改
    expected += 10;
    
    // 由于这次比较，expected所存储的值与atom的值不相等，
    // 因此将atom的值重新存放到expected中，且返回false。
    equal = atomic_compare_exchange_strong(&atom, &expected, -1);
    
    // Is equal? 0, expected value is: 1
    printf("Is equal? %d, expected value is: %d\n", equal, expected);
}
```

通过这个例子，相信各位对atomic_compare_exchange操作已经有了感性认识了吧～
下面笔者将为大家来演示一下，如何通过比较与交换原子操作来实现针对一个浮点数的多线程递增计算。

```c
#include <stdio.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <pthread.h>

/// 定义一个将被多线程共享的整数原子对象，
/// 它后面将会被充当一个单精度浮点数
static volatile atomic_int sAtomicFLoatObject;

/// 对多线程共享的原子对象进行求递增操作
/// @param nLoops 指定对共享原子对象操作几次
static void AtomicValueInc(int nLoops)
{
    // 这里对共享原子对象操作nLoops次
    for(int loop = 0; loop < nLoops; loop++)
    {
        // 先读取sAtomicFLoatObject的当前值
        int orgValue = atomic_load(&sAtomicFLoatObject);
        float dstValue;
        
        do
        {
            // 我们将orgValue所表示的单精度浮点数萃取出来，
            // 保证不损失任何精度，然后在此基础上递增0.1
            dstValue = *(float*)&orgValue + 0.1f;
        }
        // 由于我们这里需要最终获得正确的值，因此这里用了weak版本，
        // 在循环条件下对于某些硬件平台能获得更好的性能
        while(!atomic_compare_exchange_weak(&sAtomicFLoatObject, &orgValue, *(int*)&dstValue));
    }
}

/// 线程处理函数
static void* ThreadProc(void *args)
{
    // 在用户线程中执行10000次
    AtomicValueInc(10000);
    
    return NULL;
}

int main(int argc, const char * argv[])
{
    const float zero = 0.0f;
    // 我们这里为了展示所使用的一些“黑科技”，
    // 而显式地用单精度浮点数所表示的IEEE整数来为
    // sAtomicFLoatObject进行初始化
    atomic_init(&sAtomicFLoatObject, *(int*)&zero);
    
    pthread_t threadID;
    // 创建线程并调度执行
    if(pthread_create(&threadID, NULL, ThreadProc, NULL) != 0)
    {
        puts("Failed to create a thread!");
        return 0;
    }
    
    // 在主线程中执行10000次
    AtomicValueInc(10000);
    
    // 等待线程执行完毕
    pthread_join(threadID, NULL);
    
    // 输出最终结果
    const int result = atomic_load(&sAtomicFLoatObject);
    printf("The final result is: %f\n", *(float*)&result);
    
    // 由于计算精度关系，最终结果可能不会正好为2000.0f，
    // 因此，我们可以在写一个简单的算法进行验证结果的正确性！
    float sum = 0.0f;
    for(int i = 0; i < 20000; i++)
        sum += 0.1f;
    
    // 由于算法相同，在没经过任何优化的情况下，
    // 两者在IEEE二进制表达上应该是完全一致的！
    if(sum == *(float*)&result)
        puts("Equal!");
}
```

同样，这里也用到了pthread库，因此如果各位在Linux环境下编译运行的话需要添加 `-pthread` 编译选项，macOS、iOS等Apple系统环境则不需要，pthread是被默认连接的。此外，上述代码以及后续代码都要用到C11标准，所以各位所使用的编译器如果稍旧的话（比如GCC 4.8，Clang 3.6），那么必须显式地加上 `-std=gnu11` 编译选项。
而在Windows系统下MSVC没有提供原子操作的库，笔者这里为Windows平台的开发者封装了一个，可供使用：https://github.com/zenny-chen/simple-stdatomic-for-VS-Clang

<br />

##### 六、整数原子类型对象的基本算术逻辑操作：

C11标准中提供了针对整数原子类型对象的基本算术逻辑操作，这又被称为**原子获取与修改**（***atomic fetch and modify***）操作。这里各位需要注意的是，以下这些操作不适用于 **`atomic_bool`** 原子类型，而只能应用于除此之外的其他整数原子类型。
原子获取与修改操作有如下这些品种：加法（add），减法（sub），按位与（and），按位或（or），按位异或（xor）。每种原子获取与修改操作都有两版本，一个版本为 默认存储器次序，另一个版本为显式指定存储器次序。其函数原型如下所示：

```c
C atomic_fetch_<key>(volatile A *object, M operand); 

C atomic_fetch_<key>_explicit(volatile A *object, M operand, memory_order order);
```

上述函数原型的标识符中，<key>对应于具体操作名称，对于原子加法，其<key>就是 add；对于原子按位与操作，其<key>就是 and。对于默认存储器次序的版本，这些函数具有两个参数，第一个参数指向某个整数原子对象；第二个参数为修改操作的操作数，比如对于加法操作就是“加数”，对于减法操作则是“减数”；而原子对象则分别作为“被加数”和“被减数”。

之所以称这些原子操作为“原子获取与修改”操作，是因为这些原子操作的步骤都是先获取指定原子对象的当前值，然后在此基础上做算术逻辑运算，最后将计算结果写入到该原子对象中并返回该原子对象做此操作之前的值。这一过程很明显，就是先获取后修改。下面我们来看一些代码例子。

```c
#include <stdio.h>
#include <stdatomic.h>

int main(int argc, const char * argv[])
{
    volatile atomic_int atom = ATOMIC_VAR_INIT(10);
    
    // 这里对原子对象atom做原子加法操作，
    // 将它与5相加，再将结果存入该原子对象
    int value = atomic_fetch_add(&atom, 5);
    
    // 输出：value = 10, atom = 15
    printf("value = %d, atom = %d\n", value, atomic_load(&atom));
    
    // 这里对原子对象atom做原子减法操作，
    // 将它与8相减，再将结果存入该原子对象
    value = atomic_fetch_sub(&atom, 8);
    
    // 输出：value = 15, atom = 7
    printf("value = %d, atom = %d\n", value, atomic_load(&atom));
    
    // 这里对原子对象atom做原子按位异或操作，
    // 将它与7做按位异或j运算，再将结果存入该原子对象
    value = atomic_fetch_xor(&atom, 7);
    
    // 输出：value = 7, atom = 0
    printf("value = %d, atom = %d\n", value, atomic_load(&atom));
}
```

原子获取与修改操作能应用在很多场合，比如我们要利用多线程对某些资源进行计算，然后进行汇总时可能就会对其中一个共享资源做原子获取与修改操作，这也属于我们在操作系统中常用的“fork-join”的一种机制。
下面我们将举一个比较实际的例子。假定我们有100个数组，每个数组有10000个元素，我们现在要对所有这1000个数组中的所有元素进行求和操作，我们怎么算比较快呢？传统的思路是先查看我们当前的计算环境有多少CPU，每个CPU含有多少核心，然后进行平均划分。但这里有个问题是，计算机系统往往不会只有我们当前一个前台程序在运行，可能会有其他一些后台任务，甚至有一些高优先级的任务需要处理等等，比如我们边运行我们这个程序，可能又在听音乐，开着浏览器在网上冲浪等等……所有这些任务都需要占用CPU资源。因此，一种可能更好的方法是仍然针对核心个数开线程（比如你的CPU有四个核心，就开四个线程），但是每个线程不是平均分配给它所要计算的元素个数，而是给一批，这样每个线程完成一批数据处理之后再去取下一批进行计算。这样即便某些线程受到其他任务调度而被阻塞，但也不至于使当前的任务被“卡住”，其他线程可以“接手”它后面所要计算的资源。
因此，对于下面这个demo，为了简单起见，笔者仍然用两个线程，每个线程一次迭代就处理其中一个数组的所有元素之和，然后接着取下一个可操作的数组。我们利用原子加法操作来操纵当前所要操作数组的索引。这种解决多线程并行任务的方法想必能给各位一定的启发。

```c
#include <stdio.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <pthread.h>

/// 我们定义了带有100个元素的数组，
/// 每个数组元素是一个含有10000个int元素的数组
static int sArrays[100][10000];

/// 此整数原子对象用于指示当前线程所要操作的数组索引
static volatile atomic_int sAtomicArrayIndex = ATOMIC_VAR_INIT(0);

/// 此整数原子对象用于存放最终的求和结果
static volatile atomic_int sAtomicArraySum = ATOMIC_VAR_INIT(0);

/// 对共享数组进行求和操作
/// 如果当前数组还没计算完，返回true；否则返回false
static bool AtomicComputeArraySum(void)
{
    // 获取当前所要计算的数组个数
    const int nLoops = (int)(sizeof(sArrays[0]) / sizeof(sArrays[0][0]));
    
    // 获取数组sArrays总共有多少元素
    const int arrayLen = (int)(sizeof(sArrays) / sizeof(sArrays[0]));
    
    // 利用原子加法来获取当前所要操作数组的索引
    const int currArrayIndex = atomic_fetch_add(&sAtomicArrayIndex, 1);
    
    // 若当前索引已经达到了数组长度，则直接返回false，说明数组已经全部计算完成
    if(currArrayIndex >= arrayLen)
        return false;
    
    // 对当前指派到的数组元素进行求和
    int sum = 0;
    for(int index = 0; index < nLoops; index++)
        sum += sArrays[currArrayIndex][index];
    
    // 将结果进行累加
    atomic_fetch_add(&sAtomicArraySum, sum);
    
    return true;
}

/// 线程处理函数
static void* ThreadProc(void *args)
{
    // 在用户线程中计算
    while(AtomicComputeArraySum());
    
    return NULL;
}

int main(int argc, const char * argv[])
{
    // 获取数组每个元素的数组长度
    const int nElems = (int)(sizeof(sArrays[0]) / sizeof(sArrays[0][0]));
    
    // 获取数组sArrays总共有多少元素
    const int arrayLen = (int)(sizeof(sArrays) / sizeof(sArrays[0]));
    
    // 我们先对共享的二维数组进行初始化，
    // 为了方便验证结果，将它所有数组的所有元素初始化为1
    for(int i = 0; i < arrayLen; i++)
    {
        for(int j = 0; j < nElems; j++)
            sArrays[i][j] = 1;
    }

    pthread_t threadID;
    // 创建线程并调度执行
    if(pthread_create(&threadID, NULL, ThreadProc, NULL) != 0)
    {
        puts("Failed to create a thread!");
        return 0;
    }
    
    // 在主线程中计算
    while(AtomicComputeArraySum());
    
    // 等待线程执行完毕
    pthread_join(threadID, NULL);
    
    // 输出最终结果
    const int result = atomic_load(&sAtomicArraySum);
    printf("The final result is: %d\n", result);
}
```

<br/>

## C11标准中的所引入的存储器次序机制

C11标准引入了一组存储器次序枚举类型：
```c
typedef enum memory_order {
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
} memory_order;
```
用于指定存储器访问按哪种次序进行，这里不仅针对原子对象，而且也能包含常规的、非原子的存储器访问。在一个多核多处理器环境下，当多个线程同时对几个变量**以宽松的存储器次序**（即使用*memory_order_relaxed*）进行读写时，其中一个线程所观察到的这些变量值的变化与另一个写这些变量的线程所见的次序可能是不同的。实际上，甚至在多个读线程之间，这些变量值变化的所见次序也可能是不同的。此外，在单核单线程环境下，C11也允许使用这组存储器次序，因为出于优化目的，编译器可以重新编排相互独立的读写操作顺序。

我们如何来理解基于多核多线程的存储器次序呢？对于不同线程所观察到的若干变量的读写次序而有所不同，是如何引发的呢？这得从多核处理器的存储器结构层级说起。

![memory-hierarchy](https://github.com/zenny-chen/C11-atomic-operations-in-detail/blob/master/memory-hierarchy.png)

现代多核处理器的存储器层级一般分为多个层，最靠近CPU的为L1 Cache，容量最小但速度最快，并且它是仅针对一个处理器核心独享的。
然后再向外一层是L2 Cache，在某些移动设备上，它是被所有核心共享的，而在其他一些设备以及主流的桌面处理器上它也是被单个核心独享的，并且其容量比L1 Cache更大一些，速度也稍慢一些。
然后再向外一层是L3 Cache，当然有些中低端设备可能没有L3 Cache。L3 Cache是被所有核心共享的。其容量很大，目前来说基本都至少1MB了，不过速度比L2 Cache要慢。此外，如果它作为最后一层Cache的话（简称为**LLC**），那么它也可能被核心GPU所共享。
最后就是我们上面提到的LLC，如果有L4 Cache的话就是L4 Cache，否则就是L3 Cache。上面提到了，LLC一般是给整个系统所共享。当前有些系统使用eDRAM作为LLC，其容量可以做得很大，当然速度也会更慢一些，一般来说SRAM的速度会更快一些，但它肯定比需要通过总线才能访问的外部DDR要来得快了～
然后最外部的就是外部存储器了。

所以我们看到，我们用C语言写一句看似很简单的加载或存储赋值语句，而对于CPU来说可能要做非常多的工作，这其中就是既要保证一定的访存效率，还得保证Cache缓存的数据一致性等。因此，对于现代不少种类的处理器架构而言，都在其系统中引入了更弱的存储器次序，使得整个访存效率能得以提升。下面我们举一个相对比较容易理解的一个例子来说明这种所谓的观察到的存储器次序不同的情景。

![Cache](https://github.com/zenny-chen/C11-atomic-operations-in-detail/blob/master/cache.png)

上图中展示了一个具有三层Cache层级，并且有两个处理器核心的系统。向下箭头表示存储（写）操作；向上箭头表示加载（读）操作。并且左边的操作时序早于右边的操作。这里有两个被两个核心线程所共享的变量x和y，同时假定右边的核心线程先做 y = 0 的存储操作，使得在它的Cache中都安排好了存放变量y的Cache条目（entry），并且假定此时没有关于任何针对变量x的Cache条目。

首先，右边核心线程先对y进行 y = 0 的存储操作，等该操作全部完毕后再执行左边核心线程的操作。
左边核心线程先做 x = 1 的存储操作，紧接着再做 y = 2 的存储操作。完成之后，右边核心线程立即对x和y进行读取。
我们从图中可观察到，x和y的存储操作依次经过了左边核心的L1 Cache，L2 Cache，再是两个核心共享的L3 Cache，最后到外部存储器。此时，L3 Cache中已经有了x和y这两个变量所对应的Cache 条目。因此在右边核心读的时候，x和y的写次序基本是一致的。
然后到了右边核心的L2 Cache，由于之前没有x相关的Cache条目，因此此时L2 Cache控制器会进行判断是直接将它分配给一个空白的Cache条目还是将已有的Cache条目进行逐出，然后将变量x所处的Cache行添加到Cache条目中。这里就会有一些延迟与存储器重新安排的情况。此时，由于变量y已经处于Cache条目中，因此它有可能被直接写回（write back），只要之前针对x的Cache行的安排过程不将y所处的Cache行逐出。
这么一来，右边核心线程所观察到的写次序就会变为先写y再写x了。

当然，上述情况仅仅是存储器次序的某一种，像x86、ARM处理器中均引入了非临时（Non-Temporal）加载与存储操作，这些操作不会通过Cache，而是直接针对外部存储器控制器进行访存。而它们的访存次序就是典型的弱存储器次序，因为即便在总线上都会有各种不同情况发生。这就好比，我们在做网络通信的时候会碰到，先发送的请求反而后送达的情况。最简单的例子，比如我们用微信或QQ在发消息，如果此时网络信号不好，你会看到之前发送的几条消息都在“转圈圈”，等信号好的时候，往往是之前最后发送的那条消息率先送达给对方～笔者已经遇到过不少次这种情况了😂

我们在上一章已经看到了，C11标准中所引入的大部分原子操作都有两个版本，其中一个是具有默认存储器次序的原子操作；还有一个则是显式指定存储器次序的原子操作。对于默认存储器次序的原子操作而言，其存储器次序为最严格的 **`memory_order_seq_cst`** ，它表示当前的原子操作必须满足**存储器顺序一致的**（**sequentially consistent**）。一般来说，默认的存储器次序，即 **`memory_order_seq_cst`** ，对于某些场景下可能会过于严苛，从而会影响整体性能。而对于显式指定的存储器次序，无论是处理器系统还是编译器都必须严格遵循所指定存储器次序的约束条件，所实现的存储器次序强度**不能弱于**所指定的存储器次序类型。比如：
```c
    volatile atomic_int atom = ATOMIC_VAR_INIT(0);
    
    // 这里使用acquire次序加载atom原子对象
    int value = atomic_load_explicit(&atom, memory_order_acquire);
```
上述代码用了 `memory_order_acquire` 存储器次序去加载atom原子对象。那么无论是处理器系统还是编译器实现，对atom原子对象加载所用的存储器次序不能是比 `memory_order_acquire` 更弱的次序（比如 `memory_order_relaxed`）；当然，比它更强没有问题，比如使用 `memory_order_seq_cst` 完全顺序一致的存储器次序。

<br/>

#### C11标准中的栅栏操作

在正式描述上述列出的六种存储器次序之前，我们这里先插播一条关于栅栏操作的消息。有时候，我们可能对多线程所共享变量的不要求对它用原子操作，而仅仅想确保在某个点，对这些共享变量访问可见的次序一致性。C11提供了一种栅栏操作可满足此需求，其原型为：
```c
void atomic_thread_fence(memory_order order);
```
我们看到，它就一个参数，用于指定当前操作的存储器次序，并且没有指明针对某一对象进行操作，而是在当前点对所有对存储器次序具有依赖性的操作均起作用。
我们后面会谈到**存储器次序依赖性**（**Dependency-ordered**）以及**依赖链**（**dependency chain**）。

<br/>

#### C11标准中的六种存储器次序

下面我们就来详细谈谈这六种存储器次序。这里先介绍C11标准对这些存储器次序的大概定义，因为有些概念会相互穿插，所以把这些存储器次序都列完再做更深入的描述。

1. **`memory_order_relaxed`** ：对当前操作的其他读写不施加任何同步或排序上的约束。如果用此次序的当前操作为原子操作，那么仅仅保证该操作的原子性。
1. **`memory_order_consume`** ：带有此存储器次序的一次加载操作在受影响的存储器位置执行了一次*消费操作*（*consume operation*）：在当前线程中依赖于当前加载值的任何读或写都不能在此加载操作之前重新排序。在其他线程中，释放同一原子变量的对具有数据依赖变量的写在当前线程中是可见的。在大部分平台上，此存储器次序只是影响了编译器优化。另外，消费操作引入了存储器次序依赖性。
1. **`memory_order_acquire`** ：带有此存储器次序的加载操作在受影响的存储器位置执行*获得操作*（*acquire operation*）：在当前线程，没有读和写在此加载之前可以被重新排序。在其他线程中，释放同一原子变量的所有写在当前线程中是可见的。
1. **`memory_order_release`** ：带有此存储器次序的一次存储操作执行*释放操作*（*release operation*）：在当前线程中，没有读和写可以在此存储之后被重新排序。在当前线程中对原子变量的所有写对其他线程中获得同一原子变量的操作是可见的。并且对原子变量携带依赖的写在其他线程中消费同一原子变量的操作也变为可见的。
1. **`memory_order_acq_rel`** ：带有此存储器次序的一次读-修改-写操作同时具备了一次*获得操作*和一次*释放操作*。在当前线程中，没有存储器读和写可以在此存储之前或之后被重新排序。在其他线程中，释放同一原子变量的所有写在此修改前都是可见的（通过当前线程的此操作的acquire语义）；并且此修改对其他线程中获得同一原子变量的操作是可见的（通过当前线程的此操作的release语义）。
1. **`memory_order_seq_cst`** ：带有此存储器次序的一次加载操作执行一个*获得操作*，而一次存储则执行一次*释放操作*，并且一次读-修改-写操作同时执行一次*获得操作*和一次*释放操作*，外加一单个总和次序，在所有线程中均以相同次序观察到对同一原子变量的所有修改。

在以上六种存储器次序中，除了松弛（relax）存储器次序，其他主要围绕着**获得**（**acquire**）语义和**释放**（**release**）语义在讲。我们不需要对这些概念感到恐慌，因为它们其实是非常自然的。从一般程序逻辑上讲，当我们要加载一个多线程共享原子对象时，我们肯定要拿到当前最新的数据（或状态），并且对于具有“获得”语义操作的原子对象往往会以“锁”的形式出现，我们可以回顾一下本文开头时介绍SWAP操作的那段伪代码。这也就意味着在做“获得”语义的时候，我们肯定不想让将作用于共享临界资源的对象在此获得操作之后产生副作用吧？否则的话，在临界区中对该对象的使用可能仍然是无效的。而“释放”语义往往用于伴随着存储操作，我们使用“释放”语义通常可用于释放一个锁，这就使得释放操作后面的那些访存操作不应该被提前到释放操作之前，否则的话也相当于锁失效。

为了帮助大家理解获得语义和释放语义，笔者这里通过“基于锁的”原子操作更形象地帮助大家理解。

![memory-order](https://github.com/zenny-chen/C11-atomic-operations-in-detail/blob/master/memory-order.png)

上图中，虚线箭头表示当前线程的释放操作对其他线程可见。（）里的单词描述了当前操作所使用的存储器次序。如果没有（），则表示使用松弛的存储器次序。
我们可以看到，这里线程B先执行，线程A后执行，然后一开始是在当前上下文中针对某个数组做求和计算，然后把结果给sum。大家注意，这里的sum是在当前线程中独有的，而不是多线程共享的。因此整个操作不采用任何存储器次序，换句话说，其存储器次序是松弛的。
然后到下面，“获得锁”这个操作同时具有“获得”语义和“释放语义”。这里使用获得语义使得前面的对sum对象的赋值操作不会被安排到“获得锁”操作的下面，也就是说，“获得锁”这个操作执行的时候，一定对sum的赋值所产生的副作用可见。这么一来，sum的值对于与之下面的多线程共享原子对象的求和操作确保是有效的。此外，这里的“获得”语义也使得当前的“获得锁”操作能“看见”其他线程对此锁的“释放”操作。而这里使用“释放”语义也是为了告诉其他线程，当前已经把锁给锁了。当然，如果此时上锁失败，那么我们就不需要使用“释放”语义。因此我们看到像C11标准中的`atomic_compare_exchange_weak_explicit`函数原型，对成功和失败各设置了一个存储器次序参数。
再下面对多线程共享原子对象的求和操作也同时用了“获得”语义和“释放”语义。这里使用这两个语义跟当前线程中的操作安排没啥关系，毕竟它前后都有了“获得”语义跟“释放”语义的保护，已经不会被随便安排了，这里主要是对外的可见性。毕竟这里是对多线程共享原子对象的操作，因此这里既要保证该原子对象在当前线程可见到外部线程对它的修改（所以用了获得语义），而且在当前线程对它的修改也要让其他线程可见（所以用了释放语义）。
再下面是“释放锁”操作。这里使用“释放”语义非常自然，一方面在当前线程不让它后续的访存操作被重新安排到它前面去（否则的话，后面的打印结果未必是计算完整的。）；另一方面，当前线程对锁释放后要对其他线程可见。
最后就是对共享原子对象值的获取。这里不需要添加任何存储器次序，因为它前面的释放操作已经确保了本次操作是在整个临界区域结束之后才执行，更术语化地来说，它前面的释放操作确保了之前对该多线程共享的原子操作的计算所产生的副作用对当前操作可见。
