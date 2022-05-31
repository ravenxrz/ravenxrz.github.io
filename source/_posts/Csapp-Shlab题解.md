---
title: Csapp-Shlab题解
abbrlink: cf3e2591
date: 2020-08-15 19:36:25
categories: Csapp
tags: shlab
---

本次lab, shlab. 即编写一个简单的shell。

## 0. 背景知识

shlab相对其它lab要简单一些，但是其中牵涉出来的一些概念是相当难的。简单说一下这次lab依赖的背景知识吧。

> 书籍对应 异常控制流 章节

1. 理解进程的概念，linux如何创建新进程，进程的状态转换，回收等过程。
2. 理解linux中的signal概念，掌握如何发送、接收、处理signal事件，理解signal handler中的 **Async-Signal-Safety问题及解决方案（也是本次实验的难点）** ，掌握async-signal-safety的guideline.

<!--more-->

## 1. 题目要求

题目的要求主要看shlab的writeup的 《The tsh Specification》小节：

Your tsh shell should have the following features:

1. The prompt should be the string “tsh> ”.
2.  The command line typed by the user should consist of a name and zero or more arguments, all separated by one or more spaces. If name is a built-in command, then tsh should handle it immediately and wait for the next command line. Otherwise, tsh should assume that name is the path of an executable file, which it loads and runs in the context of an initial child process (In this context, the term *job* refers to this initial child process).
3. tsh need not support pipes (|) or I/O redirection (< and >).
4. Typing ctrl-c (ctrl-z) should cause a SIGINT (SIGTSTP) signal to be sent to the current foreground job, as well as any descendents of that job (e.g., any child processes that it forked). If there is no foreground job, then the signal should have no effect.
5. If the command line ends with an ampersand &, then tsh should run the job in the background. Otherwise, it should run the job in the foreground.
6. Each job can be identified by either a process ID (PID) or a job ID (JID), which is a positive integer assigned by tsh. JIDs should be denoted on the command line by the prefix ’%’. For example, “%5” denotes JID 5, and “5” denotes PID 5. (We have provided you with all of the routines you need for manipulating the job list.)

7. tsh should support the following built-in commands:
   – The quit command terminates the shell.
   – The jobs command lists all background jobs.
   – The bg <job> command restarts <job> by sending it a SIGCONT signal, and then runs it inthe background. The <job> argument can be either a PID or a JID.
   – The fg <job> command restarts <job> by sending it a SIGCONT signal, and then runs it in the foreground. The <job> argument can be either a PID or a JID.

8. tsh should reap all of its zombie children. If any job terminates because it receives a signal that it didn’t catch, then tsh should recognize this event and print a message with the job’s PID and a description of the offending signal.

上述是整个shell的功能要求，但是我们不必从0写整个代码，整体代码的框架是已经搭建好了的，只用完成tsh.c文件中的以下几个函数：

1. eval: Main routine that parses and interprets the command line. [70 lines]
2. builtin cmd: Recognizes and interprets the built-in commands: quit, fg, bg, and jobs. [25 lines]
3. do bgfg: Implements the bg and fg built-in commands. [50 lines]
4. waitfg: Waits for a foreground job to complete. [20 lines]
5. sigchld handler: Catches SIGCHILD signals. 80 lines]
6. sigint handler: Catches SIGINT (ctrl-c) signals. [15 lines]
7. sigtstp handler: Catches SIGTSTP (ctrl-z) signals. [15 lines]

一共7个函数，每个函数也给出了建议的行数。

再看下如何验证lab。 shlab提供了tshref，用于参考。同时给出了16个trace file以及一个driver，具体验证过程为， trace file指导driver运行制定shell，查看我们写的shell是否和tshref输出一致。举个例子，执行：

```shell
make test01
```

driver会读取trace01.txt, 在tsh（我们写的shell）下执行trace01.txt中的命令，然后tsh会给出一些输出。此时再执行：

```shell
make rtest01
```

driver会读取trace01.txt 在tshref(参考的shell)下执行trace01.txt中的命令，然后tshref会给出一些输出。对比两次输出，即可知道自己写的shell是否正确了。

当然了，一共16个trace file,每次都make，相当麻烦，所以lab中还给出了 tshref.out文件，这里面给出了所有tshref的输出结果。

## 2. 题目分析

稍微思考下，可以分析出，其实shell就两种命令，一种buildin命令(fg,bg,quit,jobs)，需要立即在shell中执行，一种外部命令，需要fork出新进程来执行，只不过这里需要考虑是前台进程和后台进程。

那先说说**buildin命令**，其中 jobs 已经有默认实现了，quit也相对简单，剩下的就是fg和bg。

那fg和bg做了什么事情呢? 其实代码中已经给出了答案：

```c
/* 
 * Jobs states: FG (foreground), BG (background), ST (stopped)
 * Job state transitions and enabling actions:
 *     FG -> ST  : ctrl-z
 *     ST -> FG  : fg command
 *     ST -> BG  : bg command
 *     BG -> FG  : fg command
 * At most 1 job can be in the FG state.
 */
```

fg命令，无法是将后台（Running 或 Stopped）转换到前台来。

bg命令，将后台Stopped的进程转换为后台Running。

额外的一些工作就是fg和bg的语法：

```shell
fg	%jid
或
fg pid
```

自己通过是否有%来解析就好。

再来看看**外部命令** ， 在parseline函数执行时，会返回当前的命令是前台进程还是后台进程。所以这部分不用我们操心。我们唯一需要做得，就是对前台进程和后台进程”分别处理“。

**这里会用到几个知识点：**

1. 子进程在terminate或stop时，内核会发送SIGCHLD给父进程

2. 父进程需要通过wait族函数回收子进程，否则子进程变为zombie进程，耗费系统资源。

ok，有了两个知识点，思考一下如何对待前台进程和后台进程：

1. 前台进程：最为直接的想法就是父进程调用wait系统调用，等待前台进程结束。但请注意刚才提到的知识点1，前台进程结束后，wait系统调用的确返回了，但同时，父进程会收到SIGCHLD信号，sigchld_handler会被调用。这会带来什么问题？接着看后台进程如何处理。
2. 后台进程：后台进程意味不能阻塞tsh进程，也就意味着我们不能在tsh中调用wait系统调用，那唯一能回收后台进程（除非强制kill）就是当后台进程结束后，父进程会收到SIGHLD信号，通过sigchld_handler来回收，也就意味着我们的sigchld_handler中会有wait系统调用的存在。**ok，结合目前前台进程的处理，就会存在一个严重的问题。**

**前台进程，在tsh.c 中出现了两次wait操作。一次在最开始fork出前台进程时，一次出现在sigchld_handler中。**所以这种情况需要处理，大体思路有两种：

1. 前台进程，fork后立即wait保持不变，在sigchld_handler中过滤掉前台进程的terminate，避免重复wait。
2. 前台进程，fork后的wait变为”伪wait", 前后台进程统一在sigchld_handler中回收处理。那什么是”伪wait“，我们要之前前台进程的意思是什么，作用是什么，作用就是阻塞住当前的tsh，不允许输入新命令，除此之外，前后台进程都是一样的。ok，既然这样，我们只用让tsh阻塞住即可，采用什么方法则看个人实现了。

显然，第二种统一处理更容易理解，维护。当然，writeup其实给出了一种解决方案：

![image-20200805133536133](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200805133536133.png)

另外一个问题就是，**信号的转发**。

![image-20200805133902911](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200805133902911.png)

这里要解决的是，后台进程不应该接收到Ctrl-C所引发的信号。

还有一个问题就是**保持和tshref的输出相同**，那需要在恰当的位置，打印出相应的信息，这部分不难，但其实挺磨人的。

最后就是非tsh需求的逻辑部分了，那就是**如何编写 async-signal-safety 的代码**。如果你是看了csapp课程或书籍的话， 那应该可以意识这里会有一些async-signal-safety问题。ppt中其实基本上给出所有我们会遇到的问题，以及一些常用的guideline。这部分需要仔细阅读，理解清楚，必然很可能出现一些莫名的bug。

下面贴出我个人的代码：

```c
/* 
 * tsh - A tiny shell program with job control
 * 
 * <Put your name and login ID here>
 */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <ctype.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <errno.h>

/* Misc manifest constants */
#define MAXLINE 1024   /* max line size */
#define MAXARGS 128    /* max args on a command line */
#define MAXJOBS 16     /* max jobs at any point in time */
#define MAXJID 1 << 16 /* max job ID */

/* Job states */
#define UNDEF 0 /* undefined */
#define FG 1    /* running in foreground */
#define BG 2    /* running in background */
#define ST 3    /* stopped */

/* 
 * Jobs states: FG (foreground), BG (background), ST (stopped)
 * Job state transitions and enabling actions:
 *     FG -> ST  : ctrl-z
 *     ST -> FG  : fg command
 *     ST -> BG  : bg command
 *     BG -> FG  : fg command
 * At most 1 job can be in the FG state.
 */

/* Global variables */
extern char **environ;   /* defined in libc */
char prompt[] = "tsh> "; /* command line prompt (DO NOT CHANGE) */
int verbose = 0;         /* if true, print additional output */
int nextjid = 1;         /* next job ID to allocate */
char sbuf[MAXLINE];      /* for composing sprintf messages */

struct job_t
{                          /* The job struct */
    pid_t pid;             /* job PID */
    int jid;               /* job ID [1, 2, ...] */
    int state;             /* UNDEF, BG, FG, or ST */
    char cmdline[MAXLINE]; /* command line */
};
struct job_t jobs[MAXJOBS]; /* The job list */
/* End global variables */

/* Function prototypes */

/* Here are the functions that you will implement */
void eval(char *cmdline);
int builtin_cmd(char **argv);
void do_bgfg(char **argv);
void waitfg(pid_t pid);

void sigchld_handler(int sig);
void sigtstp_handler(int sig);
void sigint_handler(int sig);

/* Here are helper routines that we've provided for you */
int parseline(const char *cmdline, char **argv);
void sigquit_handler(int sig);

void clearjob(struct job_t *job);
void initjobs(struct job_t *jobs);
int maxjid(struct job_t *jobs);
int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline);
int deletejob(struct job_t *jobs, pid_t pid);
pid_t fgpid(struct job_t *jobs);
struct job_t *getjobpid(struct job_t *jobs, pid_t pid);
struct job_t *getjobjid(struct job_t *jobs, int jid);
int pid2jid(pid_t pid);
void listjobs(struct job_t *jobs);

void usage(void);
void unix_error(char *msg);
void app_error(char *msg);
typedef void handler_t(int);
handler_t *Signal(int signum, handler_t *handler);

/*
 * main - The shell's main routine 
 */
int main(int argc, char **argv)
{
    char c;
    char cmdline[MAXLINE];
    int emit_prompt = 1; /* emit prompt (default) */

    /* Redirect stderr to stdout (so that driver will get all output
     * on the pipe connected to stdout) */
    dup2(1, 2);

    /* Parse the command line */
    while ((c = getopt(argc, argv, "hvp")) != EOF)
    {
        switch (c)
        {
        case 'h': /* print help message */
            usage();
            break;
        case 'v': /* emit additional diagnostic info */
            verbose = 1;
            break;
        case 'p':            /* don't print a prompt */
            emit_prompt = 0; /* handy for automatic testing */
            break;
        default:
            usage();
        }
    }

    /* Install the signal handlers */

    /* These are the ones you will need to implement */
    Signal(SIGINT, sigint_handler);   /* ctrl-c */
    Signal(SIGTSTP, sigtstp_handler); /* ctrl-z */
    Signal(SIGCHLD, sigchld_handler); /* Terminated or stopped child */

    /* This one provides a clean way to kill the shell */
    Signal(SIGQUIT, sigquit_handler);

    /* Initialize the job list */
    initjobs(jobs);

    /* Execute the shell's read/eval loop */
    while (1)
    {

        /* Read command line */
        if (emit_prompt)
        {
            printf("%s", prompt);
            fflush(stdout);
        }
        if ((fgets(cmdline, MAXLINE, stdin) == NULL) && ferror(stdin))
            app_error("fgets error");
        if (feof(stdin))
        { /* End of file (ctrl-d) */
            fflush(stdout);
            exit(0);
        }

        /* Evaluate the command line */
        eval(cmdline);
        fflush(stdout);
        fflush(stdout);
    }

    exit(0); /* control never reaches here */
}

/* 
 * eval - Evaluate the command line that the user has just typed in
 * 
 * If the user has requested a built-in command (quit, jobs, bg or fg)
 * then execute it immediately. Otherwise, fork a child process and
 * run the job in the context of the child. If the job is running in
 * the foreground, wait for it to terminate and then return.  Note:
 * each child process must have a unique process group ID so that our
 * background children don't receive SIGINT (SIGTSTP) from the kernel
 * when we type ctrl-c (ctrl-z) at the keyboard.  
*/
void eval(char *cmdline)
{
    int bg;                               // 是否是后台程序
    char buf[MAXLINE];                    // cmd copy
    char *argv[MAXARGS];                  // 解析后的命令
    pid_t pid = -1;                       // 子进程id
    sigset_t mask_all, mask_one, pre_one; // 用于同步

    strcpy(buf, cmdline);
    bg = parseline(cmdline, argv);
    if (argv[0] == NULL)
        return;

    // 初始化blocking mask
    sigfillset(&mask_all);
    sigemptyset(&mask_one);
    sigaddset(&mask_one, SIGCHLD);

    // 正式执行命令
    if (!builtin_cmd(argv))
    {
        // 外部命令
        if (access(argv[0], F_OK) == -1)
        {
            printf("%s: Command not found\n", argv[0]);
            return;
        }

        // 能够运行的外部命令
        // Parent: 在fork前，block SIGCHLD 信号，避免在addjob之前Child结束，触发sigchld_handler,从而deltejob比addjob先执行
        sigprocmask(SIG_BLOCK, &mask_one, &pre_one);
        if ((pid = fork()) == 0)
        {                  // Child
            setpgid(0, 0); // 改变组id，避免信号的自动传送(write up中有更相似的描述)
            // 执行exec之前，解除从Parent中继承过来的block_mask
            sigprocmask(SIG_SETMASK, &pre_one, NULL);
            // 执行
            if (execve(argv[0], argv, environ) == -1)
            {
                unix_error("execve child process error");
            }
        }
        else
        { // Parent
            // addjob, block所有信号，避免竞争问题
            sigprocmask(SIG_SETMASK, &mask_all, NULL);
            addjob(jobs, pid,
                   bg ? BG : FG,
                   buf);
            // 还原信号
            sigprocmask(SIG_SETMASK, &pre_one, NULL);
        }

        // Parent: 前台进程处理
        if (!bg)
        {
            waitfg(pid);
        }
        else
        {
            // TODO: 在这里输出，耦合了addjob函数(内部对nextjid++)，如何优化？
            printf("[%d] (%d) %s &\n", nextjid - 1, pid, argv[0]);
        }
    }
    return;
}

/* 
 * parseline - Parse the command line and build the argv array.
 * 
 * Characters enclosed in single quotes are treated as a single
 * argument.  Return true if the user has requested a BG job, false if
 * the user has requested a FG job.  
 */
int parseline(const char *cmdline, char **argv)
{
    static char array[MAXLINE]; /* holds local copy of command line */
    char *buf = array;          /* ptr that traverses command line */
    char *delim;                /* points to first space delimiter */
    int argc;                   /* number of args */
    int bg;                     /* background job? */

    strcpy(buf, cmdline);
    buf[strlen(buf) - 1] = ' ';   /* replace trailing '\n' with space */
    while (*buf && (*buf == ' ')) /* ignore leading spaces */
        buf++;

    /* Build the argv list */
    argc = 0;
    if (*buf == '\'')
    {
        buf++;
        delim = strchr(buf, '\'');
    }
    else
    {
        delim = strchr(buf, ' ');
    }

    while (delim)
    {
        argv[argc++] = buf;
        *delim = '\0';
        buf = delim + 1;
        while (*buf && (*buf == ' ')) /* ignore spaces */
            buf++;

        if (*buf == '\'')
        {
            buf++;
            delim = strchr(buf, '\'');
        }
        else
        {
            delim = strchr(buf, ' ');
        }
    }
    argv[argc] = NULL;

    if (argc == 0) /* ignore blank line */
        return 1;

    /* should the job run in the background? */
    if ((bg = (*argv[argc - 1] == '&')) != 0)
    {
        argv[--argc] = NULL;
    }
    return bg;
}

/* 
 * builtin_cmd - If the user has typed a built-in command then execute
 *    it immediately.  
 */
int builtin_cmd(char **argv)
{
    char *cmd = argv[0];
    // 检查合法性
    if (!cmd || !strlen(cmd))
    {
        printf("Cmmond not found");
    }

    // 判定4种内置命令
    if (!strcmp("quit", cmd))
    {
        // 考虑当前还有后台进程，需要kill
        sigset_t mask_all, pre_one;
        sigfillset(&mask_all);
        sigprocmask(SIG_SETMASK, &mask_all, &pre_one);
        // 先kill 所有子进程
        for (int i = 0; i < MAXJOBS; i++)
        {
            if (jobs[i].pid)
            {
                kill(jobs[i].pid, SIGKILL);
            }
        }
        clearjob(jobs);
        sigprocmask(SIG_SETMASK, &pre_one, NULL);
        // 退出
        exit(0);
    }
    else if (!strcmp("jobs", cmd))
    {
        listjobs(jobs);
    }
    else if (!strcmp("fg", cmd) || !strcmp("bg", cmd))
    {
        do_bgfg(argv);
    }
    else
    {
        return 0; /* not a builtin command */
    }
    return 1;
}

/* 
 * do_bgfg - Execute the builtin bg and fg commands
 */
void do_bgfg(char **argv)
{
    // NOTE:本函数假设argv[0] 只能 = fg 或者 = bg
    char *arg = argv[1]; // 命令参数
    int jid = -1, pid = -1;
    struct job_t *job;
    sigset_t mask_all, pre_one;

    // 命令是否有效？
    if (argv[1] == NULL)
    {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    }

    // 判断参数后续是否全是数字
    if (arg[0] == '%')
    {
        arg++;
    }
    while (*arg != '\0')
    {
        if (!isdigit(*arg))
        {
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }
        arg++;
    }
    arg = argv[1]; // 还原

    // 获取job
    if (arg[0] == '%')
    {
        // jid mod
        jid = atoi(arg + 1);
    }
    else
    {
        // pid mod
        pid = atoi(arg);
        jid = pid2jid(pid);
    }
    job = getjobjid(jobs, jid);
    if (!job)
    {
        if (arg[0] != '%')
        {
            printf("(%d): No such process\n", pid);
        }
        else
        {
            printf("%%%d: No such job\n", jid);
        }
        return;
    }

    if (!strcmp("fg", argv[0]))
    {
        sigprocmask(SIG_SETMASK, &mask_all, &pre_one);
        // fg mode
        // 后台进程转前台
        if (job->state == FG)
        {
            printf("process is foreground already\n");
            return;
        }
        else if (job->state == ST)
        {
            // ST -> RUNNING
            // TODO: 测试后发现，发送SIGCONT后，也会触发一次SIGCHLD,目前原因未知
            kill(job->pid, SIGCONT);
        }

        job->state = FG;
        sigprocmask(SIG_SETMASK, &pre_one, 0);
        waitfg(job->pid);
    }
    else if (!strcmp("bg", argv[0]))
    { // bg mode
        sigprocmask(SIG_SETMASK, &mask_all, &pre_one);
        // 后台stop进程转running进程
        if (job->state != ST)
        {
            printf("process isn't stoped\n");
            return;
        }

        kill(job->pid, SIGCONT);
        job->state = BG;
        sigprocmask(SIG_SETMASK, &pre_one, 0);
        // TODO: 这里的printf后面没有加\n，因为cmdline自带\n
        printf("[%d] (%d) %s", job->jid, job->pid, job->cmdline);
    }

    return;
}

/* 
 * waitfg - Block until process pid is no longer the foreground process
 */
void waitfg(pid_t pid)
{
    // Note: 因为前台进程结束后，会自动调用chld_handler,所以回收工作交给chld_handler处理
    // 这里只用保证tsh被阻塞即可。
    sigset_t mask_all, pre_one;
    // 初始化blocking mask
    sigfillset(&mask_all);

    while (1)
    { // 从全局变量jobs中读取 fg pid，避免jobs的竞争
        sigprocmask(SIG_SETMASK, &mask_all, &pre_one);
        if (!fgpid(jobs))
        {
            // 没有前台进程
            break;
        }
        sigprocmask(SIG_SETMASK, &pre_one, 0);
        sleep(1);
    }
    sigprocmask(SIG_SETMASK, &pre_one, 0);
    return;
}

/*****************
 * Signal handlers
 *****************/

/* 
 * sigchld_handler - The kernel sends a SIGCHLD to the shell whenever
 *     a child job terminates (becomes a zombie), or stops because it
 *     received a SIGSTOP or SIGTSTP signal. The handler reaps all
 *     available zombie children, but doesn't wait for any other
 *     currently running children to terminate.  
 */
void sigchld_handler(int sig)
{
    int olderrno = errno;
    sigset_t mask_all, pre_one;
    int pid, wstate;
    struct job_t *job; // 触发本次sigchld_handler的job

    sigfillset(&mask_all);

    pid = waitpid(-1, &wstate, WNOHANG | WUNTRACED);
    job = getjobpid(jobs, pid);
    if (!job)
    {
        return;
    }

    // waitpid调用返回有两种情况：
    // 1. 子进程terminate
    // 2. 通过信号，被stop了，如用户键入 ctrl+z
    sigprocmask(SIG_SETMASK, &mask_all, &pre_one);
    if (WIFSTOPPED(wstate))
    { 
        // stoped，更新状态即可
        // TODO: 不应该在handler中出现printf这类async-unsafe, 替换为safe-library即可
        printf("Job [%d] (%d) stoped by signal %d\n", job->jid, job->pid, SIGTSTP);
        job->state = ST;
    }
    else
    {   // terminate，需要deletejob
        if (WIFSIGNALED(wstate))
        {
            // TODO: 不应该在handler中出现printf这类async-unsafe, 替换为safe-library即可
            printf("Job [%d] (%d) terminated by signal %d\n", job->jid, job->pid, SIGINT);
        }
        deletejob(jobs, pid);
    }
    sigprocmask(SIG_SETMASK, &pre_one, NULL);
    errno = olderrno;

    return;
}

/* 
 * sigint_handler - The kernel sends a SIGINT to the shell whenver the
 *    user types ctrl-c at the keyboard.  Catch it and send it along
 *    to the foreground job.  
 */
void sigint_handler(int sig)
{
    int olderrno = errno;
    int pid;
    struct job_t *job;

    pid = fgpid(jobs);
    if (!pid)
    {
        return;
    }
    job = getjobpid(jobs, pid);
    if (!job)
    {
        return;
    }

    // 有前台进程, forward to it， 后续处理交给chld_handler
    // 注意传递INT信号可能会造成死循环：具体参考https://blog.csdn.net/guozhiyingguo/article/details/53837424
    // 同时注意，这里kill需要发送给整个进程组
    kill(-job->pid, SIGINT);

    errno = olderrno;
    return;
}

/*
 * sigtstp_handler - The kernel sends a SIGTSTP to the shell whenever
 *     the user types ctrl-z at the keyboard. Catch it and suspend the
 *     foreground job by sending it a SIGTSTP.  
 */
void sigtstp_handler(int sig)
{
    int olderrno = errno;
    int pid;
    struct job_t *job;

    pid = fgpid(jobs);
    if (!pid)
    {
        return;
    }
    job = getjobpid(jobs, pid);
    if (!job)
    {
        return;
    }

    // 有前台进程, forward to it， 后续处理交给chld_handler
    kill(job->pid, SIGTSTP);

    errno = olderrno;
}

/*********************
 * End signal handlers
 *********************/

/***********************************************
 * Helper routines that manipulate the job list
 **********************************************/

/* clearjob - Clear the entries in a job struct */
void clearjob(struct job_t *job)
{
    job->pid = 0;
    job->jid = 0;
    job->state = UNDEF;
    job->cmdline[0] = '\0';
}

/* initjobs - Initialize the job list */
void initjobs(struct job_t *jobs)
{
    int i;

    for (i = 0; i < MAXJOBS; i++)
        clearjob(&jobs[i]);
}

/* maxjid - Returns largest allocated job ID */
int maxjid(struct job_t *jobs)
{
    int i, max = 0;

    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].jid > max)
            max = jobs[i].jid;
    return max;
}

/* addjob - Add a job to the job list */
int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline)
{
    int i;

    if (pid < 1)
        return 0;

    for (i = 0; i < MAXJOBS; i++)
    {
        if (jobs[i].pid == 0)
        {
            jobs[i].pid = pid;
            jobs[i].state = state;
            jobs[i].jid = nextjid++;
            if (nextjid > MAXJOBS)
                nextjid = 1;
            strcpy(jobs[i].cmdline, cmdline);
            if (verbose)
            {
                printf("Added job [%d] %d %s\n", jobs[i].jid, jobs[i].pid, jobs[i].cmdline);
            }
            return 1;
        }
    }
    printf("Tried to create too many jobs\n");
    return 0;
}

/* deletejob - Delete a job whose PID=pid from the job list */
int deletejob(struct job_t *jobs, pid_t pid)
{
    int i;

    if (pid < 1)
        return 0;

    for (i = 0; i < MAXJOBS; i++)
    {
        if (jobs[i].pid == pid)
        {
            clearjob(&jobs[i]);
            nextjid = maxjid(jobs) + 1;
            return 1;
        }
    }
    return 0;
}

/* fgpid - Return PID of current foreground job, 0 if no such job */
pid_t fgpid(struct job_t *jobs)
{
    int i;

    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].state == FG)
            return jobs[i].pid;
    return 0;
}

/* getjobpid  - Find a job (by PID) on the job list */
struct job_t *getjobpid(struct job_t *jobs, pid_t pid)
{
    int i;

    if (pid < 1)
        return NULL;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].pid == pid)
            return &jobs[i];
    return NULL;
}

/* getjobjid  - Find a job (by JID) on the job list */
struct job_t *getjobjid(struct job_t *jobs, int jid)
{
    int i;

    if (jid < 1)
        return NULL;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].jid == jid)
            return &jobs[i];
    return NULL;
}

/* pid2jid - Map process ID to job ID */
int pid2jid(pid_t pid)
{
    int i;

    if (pid < 1)
        return 0;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].pid == pid)
        {
            return jobs[i].jid;
        }
    return 0;
}

/* listjobs - Print the job list */
void listjobs(struct job_t *jobs)
{
    int i;

    for (i = 0; i < MAXJOBS; i++)
    {
        if (jobs[i].pid != 0)
        {
            printf("[%d] (%d) ", jobs[i].jid, jobs[i].pid);
            switch (jobs[i].state)
            {
            case BG:
                printf("Running ");
                break;
            case FG:
                printf("Foreground ");
                break;
            case ST:
                printf("Stopped ");
                break;
            default:
                printf("listjobs: Internal error: job[%d].state=%d ",
                       i, jobs[i].state);
            }
            printf("%s", jobs[i].cmdline);
        }
    }
}
/******************************
 * end job list helper routines
 ******************************/

/***********************
 * Other helper routines
 ***********************/

/*
 * usage - print a help message
 */
void usage(void)
{
    printf("Usage: shell [-hvp]\n");
    printf("   -h   print this message\n");
    printf("   -v   print additional diagnostic information\n");
    printf("   -p   do not emit a command prompt\n");
    exit(1);
}

/*
 * unix_error - unix-style error routine
 */
void unix_error(char *msg)
{
    fprintf(stdout, "%s: %s\n", msg, strerror(errno));
    exit(1);
}

/*
 * app_error - application-style error routine
 */
void app_error(char *msg)
{
    fprintf(stdout, "%s\n", msg);
    exit(1);
}

/*
 * Signal - wrapper for the sigaction function
 */
handler_t *Signal(int signum, handler_t *handler)
{
    struct sigaction action, old_action;

    action.sa_handler = handler;
    sigemptyset(&action.sa_mask); /* block sigs of type being handled */
    action.sa_flags = SA_RESTART; /* restart syscalls if possible */

    if (sigaction(signum, &action, &old_action) < 0)
        unix_error("Signal error");
    return (old_action.sa_handler);
}

/*
 * sigquit_handler - The driver program can gracefully terminate the
 *    child shell by sending it a SIGQUIT signal.
 */
void sigquit_handler(int sig)
{
    printf("Terminating after receipt of SIGQUIT signal\n");
    exit(1);
}
```