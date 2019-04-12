# Linux 内核中的 Soft 和 Hard Lockup

这周遇到了一个内核关于 `softlockup` 和 `hardlockup` 相关的 bug, 首先
在[内核文档][1]中找到了关于他们的定义和实现的介绍的非常详细，还在网上找到了
更多关于他们的介绍和很细可以查看文后参考的博客

## 1. 首先来介绍下 `softlockup` 和 `hardlockup` 在内核中怎么定义的：

`softlockup` 是导致内核在内核态下循环超过20秒（这个时间是可以通过内核参数设置的）
的错误，而不给**其他任务**提供运行机会。检测时显示当前堆栈跟踪，默认情况下，系统将保持
锁定状态。或者，内核可以配置为 kernel panic; 通过 `sysctl kernel.softlockup_panic`，内
核提供了一个参数 `softlockup_panic`，并为此提供了编译选项 `BOOTPARAM_SOFTLOCKUP_PANIC`。

`hardlockup` 是导致 CPU 在内核态下循环超过10秒的错误，而不会让其他**中断**有机会运行。
与 `softlockup` 情况类似，当检测到时会显示当前堆栈跟踪，除非更改默认行为，否则系统将保持锁定
状态，这个状态也可以通过 `sysctl hardlockup_panic`进行修改，编译内核的选项是`BOOTPARAM_HARDLOCKUP_PANIC`，还有一个和其有关的内核参数 `nmi_watchdog`

panic 选项可以与 panic_timeout 结合使用（可以`kernel.panic` sysctl设置），以使系统在指定的时间后自动重启。

具体相关的详细参数可以参考内核中的 "Documentation/admin-guide/kernel-parameters.rst"

总的说来：

- `softlockup` 是该 CPU 的无法调度到其他的进程运行
- `hardlockup` 是该 CPU 不仅进程无法调度，而且中断也不能运行了（NMI除外， NMI是不可屏蔽的中断）

这里说的 lockup 是指运行在内核态的代码 lockup,用户态的代码是可以被抢占的，还有就是内核代码必须处于
禁止内核抢占的状态(preemption disabled), 因为Linux内核是支持抢占的，只有在一些特定的代码段中才
禁止内核抢占，在这些代码段中才有可嫩发生 lockup

## 2. softlockup 和 hardlockup 原理

soft 和 hard lockup 是基于 `hrtimer` 和 `perf` 子系统实现的

### hardlockup 实现的原理：

`hardlockup` 利用周期性的 `hrtimer` 运行以生成中断并启动监视任务。每个 `watchdog_thresh`
（编译时初始化为 10， 并可通过sysctl进行配置）秒生成 `NMI perf事件`，以检查hardlockup。
如果系统中的任何CPU在此期间没有收到任何 hrtimer 中断，则 `hardlockup detector`（NMI perf事件的处理程序）
将生成内核警告或调用恐慌，具体取决于配置。

### softlockup 实现的原理：

watchdog 任务是一个高优先级内核线程，每次调度时都会更新一个时间戳。如果该时间戳没有更新 `2 * watchdog_thresh` 秒（软锁定阈值），则 `softlockup detector` （在hrtimer回调函数内编码）会将有用的调试信息转储到系统日志中，之后
它将根据系统设置是 调用 panic 或 继续执行其他内核代码

hrtimer 的周期是 `2 * watchdog_thresh / 5`，这意味着它有两到三次机会在硬件锁定检测器启动之前产生中断。

默认情况下，每个激活的 CPU 上都运行一个 watchdog 线程，命名为 [watchdog/%d]。但是，在配置了 `NO_HZ_FULL`
的内核上，默认情况下，watchdog 仅在核心上运行，而不在 `nohz_full` 引导参数中指定的核心上运行。
如果我们允许看门狗默认运行在“nohz_full”内核上，我们必须运行定时器滴答来激活调度程序，这将阻止 `nohz_full`
功能保护这些内核上的用户代码不受内核影响。

当然，默认情况下在 `nohz_full` 核心上禁用它意味着当这些 CPU核 进入内核时，默认情况下我们将无法检测它们是否锁定。
但是，允许 watchdog 继续在 housekeeping（non-tickless）核上运行意味着我们将继续在这些内核上正确检测锁定。

在任何一种情况下，可以通过 `kernel.watchdog_cpumask` sysctl调整排除运行看门狗的核心集。对于 `nohz_full`核，
这可能对调试内核似乎挂在 `nohz_full` 内核上的情况很有用。

```
softlockup_panic=
        [KNL] Should the soft-lockup detector generate panics.
        Format: <integer>

nmi_watchdog=   [KNL,BUGS=X86] Debugging features for SMP kernels
        Format: [panic,][nopanic,][num]
        Valid num: 0 or 1
        0 - turn hardlockup detector in nmi_watchdog off
        1 - turn hardlockup detector in nmi_watchdog on
        When panic is specified, panic when an NMI watchdog
        timeout occurs (or 'nopanic' to override the opposite
        default). To disable both hard and soft lockup detectors,
        please see 'nowatchdog'.
        This is useful when you use a panic=... timeout and
        need the box quickly up again.

watchdog_cpumask:

This value can be used to control on which cpus the watchdog may run.
The default cpumask is all possible cores, but if NO_HZ_FULL is
enabled in the kernel config, and cores are specified with the
nohz_full= boot argument, those cores are excluded by default.
Offline cores can be included in this mask, and if the core is later
brought online, the watchdog will be started based on the mask value.

Typically this value would only be touched in the nohz_full case
to re-enable cores that by default were not running the watchdog,
if a kernel lockup was suspected on those cores.

The argument value is the standard cpulist format for cpumasks,
so for example to enable the watchdog on cores 0, 2, 3, and 4 you
might say:

  echo 0,2-4 > /proc/sys/kernel/watchdog_cpumask
```

## 3. 代码实现与分析

下面我们以 Linux 4.14 的内核版本来分析 Linux 如何实现这两种lockup的探测的：

### soft 和 hard lockup 是基于 `hrtimer` 和 `perf` 子系统实现的

这里我们可以看到要把 watchdog 注册到内核中去，这里的 watchdog 不是硬件的 watchdog，而是通过 NMI 来模拟
一个监控程序起个名字叫 watchdog 而已

从下面的代码可以看到在系统启动转载这个模块的时候，会为每个 CPU core 注册一个 kernel 线程，名字为 `watchdgo/%u`
这个线程会定期调用 watchdog 函数

```c
static struct smp_hotplug_thread watchdog_threads = {
        .store                  = &softlockup_watchdog,
        .thread_should_run      = watchdog_should_run,
        .thread_fn              = watchdog,
        .thread_comm            = "watchdog/%u",
        .setup                  = watchdog_enable,
        .cleanup                = watchdog_cleanup,
        .park                   = watchdog_disable,
        .unpark                 = watchdog_enable,
};

/*
 * Create the watchdog thread infrastructure and configure the detector(s).
 *
 * The threads are not unparked as watchdog_allowed_mask is empty.  When
 * the threads are sucessfully initialized, take the proper locks and
 * unpark the threads in the watchdog_cpumask if the watchdog is enabled.
 */
static __init void lockup_detector_setup(void)
{
        int ret;

        /*
         * If sysctl is off and watchdog got disabled on the command line,
         * nothing to do here.
         */
        lockup_detector_update_enable();

        if (!IS_ENABLED(CONFIG_SYSCTL) &&
            !(watchdog_enabled && watchdog_thresh))
                return;

        ret = smpboot_register_percpu_thread_cpumask(&watchdog_threads,
                                                     &watchdog_allowed_mask);
        if (ret) {
                pr_err("Failed to initialize soft lockup detector threads\n");
                return;
        }

        mutex_lock(&watchdog_mutex);
        softlockup_threads_initialized = true;
        lockup_detector_reconfigure();
        mutex_unlock(&watchdog_mutex);
}

/* Return 0, if a NMI watchdog is available. Error code otherwise */
int __weak __init watchdog_nmi_probe(void)
{
        return hardlockup_detector_perf_init();
}

void __init lockup_detector_init(void)
{
#ifdef CONFIG_NO_HZ_FULL
        if (tick_nohz_full_enabled()) {
                pr_info("Disabling watchdog on nohz_full cores by default\n");
                cpumask_copy(&watchdog_cpumask, housekeeping_mask);
        } else
                cpumask_copy(&watchdog_cpumask, cpu_possible_mask);
#else
        cpumask_copy(&watchdog_cpumask, cpu_possible_mask);
#endif

        if (!watchdog_nmi_probe())
                nmi_watchdog_available = true;
        lockup_detector_setup();
}
```

而 watchdog 函数就是更新一个时间戳为以后使用，在 watchdog 中设置了一个 hrtimer 定时器，并把
watchdog_timer_fn 设置为定时器的处理函数

```c
static void watchdog_enable(unsigned int cpu)
{
        struct hrtimer *hrtimer = this_cpu_ptr(&watchdog_hrtimer);

        /*
         * Start the timer first to prevent the NMI watchdog triggering
         * before the timer has a chance to fire.
         */
        hrtimer_init(hrtimer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
        hrtimer->function = watchdog_timer_fn;
        hrtimer_start(hrtimer, ns_to_ktime(sample_period),
                      HRTIMER_MODE_REL_PINNED);

        /* Initialize timestamp */
        __touch_watchdog();
        /* Enable the perf event */
        if (watchdog_enabled & NMI_WATCHDOG_ENABLED)
                watchdog_nmi_enable(cpu);

        watchdog_set_prio(SCHED_FIFO, MAX_RT_PRIO - 1);
}

/* Commands for resetting the watchdog */
static void __touch_watchdog(void)
{
        __this_cpu_write(watchdog_touch_ts, get_timestamp());
}

static int watchdog_should_run(unsigned int cpu)
{
        return __this_cpu_read(hrtimer_interrupts) !=
                __this_cpu_read(soft_lockup_hrtimer_cnt);
}

/*
 * The watchdog thread function - touches the timestamp.
 *
 * It only runs once every sample_period seconds (4 seconds by
 * default) to reset the softlockup timestamp. If this gets delayed
 * for more than 2*watchdog_thresh seconds then the debug-printout
 * triggers in watchdog_timer_fn().
 */
static void watchdog(unsigned int cpu)
{
        __this_cpu_write(soft_lockup_hrtimer_cnt,
                         __this_cpu_read(hrtimer_interrupts));
        __touch_watchdog();
}
```

watchdog_timer_fn 中做了一些事情：

- 更新 hrtimer_interrupts 计数
- 唤醒 watchdog 线程，该线程会首先调用 watchdog_should_run() 检测 watchdog 是否运行，
只有 hrtimer_interrupts 更新， 该线程才运行

而这个函数的运行周期是多长时间呢？

可以通过初始化的 sample_period 看出是 10 * 2 / 5 = 4s，也就是说内核的 watchdog 线程和
时钟中断函数都是以 4 秒的时间为周期来运行的

```c
int __read_mostly watchdog_thresh = 10;

/*
 * Hard-lockup warnings should be triggered after just a few seconds. Soft-
 * lockups can have false positives under extreme conditions. So we generally
 * want a higher threshold for soft lockups than for hard lockups. So we couple
 * the thresholds with a factor: we make the soft threshold twice the amount of
 * time the hard threshold is.
 */
static int get_softlockup_thresh(void)
{
        return watchdog_thresh * 2;
}

static void set_sample_period(void)
{
        /*
         * convert watchdog_thresh from seconds to ns
         * the divide by 5 is to give hrtimer several chances (two
         * or three with the current relation between the soft
         * and hard thresholds) to increment before the
         * hardlockup detector generates a warning
         */
        sample_period = get_softlockup_thresh() * ((u64)NSEC_PER_SEC / 5);
        watchdog_update_hrtimer_threshold(sample_period);
}

static void watchdog_interrupt_count(void)
{
        __this_cpu_inc(hrtimer_interrupts);
}

/* watchdog kicker functions */
static enum hrtimer_restart watchdog_timer_fn(struct hrtimer *hrtimer)
{
        unsigned long touch_ts = __this_cpu_read(watchdog_touch_ts);
        struct pt_regs *regs = get_irq_regs();
        int duration;
        int softlockup_all_cpu_backtrace = sysctl_softlockup_all_cpu_backtrace;

        if (!watchdog_enabled)
                return HRTIMER_NORESTART;

        /* kick the hardlockup detector */
        watchdog_interrupt_count();

        /* kick the softlockup detector */
        wake_up_process(__this_cpu_read(softlockup_watchdog));

        /* .. and repeat */
        hrtimer_forward_now(hrtimer, ns_to_ktime(sample_period));

        if (touch_ts == 0) {
                if (unlikely(__this_cpu_read(softlockup_touch_sync))) {
                        /*
                         * If the time stamp was touched atomically
                         * make sure the scheduler tick is up to date.
                         */
                        __this_cpu_write(softlockup_touch_sync, false);
                        sched_clock_tick();
                }

                /* Clear the guest paused flag on watchdog reset */
                kvm_check_and_clear_guest_paused();
                __touch_watchdog();
                return HRTIMER_RESTART;
        }
        /* check for a softlockup
         * This is done by making sure a high priority task is
         * being scheduled.  The task touches the watchdog to
         * indicate it is getting cpu time.  If it hasn't then
         * this is a good indication some task is hogging the cpu
         */
        duration = is_softlockup(touch_ts);
        if (unlikely(duration)) {
                /*
                 * If a virtual machine is stopped by the host it can look to
                 * the watchdog like a soft lockup, check to see if the host
                 * stopped the vm before we issue the warning
                 */
                if (kvm_check_and_clear_guest_paused())
                        return HRTIMER_RESTART;

                /* only warn once */
                if (__this_cpu_read(soft_watchdog_warn) == true) {
                        /*
                         * When multiple processes are causing softlockups the
                         * softlockup detector only warns on the first one
                         * because the code relies on a full quiet cycle to
                         * re-arm.  The second process prevents the quiet cycle
                         * and never gets reported.  Use task pointers to detect
                         * this.
                         */
                        if (__this_cpu_read(softlockup_task_ptr_saved) !=
                            current) {
                                __this_cpu_write(soft_watchdog_warn, false);
                                __touch_watchdog();
                        }
                        return HRTIMER_RESTART;
                }

                if (softlockup_all_cpu_backtrace) {
                        /* Prevent multiple soft-lockup reports if one cpu is already
                         * engaged in dumping cpu back traces
                         */
                        if (test_and_set_bit(0, &soft_lockup_nmi_warn)) {
                                /* Someone else will report us. Let's give up */
                                __this_cpu_write(soft_watchdog_warn, true);
                                return HRTIMER_RESTART;
                        }
                }

                pr_emerg("BUG: soft lockup - CPU#%d stuck for %us! [%s:%d]\n",
                        smp_processor_id(), duration,
                        current->comm, task_pid_nr(current));
                __this_cpu_write(softlockup_task_ptr_saved, current);
                print_modules();
                print_irqtrace_events(current);
                if (regs)
                        show_regs(regs);
                else
                        dump_stack();

                if (softlockup_all_cpu_backtrace) {
                        /* Avoid generating two back traces for current
                         * given that one is already made above
                         */
                        trigger_allbutself_cpu_backtrace();

                        clear_bit(0, &soft_lockup_nmi_warn);
                        /* Barrier to sync with other cpus */
                        smp_mb__after_atomic();
                }

                add_taint(TAINT_SOFTLOCKUP, LOCKDEP_STILL_OK);
                if (softlockup_panic)
                        panic("softlockup: hung tasks");
                __this_cpu_write(soft_watchdog_warn, true);
        } else
                __this_cpu_write(soft_watchdog_warn, false);

        return HRTIMER_RESTART;
}
```

该函数通过 is_softlockup() 检测是否发生 softlockup 查看 watchdog_touch_ts 变量在最近20秒的
时间内，有没有被创建的 watchdog 更新过，如果没有就是线程没有调度，说明[watchdog/x]未得到运行机
会，意味着CPU被霸占，调度器没有办法进行调度，也就发生了soft lockup，这种情况下系统可能不会死掉
系统响应会非常慢，可以通过视同 top/ps 命令把占用 CPU 的进程的相应的优先级调低进行测试

```c
static int is_softlockup(unsigned long touch_ts)
{
        unsigned long now = get_timestamp();

        if ((watchdog_enabled & SOFT_WATCHDOG_ENABLED) && watchdog_thresh){
                /* Warn about unreasonable delays. */
                if (time_after(now, touch_ts + get_softlockup_thresh()))
                        return now - touch_ts;
        }
        return 0;
}
```

再看看 watchdog_enable()->wachdog_nmi_enable() 这个函数是打开 hard lockup 在
watchdog_nmi_enable()->hardlockup_detector_perf_enable()->hardlockup_detector_event_create()
中通过 perf_event_create_kernel_counter 注册一个硬件事件 watchdog_overflow_callback()

在 is_hardlockup 这个函数主要就是查看 hrtimer_interrupts 变量在时钟中断处理函数里有没有被更新。
假如没有更新，就意味着中断出了问题，可能被错误代码长时间的关中断了

```c
/* watchdog detector functions */
bool is_hardlockup(void)
{
        unsigned long hrint = __this_cpu_read(hrtimer_interrupts);

        if (__this_cpu_read(hrtimer_interrupts_saved) == hrint)
                return true;

        __this_cpu_write(hrtimer_interrupts_saved, hrint);
        return false;
}

/* Callback function for perf event subsystem */
static void watchdog_overflow_callback(struct perf_event *event,
                                       struct perf_sample_data *data,
                                       struct pt_regs *regs)
{

    /* check for a hardlockup
    * This is done by making sure our timer interrupt
    * is incrementing.  The timer interrupt should have
    * fired multiple times before we overflow'd.  If it hasn't
    * then this is a good indication the cpu is stuck
    */
    if (is_hardlockup()) {
    }
}

static int hardlockup_detector_event_create(void)
{
        unsigned int cpu = smp_processor_id();
        struct perf_event_attr *wd_attr;
        struct perf_event *evt;

        wd_attr = &wd_hw_attr;
        wd_attr->sample_period = hw_nmi_get_sample_period(watchdog_thresh);

        /* Try to register using hardware perf events */
        evt = perf_event_create_kernel_counter(wd_attr, cpu, NULL,
                                               watchdog_overflow_callback, NULL)
        if (IS_ERR(evt)) {
                pr_info("Perf event create on CPU %d failed with %ld\n", cpu,
                        PTR_ERR(evt));
                return PTR_ERR(evt);
        }
        this_cpu_write(watchdog_ev, evt);
        return 0;
}
/*
 * These functions can be overridden if an architecture implements its
 * own hardlockup detector.
 *
 * watchdog_nmi_enable/disable can be implemented to start and stop when
 * softlockup watchdog threads start and stop. The arch must select the
 * SOFTLOCKUP_DETECTOR Kconfig.
 */
int __weak watchdog_nmi_enable(unsigned int cpu)
{
        hardlockup_detector_perf_enable();
        return 0;
}

```

## Reference

1. [Softlockup detector and hardlockup detector (aka nmi_watchdog)][1]
2. [soft lockup和hard lockup介绍][2]
3. [内核如何检测soft lockup与hard lockup？][3]

[1]: https://www.kernel.org/doc/Documentation/lockup-watchdogs.txt "Softlockup detector and hardlockup detector (aka nmi_watchdog)"
[2]: https://blog.csdn.net/panzhenjie/article/details/10074551/ "soft lockup和hard lockup介绍"
[3]: http://linuxperf.com/?p=83 "内核如何检测soft lockup与hard lockup？"
