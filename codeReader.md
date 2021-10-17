- [6.S081源码解读](#6s081源码解读)
    - [sh](#sh)
      - [主函数](#主函数)
      - [int getcmd(char *buf, int nbuf)](#int-getcmdchar-buf-int-nbuf)
      - [peek函数](#peek函数)
# 6.S081源码解读
10:10 
user/sh.c
kernel/console.c
kernel/sysproc.c 

### sh
#### 主函数
    int
    main(void)
    {
    static char buf[100];
    int fd;

    // Ensure that three file descriptors are open.
    while((fd = open("console", O_RDWR)) >= 0){
        if(fd >= 3){
        close(fd);
        break;
        }
    }

    // Read and run input commands.
    while(getcmd(buf, sizeof(buf)) >= 0){
        if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
        // Chdir must be called by the parent, not the child.
        buf[strlen(buf)-1] = 0;  // chop \n
        if(chdir(buf+3) < 0)
            fprintf(2, "cannot cd %s\n", buf+3);
        continue;
        }
        if(fork1() == 0)
        runcmd(parsecmd(buf));
        wait(0);
    }
    exit(0);
    }
- 首先打开控制台，确保stdin,stdout,stderr全部打开
- getcmd()获取用户输入指令，如果是cd内置指令，则在shell进程执行，如果是其他用户指令，则打开一个子进程运行，然后shell等待。

#### int getcmd(char *buf, int nbuf)
这个函数用来获取用户输入指令

    int
    getcmd(char *buf, int nbuf)
    {
    fprintf(2, "$ ");
    memset(buf, 0, nbuf);
    gets(buf, nbuf);
    if(buf[0] == 0) // EOF
        return -1;
    return 0;
    }
- gets函数读取nbuf个字符到buf中，直到遇到EOF或者\n|\r

- stderr vs stdout
  stdout是fully buffered，而stderr要么line buffered ， 要么没有缓冲区。stdout和stderr区别开来，stdout的输出可以redirect 也可以用管道，而stderr中的内容直接打印出来。即时处理。
  所以似乎stdout装的是程序的输出，而stderr可以装everything!
  因此这里的getcmd使用fprintf(2, "$ ");


#### peek函数
    int
    peek(char **ps, char *es, char *toks)
    {
    char *s;

    s = *ps;
    while(s < es && strchr(whitespace, *s))
        s++;
    *ps = s;
    return *s && strchr(toks, *s);
    }
peek函数把ps移动到以非空格开始的第一个字符位置上
