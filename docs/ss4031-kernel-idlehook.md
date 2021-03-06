# 空闲线程钩子的使用 #

## 例程目的 ##

了解使用空闲线程钩子函数。空闲线程是系统线程中一个比较特殊的线程，它具有最低的优先级，当系统中无其他线程可运行时，调度器将调度到空闲线程。空闲线程通常是一个死循环，永远不被挂起。RT-Thread实时操作系统为空闲线程提供了钩子函数（钩子函数：用户提供的一段代码，在系统运行的某一路径上设置一个钩子，当系统经过这个位置时，转而执行这个钩子函数，然后再返回到它的正常路径上），可以让系统在空闲的时候执行一些特定的任务，例如系统运行指示灯闪烁，电源管理等。

## 程序结构及例程原理 ##

### 程序清单 ###

```{.c}
/*
 * 程序清单：空闲任务钩子例程
 *
 * 这个程序设置了一个空闲函数钩子用于计算CPU使用率，并创建一个线程循环打印CPU使用率
 * 通过修改CPU使用率打印线程中的休眠tick时间可以看到不同的CPU使用率
 */
#include <rtthread.h>
#include <rthw.h>

#if RT_THREAD_PRIORITY_MAX == 8
#define THREAD_PRIORITY        6
#elif RT_THREAD_PRIORITY_MAX == 32
#define THREAD_PRIORITY        25
#elif RT_THREAD_PRIORITY_MAX == 256
#define THREAD_PRIORITY        200
#endif
#define THREAD_STACK_SIZE    512
#define THREAD_TIMESLICE    5

/* 指向线程控制块的指针 */
static rt_thread_t tid = RT_NULL;

#define CPU_USAGE_CALC_TICK    10
#define CPU_USAGE_LOOP        100

static rt_uint8_t  cpu_usage_major = 0, cpu_usage_minor= 0;

/* 记录CPU使用率为0时的总count数 */
static rt_uint32_t total_count = 0;		

/* 空闲任务钩子函数 */
static void cpu_usage_idle_hook()
{
    rt_tick_t tick;
    rt_uint32_t count;
    volatile rt_uint32_t loop;

    if (total_count == 0)
    {
        /* 获取 total_count */
        rt_enter_critical();
        tick = rt_tick_get();
        while(rt_tick_get() - tick < CPU_USAGE_CALC_TICK)
        {
            total_count ++;
            loop = 0;
            while (loop < CPU_USAGE_LOOP) loop ++;
        }
        rt_exit_critical();
    }

    count = 0;
    /* 计算CPU使用率 */
    tick = rt_tick_get();
    while (rt_tick_get() - tick < CPU_USAGE_CALC_TICK)
    {
        count ++;
        loop  = 0;
        while (loop < CPU_USAGE_LOOP) loop ++;
    }

    /* 计算整数百分比整数部分和小数部分 */
    if (count < total_count)
    {
        count = total_count - count;
        cpu_usage_major = (count * 100) / total_count;
        cpu_usage_minor = ((count * 100) % total_count) * 100 / total_count;
    }
    else
    {
        total_count = count;

        /* CPU使用率为0 */
        cpu_usage_major = 0;
        cpu_usage_minor = 0;
    }
}

void cpu_usage_get(rt_uint8_t *major, rt_uint8_t *minor)
{
    RT_ASSERT(major != RT_NULL);
    RT_ASSERT(minor != RT_NULL);

    *major = cpu_usage_major;
    *minor = cpu_usage_minor;
}

/* CPU使用率打印线程入口 */
static void thread_entry(void *parameter)
{
    rt_uint8_t major, minor;

    while(1)
    {
        cpu_usage_get(&major, &minor);
        rt_kprintf("cpu usage: %d.%d%\n", major, minor);

        /* 休眠50个OS Tick */
        /* 手动修改此处休眠 tick 时间，可以模拟实现不同的CPU使用率 */
        rt_thread_delay(50);
    }
}

int cpu_usage_init()
{
    /* 设置空闲线程钩子 */
    rt_thread_idle_sethook(cpu_usage_idle_hook);
	
    /* 创建线程 */
    tid = rt_thread_create("thread",
                            thread_entry, RT_NULL, /* 线程入口是thread_entry, 入口参数是RT_NULL */
                            THREAD_STACK_SIZE, THREAD_PRIORITY, THREAD_TIMESLICE);
    if (tid != RT_NULL)
        rt_thread_startup(tid);
    return 0;
}
INIT_APP_EXPORT(cpu_usage_init);

```

### 例程设计 ###

该例程在 `cpu_usage_init()` 中通过调用```rt_thread_idle_sethook()```设置了一个空闲任务钩子函数```cpu_usage_idle_hook()```用来计算CPU的使用率。
同时创建了一个线程```thread```来循环输出打印CPU使用率，可通过设置```thread```线程中的休眠```tick```时间来实现模拟不同的CPU使用率。

### 编译调试及观察输出信息 ###

仿真运行后，控制台一直循环输出打印CPU使用率:

		\ | /
	- RT -     Thread Operating System
	 / | \     3.0.3 build Apr 21 2018
	 2006 - 2018 Copyright by rt-thread team
	finsh >cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.2%
	cpu usage: 0.0%
	cpu usage: 0.2%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.2%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.2%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.0%
	cpu usage: 0.2%
	cpu usage: 0.2%
	cpu usage: 0.0%
	...
