---
title:[PG]--从源码理解数据库的启动过程
date:2020-09-27
---



从源码理解数据库的启动过程，该文件位于PG源代码的src\backend\main\main.c中。

### 源代码Main（）函数

从注释中Any Postgres server process begins execution here可以看出，任何的服务端进程都是由此处开始执行的。如下简略源代码中，我将省略对windows系统部分的分析方法抓住代码主干分析。

```c
/*
 * Any Postgres server process begins execution here.
 */
int
main(int argc, char *argv[])
{
    bool        do_check_root = true;

    ...此处有省略...

    progname = get_progname(argv[0]);

    startup_hacks(progname);

    argv = save_ps_display_args(argc, argv);

    MemoryContextInit();

    set_pglocale_pgservice(argv[0], PG_TEXTDOMAIN("postgres"));

    ...此处有省略...

    init_locale("LC_TIME", LC_TIME, "C");

    unsetenv("LC_ALL");

    check_strxfrm_bug();

    if (argc > 1)
    {
        if (strcmp(argv[1], "--help") == 0 || strcmp(argv[1], "-?") == 0)
        {
            help(progname);
            exit(0);
        }
        if (strcmp(argv[1], "--version") == 0 || strcmp(argv[1], "-V") == 0)
        {
            fputs(PG_BACKEND_VERSIONSTR, stdout);
            exit(0);
        }

        if (strcmp(argv[1], "--describe-config") == 0)
            do_check_root = false;
        else if (argc > 2 && strcmp(argv[1], "-C") == 0)
            do_check_root = false;
    }


    if (do_check_root)
        check_root(progname);


#ifdef EXEC_BACKEND
    if (argc > 1 && strncmp(argv[1], "--fork", 6) == 0)
        SubPostmasterMain(argc, argv);    /* does not return */
#endif

...此处有省略...

    if (argc > 1 && strcmp(argv[1], "--boot") == 0)
        ...此处有省略...
    else if (argc > 1 && strcmp(argv[1], "--describe-config") == 0)
        ...此处有省略...
    else if (argc > 1 && strcmp(argv[1], "--single") == 0)
        ...此处有省略...
    else
        PostmasterMain(argc, argv); /* does not return */
    abort();                    /* should not get here */
}
```



### 函数get_progname()

首先，在main()函数内部定义了变量do_check_root，在后面进入到识别参数之前调用check_root函数检查当前进程是否在非root用户下启动。随后，将get_progname(argv[0]);函数执行结果赋值给progname。get_progname函数的作用是获取启动文件的文件名，如下代码所示。

```shell
[postgres@localhost ~]$ ps -ef |grep postgres
...此处有省略...
postgres  57977      1  0 11:01 pts/1    00:00:00 /usr/local/psql-9.5.3/bin/postgres -D db2
...此处有省略...
```



### 函数startup_hacks()

随后进入startup_hacks函数，该函数定义在main.c文件的后半部分定义。初始化的哑参内存pg_memory_barrier()执行结果将通过&dummy_spinlock指针，在这个部分加锁。

```c
static void
startup_hacks(const char *progname)
{

#ifdef WIN32
    {
    ...此处有省略...
#if defined(_M_AMD64) && _MSC_VER == 1800
        ...此处有省略...
#endif                            /* defined(_M_AMD64) && _MSC_VER == 1800 */

    }
#endif                            /* WIN32 */
    ...此处有省略...
    SpinLockInit(&dummy_spinlock);
}
```



### 函数save_ps_display_args()

将argv和环境变量复制一份，并将原本初始化的argv和environ指针进行覆盖。



### 函数MemoryContextInit()

调用MemoryContext初始化函数，初始化内存上下文。



### 函数set_pglocale_pgservice()

设置初始化的文件目录。



### 函数init_locale()

大多属于处理平台的系统字符集.



### 参数判断

本部分较为重要，内容大致为通过判断启动的输入参数调用对应的
函数方法。

如果输入的第一个参数是--boot，调用AuxiliaryProcessMain（）复制进程函数；
如果输入的第一个参数是--describe-config，调用GucInfoMain();
如果输入的第一个参数是--single，调用PostgresMain（）;

```c
if (argc > 1 && strcmp(argv[1], "--boot") == 0)
        AuxiliaryProcessMain(argc, argv);    /* does not return */
    else if (argc > 1 && strcmp(argv[1], "--describe-config") == 0)
        GucInfoMain();            /* does not return */
    else if (argc > 1 && strcmp(argv[1], "--single") == 0)
        PostgresMain(argc, argv,
                     NULL,        /* no dbname */
                     strdup(get_user_name_or_exit(progname)));    /* does not return */
```



### 函数PostmasterMain()

最后，进入PostmasterMain()函数。