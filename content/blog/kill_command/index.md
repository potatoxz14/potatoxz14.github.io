---
title: 🎉 Kill 命令
summary: Kill 命令源码分析
date: 2025-08-26

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
  - 源码分析
---
# kill信号处理过程

以 Linux v5.3.1 为例

1. 发送信号: 当用户使用 `kill` 命令或程序调用 `kill()` 系统调用时，操作系统将向指定进程发送一个信号。在内核中，发送信号的过程如下：
   
   a. 首先，内核调用 `kill_something_info()` 函数。这个函数根据给定的进程 ID 或进程组 ID 决定发送信号的目标。
   
   b. 接下来，内核根据目标是单个进程还是进程组，调用 `kill_pid_info()` 或 `group_send_sig_info()` 函数，以发送信号到指定的进程或进程组。

2. 插入信号: 调用`do_send_sig_info()`->`send_signal()` ->`__send_signal()` 函数，将信号插入目标进程的信号队列中。这个队列包含了进程尚未处理的信号。

3. 检查信号队列: 当进程被调度运行时，内核在 `__do_signal()` 函数中检查进程的信号队列，以确定是否有待处理的信号。

4. 处理信号: 当内核获取到待处理的信号时，它会根据进程是否设置了自定义信号处理函数（信号处理器）来采取不同的行动。
   
   a. 如果进程设置了自定义信号处理函数，内核会调用 `handle_signal()` 函数。这个函数会设置信号处理器并在用户栈上保存执行上下文，然后将控制权交给信号处理函数。
   
   b. 如果进程没有设置信号处理器，那么内核会根据信号的默认行为来处理。这包括终止进程、忽略信号等。

5. 终止进程: 如果收到的信号是 SIGTERM（默认的 kill 信号）或 SIGKILL，进程将被终止。内核处理进程终止的过程如下：
   
   a. 首先，内核调用 `do_group_exit()` 函数。这个函数负责释放进程资源，如内存、文件描述符等。
   
   b. 然后，内核调用 `release_task()` 函数，从进程表中移除该进程。
   
   c. 最后，内核调用 `schedule()` 函数来调度其他进程运行，将 CPU 控制权交给其他进程。



# 涉及函数代码

**/kernel/signal.c**

```c
static int kill_something_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
    int ret;

    if (pid > 0) {
        rcu_read_lock();
        ret = kill_pid_info(sig, info, find_vpid(pid));
        rcu_read_unlock();
        return ret;
    }

    /* -INT_MIN is undefined.  Exclude this case to avoid a UBSAN warning */
    if (pid == INT_MIN)
        return -ESRCH;

    read_lock(&tasklist_lock);
    if (pid != -1) {
        ret = __kill_pgrp_info(sig, info,
                pid ? find_vpid(-pid) : task_pgrp(current));
    } else {
        int retval = 0, count = 0;
        struct task_struct * p;

        for_each_process(p) {
            if (task_pid_vnr(p) > 1 &&
                    !same_thread_group(p, current)) {
                int err = group_send_sig_info(sig, info, p,
                                  PIDTYPE_MAX);
                ++count;
                if (err != -EPERM)
                    retval = err;
            }
        }
        ret = count ? retval : -ESRCH;
    }
    read_unlock(&tasklist_lock);

    return ret;
}
```

```c
int kill_pid_info(int sig, struct kernel_siginfo *info, struct pid *pid)
{
    int error = -ESRCH;
    struct task_struct *p;

    for (;;) {
        rcu_read_lock();
        p = pid_task(pid, PIDTYPE_PID);
        if (p)
            error = group_send_sig_info(sig, info, p, PIDTYPE_TGID);
        rcu_read_unlock();
        if (likely(!p || error != -ESRCH))
            return error;

        /*
         * The task was unhashed in between, try again.  If it
         * is dead, pid_task() will return NULL, if we race with
         * de_thread() it will find the new leader.
         */
    }
```

```c
int group_send_sig_info(int sig, struct kernel_siginfo *info,
            struct task_struct *p, enum pid_type type)
{
    int ret;

    rcu_read_lock();
    ret = check_kill_permission(sig, info, p);
    rcu_read_unlock();

    if (!ret && sig)
        ret = do_send_sig_info(sig, info, p, type);

    return ret;
}
```

```c
int do_send_sig_info(int sig, struct kernel_siginfo *info, struct task_struct *p,
            enum pid_type type)
{
    unsigned long flags;
    int ret = -ESRCH;

    if (lock_task_sighand(p, &flags)) {
        ret = send_signal(sig, info, p, type);
        unlock_task_sighand(p, &flags);
    }

    return ret;
}
```

```c
static int send_signal(int sig, struct kernel_siginfo *info, struct task_struct *t,
            enum pid_type type)
{
    /* Should SIGKILL or SIGSTOP be received by a pid namespace init? */
    bool force = false;

    if (info == SEND_SIG_NOINFO) {
        /* Force if sent from an ancestor pid namespace */
        force = !task_pid_nr_ns(current, task_active_pid_ns(t));
    } else if (info == SEND_SIG_PRIV) {
        /* Don't ignore kernel generated signals */
        force = true;
    } else if (has_si_pid_and_uid(info)) {
        /* SIGKILL and SIGSTOP is special or has ids */
        struct user_namespace *t_user_ns;

        rcu_read_lock();
        t_user_ns = task_cred_xxx(t, user_ns);
        if (current_user_ns() != t_user_ns) {
            kuid_t uid = make_kuid(current_user_ns(), info->si_uid);
            info->si_uid = from_kuid_munged(t_user_ns, uid);
        }
        rcu_read_unlock();

        /* A kernel generated signal? */
        force = (info->si_code == SI_KERNEL);

        /* From an ancestor pid namespace? */
        if (!task_pid_nr_ns(current, task_active_pid_ns(t))) {
            info->si_pid = 0;
            force = true;
        }
    }
    return __send_signal(sig, info, t, type, force);
}
```

```c
static int __send_signal(int sig, struct kernel_siginfo *info, struct task_struct *t,
            enum pid_type type, bool force)
{
    struct sigpending *pending;
    struct sigqueue *q;
    int override_rlimit;
    int ret = 0, result;

    assert_spin_locked(&t->sighand->siglock);

    result = TRACE_SIGNAL_IGNORED;
    if (!prepare_signal(sig, t, force))
        goto ret;

    pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;
    /*
     * Short-circuit ignored signals and support queuing
     * exactly one non-rt signal, so that we can get more
     * detailed information about the cause of the signal.
     */
    result = TRACE_SIGNAL_ALREADY_PENDING;
    if (legacy_queue(pending, sig))
        goto ret;

    result = TRACE_SIGNAL_DELIVERED;
    /*
     * Skip useless siginfo allocation for SIGKILL and kernel threads.
     */
    if ((sig == SIGKILL) || (t->flags & PF_KTHREAD))
        goto out_set;

    /*
     * Real-time signals must be queued if sent by sigqueue, or
     * some other real-time mechanism.  It is implementation
     * defined whether kill() does so.  We attempt to do so, on
     * the principle of least surprise, but since kill is not
     * allowed to fail with EAGAIN when low on memory we just
     * make sure at least one signal gets delivered and don't
     * pass on the info struct.
     */
    if (sig < SIGRTMIN)
        override_rlimit = (is_si_special(info) || info->si_code >= 0);
    else
        override_rlimit = 0;

    q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit);
    if (q) {
        list_add_tail(&q->list, &pending->list);
        switch ((unsigned long) info) {
        case (unsigned long) SEND_SIG_NOINFO:
            clear_siginfo(&q->info);
            q->info.si_signo = sig;
            q->info.si_errno = 0;
            q->info.si_code = SI_USER;
            q->info.si_pid = task_tgid_nr_ns(current,
                            task_active_pid_ns(t));
            rcu_read_lock();
            q->info.si_uid =
                from_kuid_munged(task_cred_xxx(t, user_ns),
                         current_uid());
            rcu_read_unlock();
            break;
        case (unsigned long) SEND_SIG_PRIV:
            clear_siginfo(&q->info);
            q->info.si_signo = sig;
            q->info.si_errno = 0;
            q->info.si_code = SI_KERNEL;
            q->info.si_pid = 0;
            q->info.si_uid = 0;
            break;
        default:
            copy_siginfo(&q->info, info);
            break;
        }
    } else if (!is_si_special(info) &&
           sig >= SIGRTMIN && info->si_code != SI_USER) {
        /*
         * Queue overflow, abort.  We may abort if the
         * signal was rt and sent by user using something
         * other than kill().
         */
        result = TRACE_SIGNAL_OVERFLOW_FAIL;
        ret = -EAGAIN;
        goto ret;
    } else {
        /*
         * This is a silent loss of information.  We still
         * send the signal, but the *info bits are lost.
         */
        result = TRACE_SIGNAL_LOSE_INFO;
    }

out_set:
    signalfd_notify(t, sig);
    sigaddset(&pending->signal, sig);

    /* Let multiprocess signals appear after on-going forks */
    if (type > PIDTYPE_TGID) {
        struct multiprocess_signals *delayed;
        hlist_for_each_entry(delayed, &t->signal->multiprocess, node) {
            sigset_t *signal = &delayed->signal;
            /* Can't queue both a stop and a continue signal */
            if (sig == SIGCONT)
                sigdelsetmask(signal, SIG_KERNEL_STOP_MASK);
            else if (sig_kernel_stop(sig))
                sigdelset(signal, SIGCONT);
            sigaddset(signal, sig);
        }
    }

    complete_signal(sig, t, type);
ret:
    trace_signal_generate(sig, info, t, type != PIDTYPE_PID, result);
    return ret;
}
```

**/arch/x86/kernel/signal.c**

```c
void do_signal(struct pt_regs *regs)
{
    struct ksignal ksig;

    if (get_signal(&ksig)) {
        /* Whee! Actually deliver the signal.  */
        handle_signal(&ksig, regs);
        return;
    }

    /* Did we come from a system call? */
    if (syscall_get_nr(current, regs) >= 0) {
        /* Restart the system call - no handlers present */
        switch (syscall_get_error(current, regs)) {
        case -ERESTARTNOHAND:
        case -ERESTARTSYS:
        case -ERESTARTNOINTR:
            regs->ax = regs->orig_ax;
            regs->ip -= 2;
            break;

        case -ERESTART_RESTARTBLOCK:
            regs->ax = get_nr_restart_syscall(regs);
            regs->ip -= 2;
            break;
        }
    }

    /*
     * If there's no signal to deliver, we just put the saved sigmask
     * back.
     */
    restore_saved_sigmask();
}
```

```c
static void
handle_signal(struct ksignal *ksig, struct pt_regs *regs)
{
    bool stepping, failed;
    struct fpu *fpu = &current->thread.fpu;

    if (v8086_mode(regs))
        save_v86_state((struct kernel_vm86_regs *) regs, VM86_SIGNAL);

    /* Are we from a system call? */
    if (syscall_get_nr(current, regs) >= 0) {
        /* If so, check system call restarting.. */
        switch (syscall_get_error(current, regs)) {
        case -ERESTART_RESTARTBLOCK:
        case -ERESTARTNOHAND:
            regs->ax = -EINTR;
            break;

        case -ERESTARTSYS:
            if (!(ksig->ka.sa.sa_flags & SA_RESTART)) {
                regs->ax = -EINTR;
                break;
            }
        /* fallthrough */
        case -ERESTARTNOINTR:
            regs->ax = regs->orig_ax;
            regs->ip -= 2;
            break;
        }
    }

    /*
     * If TF is set due to a debugger (TIF_FORCED_TF), clear TF now
     * so that register information in the sigcontext is correct and
     * then notify the tracer before entering the signal handler.
     */
    stepping = test_thread_flag(TIF_SINGLESTEP);
    if (stepping)
        user_disable_single_step(current);

    failed = (setup_rt_frame(ksig, regs) < 0);
    if (!failed) {
        /*
         * Clear the direction flag as per the ABI for function entry.
         *
         * Clear RF when entering the signal handler, because
         * it might disable possible debug exception from the
         * signal handler.
         *
         * Clear TF for the case when it wasn't set by debugger to
         * avoid the recursive send_sigtrap() in SIGTRAP handler.
         */
        regs->flags &= ~(X86_EFLAGS_DF|X86_EFLAGS_RF|X86_EFLAGS_TF);
        /*
         * Ensure the signal handler starts with the new fpu state.
         */
        fpu__clear(fpu);
    }
    signal_setup_done(failed, ksig, stepping);
}
```

**/kernel/exit.c**

```c
void
do_group_exit(int exit_code)
{
    struct signal_struct *sig = current->signal;

    BUG_ON(exit_code & 0x80); /* core dumps don't get here */

    if (signal_group_exit(sig))
        exit_code = sig->group_exit_code;
    else if (!thread_group_empty(current)) {
        struct sighand_struct *const sighand = current->sighand;

        spin_lock_irq(&sighand->siglock);
        if (signal_group_exit(sig))
            /* Another thread got here before we took the lock.  */
            exit_code = sig->group_exit_code;
        else {
            sig->group_exit_code = exit_code;
            sig->flags = SIGNAL_GROUP_EXIT;
            zap_other_threads(current);
        }
        spin_unlock_irq(&sighand->siglock);
    }

    do_exit(exit_code);
    /* NOTREACHED */
}
```

```c
void release_task(struct task_struct *p)
{
    struct task_struct *leader;
    int zap_leader;
repeat:
    /* don't need to get the RCU readlock here - the process is dead and
     * can't be modifying its own credentials. But shut RCU-lockdep up */
    rcu_read_lock();
    atomic_dec(&__task_cred(p)->user->processes);
    rcu_read_unlock();

    proc_flush_task(p);
    cgroup_release(p);

    write_lock_irq(&tasklist_lock);
    ptrace_release_task(p);
    __exit_signal(p);

    /*
     * If we are the last non-leader member of the thread
     * group, and the leader is zombie, then notify the
     * group leader's parent process. (if it wants notification.)
     */
    zap_leader = 0;
    leader = p->group_leader;
    if (leader != p && thread_group_empty(leader)
            && leader->exit_state == EXIT_ZOMBIE) {
        /*
         * If we were the last child thread and the leader has
         * exited already, and the leader's parent ignores SIGCHLD,
         * then we are the one who should release the leader.
         */
        zap_leader = do_notify_parent(leader, leader->exit_signal);
        if (zap_leader)
            leader->exit_state = EXIT_DEAD;
    }

    write_unlock_irq(&tasklist_lock);
    release_thread(p);
    call_rcu(&p->rcu, delayed_put_task_struct);

    p = leader;
    if (unlikely(zap_leader))
        goto repeat;
}
```

**/kernel/sched/core.c**

```c
asmlinkage __visible void __sched schedule(void)
{
    struct task_struct *tsk = current;

    sched_submit_work(tsk);
    do {
        preempt_disable();
        __schedule(false);
        sched_preempt_enable_no_resched();
    } while (need_resched());
    sched_update_worker(tsk);
}
EXPORT_SYMBOL(schedule);
```
