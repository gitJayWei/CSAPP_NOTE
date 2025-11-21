## Shell Lab

> author: Jay Wei@gitjaywei

前言：强烈建议先看完csapp第八章再做此实验，完整的tsh.c代码贴在文章末尾了

##### 1.准备知识 

1.  进程的概念、状态以及控制进程的几个函数（fork,waitpid,execve）。
1.  信号的概念，会编写正确安全的信号处理程序。
1.  shell的概念，理解shell程序是如何利用进程管理和信号去执行一个命令行语句。

##### 2.实验目的 

shell lab主要目的是为了熟悉进程控制和信号。具体来说需要比对16个test和rtest文件的输出，实现七个函数：

```java
void eval(char *cmdline)：分析命令，并派生子进程执行 主要功能是解析cmdline并运行
int builtin_cmd(char **argv)：解析和执行bulidin命令，包括 quit, fg, bg, and jobs
void do_bgfg(char **argv) 执行bg和fg命令
void waitfg(pid_t pid)：实现阻塞等待前台程序运行结束
void sigchld_handler(int sig)：SIGCHID信号处理函数
void sigint_handler(int sig)：信号处理函数，响应 SIGINT (ctrl-c) 信号 
void sigtstp_handler(int sig)：信号处理函数，响应 SIGTSTP (ctrl-z) 信号
```

##### 3.实验内容及操作步骤 

通过阅读实验指导书我们知道此实验要求我们完成tsh.c中的七个函数从而实现一个简单的shell，能够处理前后台运行程序、能够处理ctrl+z、ctrl+c等信号。

首先我们来看一下tsh.c具体内容。

首先定义了一些宏

```java
/* 定义了一些宏 */
#define MAXLINE    1024   /* max line size */
#define MAXARGS     128   /* max args on a command line */
#define MAXJOBS      16   /* max jobs at any point in time */
#define MAXJID    1<<16   /* max job ID */
```

定义了四种进程状态

```java
/* 工作状态 */
#define UNDEF 0 /* undefined */
#define FG 1    /* 前台状态 */
#define BG 2    /* 后台状态 */
#define ST 3    /* 挂起状态 */
```

然后定义了job\_t的任务的类，并且创建了jobs\[\]数组

```java
struct job_t {
            
   
     
                   /* The job struct */
    pid_t pid;              /* job PID */
    int jid;                /* job ID [1, 2, ...] */
    int state;              /* UNDEF, BG, FG, or ST */
    char cmdline[MAXLINE];  /* command line */
};
struct job_t jobs[MAXJOBS]; /* The job list */
```

接着是需要我们完成的七个函数定义

```java
void eval(char *cmdline)：分析命令，并派生子进程执行 主要功能是解析cmdline并运行
int builtin_cmd(char **argv)：解析和执行bulidin命令，包括 quit, fg, bg, and jobs
void do_bgfg(char **argv) 执行bg和fg命令
void waitfg(pid_t pid)：实现阻塞等待前台程序运行结束
void sigchld_handler(int sig)：SIGCHID信号处理函数
void sigint_handler(int sig)：信号处理函数，响应 SIGINT (ctrl-c) 信号 
void sigtstp_handler(int sig)：信号处理函数，响应 SIGTSTP (ctrl-z) 信号
```

下面就是一些辅助的函数

```java
int parseline(const char *cmdline, char **argv);   //获取参数列表，返回是否为后台运行命令
void sigquit_handler(int sig);  //处理SIGQUIT信号
void clearjob(struct job_t *job);  //清除job结构体 
void initjobs(struct job_t *jobs);  //初始化任务jobs[]
int maxjid(struct job_t *jobs);   //返回jobs链表中最大的jid号。
int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline);  //向jobs[]添加一个任务
int deletejob(struct job_t *jobs, pid_t pid);   //在jobs[]中删除pid的job
pid_t fgpid(struct job_t *jobs);  //返回当前前台运行job的pid号
struct job_t *getjobpid(struct job_t *jobs, pid_t pid);  //根据pid找到对应的job 
struct job_t *getjobjid(struct job_t *jobs, int jid);   //根据jid找到对应的job 
int pid2jid(pid_t pid);   //根据pid找到jid 
void listjobs(struct job_t *jobs);  //打印jobs
```

接着就是main函数，作用是在文件中逐行获取命令，并且判断是不是文件结束（EOF），将命令cmdline送入eval函数进行解析。我们需要做的就是逐步完善这个过程

接下来开始实验：

1.  使用make命令编译tsh.c文件（文件有所改变的话需要先使用make clean指令清空）

```java
bo@bo:~/shlab-handout$ make
gcc -Wall -O2    tsh.c   -o tsh
gcc -Wall -O2    myspin.c   -o myspin
gcc -Wall -O2    mysplit.c   -o mysplit
gcc -Wall -O2    mystop.c   -o mystop
gcc -Wall -O2    myint.c   -o myint
```

1.  使用make testXX指令比较traceXX.txt文件在编写的shell和reference shell的运行结果；或者也可以使用”./sdriver.pl -t traceXX.txt -s ./tsh -a “-p”
1.  如果在文件名前面加上r，则是执行标准的tshref，或者将tsh变为tshref。通过比对标准tshref和自制tsh的执行结果结果，可以观察tsh的功能是否正确。如果tsh的执行结果和tshref结果一致，说明结果是正确的

接下来我们开始补充函数

**eval（）函数**

函数功能：eval函数用于解析和解释命令行。eval首先解析命令行，如果用户请求一个内置命令quit、jobs、bg或fg（即内置命令）那么就立即执行。否则，fork子进程和在子进程的上下文中运行作业。如果作业正在运行前台，等待它终止，然后返回。

函数原型：`void eval(char *cmdline)`，传入的参数为cmdline，即命令行字符串

实现思路：仿照书上的eval函数写法和所需的功能来完成函数

1.  首先调用parseline函数解析命令行，如果为空直接返回，接着使用builtin\_cmd函数判断是否为内置命令，返回0说明不是内置命令，如果是内置命令直接执行。
1.  如果不是内置命令，那么先阻塞信号（具体在第四点分析），再调用fork创建子进程。在子进程中，首先解除阻塞，设置自己的id号，然后调用execve函数来执行job。
1.  父进程判断作业是否后台运行，是的话调用addjob函数将子进程job加入job链表中，解除阻塞，然后调用waifg函数等待前台运行完成。如果不在后台工作则打印进程组jid和子进程pid以及命令行字符串。
1.  因为子进程继承了他们父进程的阻塞向量，所以在执行新程序之前，子程序必须确保解除对SIGCHLD信号的阻塞。父进程必须使用sigprocmask在它派生子进程之前也就是调用fork函数之前阻塞SIGCHLD信号，之后解除阻塞；在通过调用addjob将子进程添加到作业列表之后，再次使用sigprocmask，解除阻塞。

完整代码：

```java
/**
 * 处理命令行输入，解析并执行命令（内置命令直接执行，外部命令通过子进程执行）
 * 管理进程前后台状态，处理信号竞争问题
 * @param cmdline 用户输入的命令行字符串
 */
void eval(char *cmdline)
{
    char* argv[MAXARGS];   // 存储解析后的命令及参数，供execve()函数使用
    int state = UNDEF;     // 进程状态：FG（前台）或BG（后台），初始为未定义
    sigset_t set;          // 信号集，用于阻塞/解除阻塞特定信号
    pid_t pid;             // 子进程ID

    // 1. 解析命令行，确定进程前后台状态
    // parseline函数：拆分cmdline到argv数组，返回1表示后台运行（以&结尾），0表示前台
    if(parseline(cmdline, argv) == 1)  
        state = BG;        // 后台进程
    else
        state = FG;        // 前台进程

    // 2. 处理空命令（如用户仅输入回车）
    if(argv[0] == NULL)    // 解析后无有效命令
        return;

    // 3. 若不是内置命令，则通过子进程执行外部命令
    if(!builtin_cmd(argv))  // builtin_cmd返回1表示内置命令（如cd、exit），直接在shell进程执行
    {
        // 4. 初始化信号集，准备阻塞关键信号（避免竞争条件）
        // 清空信号集
        if(sigemptyset(&set) < 0)
            unix_error("sigemptyset error");  // 错误处理函数（输出错误信息并退出）
        
        // 向信号集添加需要阻塞的信号：
        // SIGINT（Ctrl+C，中断信号）、SIGTSTP（Ctrl+Z，暂停信号）、SIGCHLD（子进程状态改变信号）
        if(sigaddset(&set, SIGINT) < 0 || 
           sigaddset(&set, SIGTSTP) < 0 || 
           sigaddset(&set, SIGCHLD) < 0)
            unix_error("sigaddset error");
        
        // 阻塞信号集中的信号：确保子进程创建并加入作业列表前，不会因信号处理导致状态不一致
        if(sigprocmask(SIG_BLOCK, &set, NULL) < 0)
            unix_error("sigprocmask error");

        // 5. 创建子进程执行外部命令
        if((pid = fork()) < 0)  // fork失败
            unix_error("fork error");
        else if(pid == 0)       // 子进程逻辑
        {
            // 子进程继承父进程的信号阻塞集，需解除阻塞以响应信号
            if(sigprocmask(SIG_UNBLOCK, &set, NULL) < 0)
                unix_error("sigprocmask error");
            
            // 设置子进程为新的进程组组长（进程组ID=子进程ID）
            // 目的：与shell进程组隔离，避免信号（如Ctrl+C）误影响shell
            if(setpgid(0, 0) < 0)
                unix_error("setpgid error");
            
            // 加载并执行外部命令：替换子进程的代码段为目标程序
            // argv[0]为命令路径，argv为参数列表，environ为环境变量
            if(execve(argv[0], argv, environ) < 0){
                printf("%s:Command not found\n", argv[0]);  // 命令不存在时提示
                exit(0);  // 子进程退出
            }
        }

        // 6. 父进程逻辑：将子进程添加到作业列表
        addjob(jobs, pid, state, cmdline);  // jobs为全局作业管理结构，记录所有进程状态

        // 解除信号阻塞：允许shell重新处理SIGINT、SIGTSTP、SIGCHLD信号
        if(sigprocmask(SIG_UNBLOCK, &set, NULL) < 0)
            unix_error("sigprocmask error");

        // 7. 根据进程状态（前后台）进行不同处理
        if(state == FG)
            waitfg(pid);  // 前台进程：shell阻塞等待其执行完毕
        else
            // 后台进程：直接输出作业信息（作业ID、进程ID、命令行），不等待
            printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);  // pid2jid：将进程ID映射为作业ID
    }
    return;
}
```

注意：

1.  每个子进程必须有自己独一无二的进程组id，通过在fork()之后的子进程中Setpgid(0,0)实现，这样当向前台程序发送ctrl+c或ctrl+z命令时，才不会影响到后台程序。如果没有这一步，则所有的子进程与当前的tsh shell进程为同一个进程组，发送信号时，前后台的子进程均会收到。
1.  在fork()新进程前要阻塞SIGCHLD信号，防止出现竞争，这是经典的同步错误，如果不阻塞会出现子进程先结束从jobs中删除，然后再执行到主进程addjob的竞争问题。

**builtin\_cmd函数 **

函数功能：识别并执行内置命令: quit, fg, bg, 和 jobs。

函数原型：`int builtin_cmd(char **argv)`，参数为argv 参数列表

实现思路：

1.  当命令行参数为quit时，直接终止shell
1.  当命令行参数为jobs时，调用listjobs函数，显示job列表
1.  当命令行参数为bg或fg时，调用do\_bgfg函数，执行内置的bg和fg命令
1.  不是内置命令时返回0

完整代码：

```java
/**
 * 判断命令是否为shell内置命令，并执行相应操作
 * @param argv 解析后的命令及参数数组（argv[0]为命令名）
 * @return 1 表示是内置命令并已处理；0 表示非内置命令
 */
int builtin_cmd(char **argv)
{
    // 1. 处理"quit"命令：退出shell
    if(!strcmp(argv[0], "quit"))  // 比较命令名是否为"quit"（strcmp返回0表示相等，!0为真）
        exit(0);                  // 退出当前shell进程（终止程序）

    // 2. 处理"bg"（后台运行）和"fg"（前台运行）命令
    else if(!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg"))  // 命令为"bg"或"fg"
        do_bgfg(argv);  // 调用专门函数处理作业的前后台切换（如将暂停的作业继续运行）

    // 3. 处理"jobs"命令：列出所有后台作业状态
    else if(!strcmp(argv[0], "jobs"))  // 命令为"jobs"
        listjobs(jobs);  // 遍历作业列表（jobs），打印所有作业的ID、状态、命令行等信息

    // 4. 非内置命令：返回0，由调用者（eval）创建子进程执行外部命令
    else
        return 0;     /* not a builtin command */

    // 若为内置命令，处理完成后返回1
    return 1;
}
```

**do\_bgfg函数 **

函数功能：实现内置命令bg 和 fg

首先要明确的是bg和bg的作用

> `bg <job>`:将停止的后台作业更改为正在运行的后台作业。通过发送SIGCONT信号重新启动`<job>`，然后在后台运行它。`<job>`参数可以是PID，也可以是JID。ST -> BG
>
> `fg <job>`:将已停止或正在运行的后台作业更改为前台正在运行的作业。通过发送SIGCONT信号重新启`<job>`，然后在前台运行它。`<job>`参数可以是PID，也可以是JID。ST -> FG，BG -> FG

函数原型：`void do_bgfg(char **argv)`，参数为argv 参数列表

实现思路：

1.  判断argv\[\]是否带%，若为整数则传入pid，若带%则传入jid。接着调用getjobjid函数来获得对应的job结构体，如果返回为空，说明列表中并不存在jid的job，要输出提示。
1.  使用strcmp函数判断是bg命令还是fg命令
1.  若是bg，使目标进程重新开始工作，设置状态为BG(后台)，打印进程信息
1.  若是fg，使目标进程重新开始工作，设置状态为FG(前台)，等待进程结束

完整代码：

```java
/**
 * 处理bg和fg命令，实现作业的前后台切换及恢复运行
 * @param argv 解析后的命令及参数（argv[0]为"bg"或"fg"，argv[1]为目标作业的PID或%jobid）
 */
void do_bgfg(char **argv)
{
    int num;                     // 存储转换后的PID或JobID
    struct job_t *job;           // 指向目标作业的指针

    // 1. 检查是否提供了目标作业参数
    if(!argv[1]){
        // 未提供参数时报错（需要PID或%jobid）
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return ;
    }

    // 2. 解析参数：区分JobID（%开头）和PID（纯数字）
    if(argv[1][0] == '%'){
        // 解析JobID（格式为%数字）
        // 将%后面的字符串转为整数，基数为10
        if((num = strtol(&argv[1][1], NULL, 10)) <= 0){
            // 转换失败（非数字或结果≤0），提示参数格式错误
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }
        // 根据JobID从作业列表中查找作业
        if((job = getjobjid(jobs, num)) == NULL){
            printf("%%%d: No such job\n", num);  // 未找到对应作业
            return;
        }
    } else {
        // 解析PID（纯数字格式）
        if((num = strtol(argv[1], NULL, 10)) <= 0){
            // 转换失败，提示参数格式错误
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }
        // 根据PID从作业列表中查找作业
        if((job = getjobpid(jobs, num)) == NULL){
            printf("(%d): No such process\n", num);  // 未找到对应进程
            return;
        }
    }

    // 3. 处理bg命令：将作业放入后台运行
    if(!strcmp(argv[0], "bg")){
        job->state = BG;  // 更新作业状态为后台运行
        // 发送SIGCONT信号到整个进程组（-job->pid表示进程组ID），恢复暂停的作业
        if(kill(-job->pid, SIGCONT) < 0)
            unix_error("kill error");  // 信号发送失败时的错误处理
        // 打印后台作业信息（作业ID、进程ID、命令行）
        printf("[%d] (%d) %s", job->jid, job->pid, job->cmdline);
    }
    // 4. 处理fg命令：将作业放入前台运行
    else if(!strcmp(argv[0], "fg")) {
        job->state = FG;  // 更新作业状态为前台运行
        // 发送SIGCONT信号到进程组，恢复暂停的作业
        if(kill(-job->pid, SIGCONT) < 0)
            unix_error("kill error");
        // 等待前台作业完成（shell阻塞直到该作业结束）
        waitfg(job->pid);
    }
    // 5. 内部错误处理（理论上不会执行）
    else {
        puts("do_bgfg: Internal error");
        exit(0);
    }

    return;
}
```

**waitfg（）函数 **

函数功能：等待一个前台作业结束，或者说是阻塞一个前台的进程直到这个进程变为后台进程

函数原型：`void waitfg(pid_t pid)`，参数为进程ID

实现思路：判断当前的前台的进程组pid是否和当前进程的pid是否相等，如果相等则sleep直到前台进程结束。

完整代码：

```java
void waitfg(pid_t pid)
{
    sigset_t mask;
    sigemptyset(&mask);
    while (fgpid(jobs) != 0)
    {
        sigsuspend(&mask);
    }
    return;
}

```

**sigchld\_handler函数 **

函数功能：处理SIGCHILD信号

函数原型：`void sigchld_handler(int sig)`，参数为信号类型

首先了解一下父进程回收子进程的过程：当一个子进程终止或者停止时，内核会发送一个SIGCHLD信号给父进程。因此父进程必须回收子进程，以避免在系统中留下僵死进程。父进程捕获这个SIGCHLD信号，回收一个子进程。一个进程可以通过调用 waitpid 函数来等待它的子进程终止或者停止。如果回收成功，则返回为子进程的 PID, 如果 WNOHANG, 则返回为 0, 如果其他错误，则为 -1。

实现思路：

 *  用while循环调用waitpid直到它所有的子进程终止。
 *  检查己回收子进程的退出状态

    *  WIFSTOPPED：：引起返回的子进程当前是被停止的
    *  WIFSIGNALED：子进程是因为一个未被捕获的信号终止
    *  WIFEXITED：子进程通过调用exit 或者return正常终止
 *  然后分别用WSTOPSIG，WTERMSIG，WEXITSTATUS提取以上三个退出状态。注意如果引起返回的子进程当前是被停止的进程，那么要将其状态设置为ST

完整代码：

```java
/**
 * SIGCHLD信号处理函数：响应子进程状态变化（终止或暂停）
 * 负责更新作业状态、删除已结束作业，并输出相关信息
 * @param sig 信号值（此处应为SIGCHLD）
 */
void sigchld_handler(int sig)
{
    int status, jid;                // status：子进程状态；jid：作业ID
    pid_t pid;                      // 子进程PID
    struct job_t *job;              // 指向子进程对应的作业结构

    // 调试模式：输出进入信号处理函数的信息
    if(verbose)
        puts("sigchld_handler: entering");

    /*
    以非阻塞方式等待所有子进程状态变化
    waitpid参数说明：
        -1：等待任意子进程
        &status：存储子进程状态信息
        WNOHANG | WUNTRACED：非阻塞，且捕获子进程“终止”和“暂停”状态
    循环处理：确保所有状态已改变的子进程都被处理
    */
    while((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0){

        // 查找子进程对应的作业，若未找到则提示异常
        if((job = getjobpid(jobs, pid)) == NULL){
            printf("Lost track of (%d)\n", pid);  // 无法识别的子进程
            return;
        }

        jid = job->jid;  // 获取作业ID

        // 情况1：子进程被信号暂停（如Ctrl+Z发送的SIGTSTP）
        if(WIFSTOPPED(status)){
            // 输出暂停信息：作业ID、进程ID、暂停信号
            printf("Job [%d] (%d) stopped by signal %d\n", jid, job->pid, WSTOPSIG(status));
            job->state = ST;  // 更新作业状态为“暂停（ST）”
        }
        // 情况2：子进程正常终止（如调用exit或return）
        else if(WIFEXITED(status)){
            // 从作业列表中删除该作业
            if(deletejob(jobs, pid))
                if(verbose){  // 调试模式：输出删除和正常终止信息
                    printf("sigchld_handler: Job [%d] (%d) deleted\n", jid, pid);
                    printf("sigchld_handler: Job [%d] (%d) terminates OK (status %d)\n", 
                           jid, pid, WEXITSTATUS(status));
                }
        }
        // 情况3：子进程被未捕获的信号终止（如SIGINT、SIGKILL）
        else {
            // 从作业列表中删除该作业
            if(deletejob(jobs, pid)){
                if(verbose)
                    printf("sigchld_handler: Job [%d] (%d) deleted\n", jid, pid);
            }
            // 输出终止信息：作业ID、进程ID、终止信号
            printf("Job [%d] (%d) terminated by signal %d\n", jid, pid, WTERMSIG(status));
        }
    }

    // 调试模式：输出退出信号处理函数的信息
    if(verbose)
        puts("sigchld_handler: exiting");

    return;
}
```

**sigint\_handler函数 **

函数功能：捕获SIGINT信号

函数原型：`void sigchld_handler(int sig)`，参数为信号类型

实现思路：

1.  调用函数fgpid返回前台进程pid
1.  如果当前进程pid不为0，那么调用kill函数发送SIGINT信号给前台进程组
1.  在2中调用kill函数如果返回值为-1表示进程不存在。输出error

完整代码：

```java
/**
 * SIGINT信号处理函数：响应Ctrl+C，中断前台作业的执行
 * @param sig 信号值（此处应为SIGINT）
 */
void sigint_handler(int sig)
{
    // 调试模式：输出进入信号处理函数的信息
    if(verbose)
        puts("sigint_handler: entering");

    // 获取当前前台作业的进程ID（PID），无前台作业则返回0
    pid_t pid = fgpid(jobs);

    // 若存在前台作业，向前台进程组发送SIGINT信号
    if(pid){
        /* 
        发送SIGINT给前台进程组内的所有进程：
        - -pid 表示目标是进程组（进程组ID = 前台进程PID）
        - 确保前台作业的主进程及其子进程都收到中断信号
        */
        if(kill(-pid, SIGINT) < 0)
            unix_error("kill (sigint) error");  // 信号发送失败时的错误处理

        // 调试模式：输出前台作业被中断的信息
        if(verbose){
            printf("sigint_handler: Job (%d) killed\n", pid);
        }
    }

    // 调试模式：输出退出信号处理函数的信息
    if(verbose)
        puts("sigint_handler: exiting");

    return;
}
```

**sigtstp\_handler函数 **

函数功能：同sigint\_handler差不多，捕获SIGTSTP信号

函数原型：`void sigtstp_handler(int sig)`，参数为信号类型

首先了解一下SIGTSTP的作用：SIGTSPT信号默认行为是停止直到下一个 SIGCONT，是来自终端的停止信号，在键盘上输入 CTR+Z会导致一个 SIGTSPT信号被发送到外壳。外壳捕获该信号，然后发送SIGTSPT信号到这个前台进程组中的每个进程。在默认情况下，结果是停止或挂起前台作业。

实现思路：

1.  用fgpid(jobs)获取前台进程pid，判断当前是否有前台进程，如果没有直接返回。
1.  用kill(-pid,sig)函数发送SIGTSPT信号给前台进程组。

完整代码：

```java
/**
 * SIGTSTP信号处理函数：响应Ctrl+Z，暂停前台作业的执行
 * @param sig 信号值（此处应为SIGTSTP）
 */
void sigtstp_handler(int sig)
{
    // 调试模式：输出进入信号处理函数的信息
    if(verbose)
        puts("sigstp_handler: entering");

    // 获取当前前台作业的进程ID（PID），无前台作业则返回0
    pid_t pid = fgpid(jobs);
    // 根据PID查找对应的作业结构（用于获取作业ID等信息）
    struct job_t *job = getjobpid(jobs, pid);

    // 若存在前台作业，向前台进程组发送SIGTSTP信号
    if(pid){
        /*
        发送SIGTSTP给前台进程组内的所有进程：
        - -pid 表示目标是进程组（进程组ID = 前台进程PID）
        - 确保前台作业的主进程及其子进程都被暂停
        */
        if(kill(-pid, SIGTSTP) < 0)
            unix_error("kill (tstp) error");  // 信号发送失败时的错误处理

        // 调试模式：输出前台作业被暂停的信息（作业ID、进程ID）
        if(verbose){
            printf("sigstp_handler: Job [%d] (%d) stopped\n", job->jid, pid);
        }
    }

    // 调试模式：输出退出信号处理函数的信息
    if(verbose)
        puts("sigstp_handler: exiting");

    return;
}
```

注意：使用kill函数，如果 pid 小于零才会发送信号sig 给进程组中的每个进程，因此这里使用-pid

至此tsh.c文件完成