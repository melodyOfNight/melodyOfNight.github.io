title: hook之函数栈帧追溯、NSLog捕获、STDOUT重定向
date: 2016-06-22 18:04:23
categories: ios 进阶
tags: 捕获日志
---
栈帧的追溯需要用到函数：

	int backtrace(void**,int);
	char** backtrace_symbols(void* const*,int);

用法和 linux 上一样，参考链接：[这里](http://man7.org/linux/man-pages/man3/backtrace.3.html)

<!-- more -->

例如可以把追溯功能放到我的 hook 程序里：

	FILE *my_fopen(const char *path, const char *mode)
	{
	FILE *fp;
	void *buffer[100];
	char **strings;
	int nptrs, j;
	
	fp = org_fopen(path, mode);
	log_progress(“fopen(%s, %s)=%08x\n”, path, mode, fp);
	nptrs = backtrace(buffer, 100);
	strings = (char **)backtrace_symbols(buffer, nptrs);
	if (strings == NULL)
	return fp;
	for (j = 0; j < nptrs; j++)
	log_progress(“%s\n”, strings[j]);
	free(strings);
	return fp;
	}
在动态库初始化的时候挂接 hook 点：

	MSHookFunction(fopen, my_fopen, (void **)&org_fopen);
	
看到的效果如下：

	fopen(/var/mobile/Applications/39809BB6-96CE-4DE5-8701-44B3AB7738D2/tdbeta.app/Data/unity default resources, rb)=012e224c
	0   testlib.dylib                       0x011fa95c my_fopen + 100
	1   tdbeta                              0x00afe5b8 tdbeta + 11523512
	2   tdbeta                              0x00afe8d4 tdbeta + 11524308
	3   tdbeta                              0x00ae9a48 tdbeta + 11438664
	4   tdbeta                              0x00ae9ad8 tdbeta + 11438808
	5   tdbeta                              0x00aef7a4 tdbeta + 11462564
	6   tdbeta                              0x00aeaeb8 tdbeta + 11443896
	7   tdbeta                              0x00aec688 tdbeta + 11449992
	8   tdbeta                              0x009a5538 tdbeta + 10110264
	9   tdbeta                              0x00abb7f0 tdbeta + 11249648
	10  tdbeta                              0x009c6c74 tdbeta + 10247284
	11  tdbeta                              0x00ac0710 tdbeta + 11269904
	12  tdbeta                              0x00ab242c tdbeta + 11211820
	13  tdbeta                              0x008f0190 tdbeta + 9367952
	14  tdbeta                              0x000047dc tdbeta + 14300
	15  Foundation                          0×32591277 <redacted> + 450
	16  CoreFoundation                      0x31c585df <redacted> + 14
	17  CoreFoundation                      0x31c58291 <redacted> + 272
	18  CoreFoundation                      0x31c56f01 <redacted> + 1232
	19  CoreFoundation                      0x31bc9ebd CFRunLoopRunSpecific + 356
	20  CoreFoundation                      0x31bc9d49 CFRunLoopRunInMode + 104
	21  GraphicsServices                    0x357a02eb GSEventRunModal + 74
	22  UIKit                               0x33adf301 UIApplicationMain + 1120
	23  spad.dylib                          0x0138b807 _Z21new_UIApplicationMainiPPcP8NSStringS2_ + 238
	24  tdbeta                              0×00005014 tdbeta + 16404
	25  tdbeta                              0x00003bf8 tdbeta + 11256

那么，如何记录 NSLog 函数的输出？

首先看 NSLog 函数的调用关系：

	NSlog->_CFLogvEx->0x32262f20
	
_CFLogvEx 调用：

	CFStringCreateWithFormatAndArgumentsAux
	CFStringGetLength
	CFStringGetMaximumSizeForEncoding
	
进行字符串格式化，然后 malloc 一块内存通过 CFStringGetCString 写入函数 0x32262f20 

调用：CFAbsoluteTimeGetCurrent 获得系统时间 CFGetProgname 获得 mach-o 文件名，

getpid 获得进程 pid

然后重点来了：

	0x32263250 <<redacted>+816>:    movs    r0, #2
	0x32263252 <<redacted>+818>:    blx     0x322b541c <dyld_stub_writev>
	
没错，写入文件句柄2，也就是 stderr

writev 执行完之后，可以在 stderr 上看到格式化好的字符串，例如：

	2013-12-30 23:57:29.292 tdbeta[835:907] -> registered mono modules 0xf9b580

<b>结论</b>：要获得 NSLog 输出的信息只需要重定向 stderr （~~结论的描述好像不是很准确哦，重定向 stderr 只是将错误信息重定向而已，而 NSLog 也包含其他输出，这时候就要用将标准输出重定向到文件了，用这个函数~~

    dup2(fd,1)// fd 是文件句柄，1 是 STDOUT_FILENO
    
换句话说，要 hook NSLog 只要将标准输出和错误输出重定向到文件即可。

如何做重定向？

重定向只需要创建一个文件，然后将其句柄通过函数 dup2 传入，替换掉默认的句柄就好了。

将下面代码放到动态库 init 函数里，然后去看这个 log 文件吧。

	int substrateInit(void)
	{
	  int fd;
	  char pathBuffer[BUFSIZE];
	  fflush(stdout);
	  fflush(stderr);
	  snprintf(pathBuffer, sizeof(pathBuffer), "%s/Library/stdout-%li.log", getenv("HOME"), time(NULL));
	  setvbuf(stdout,NULL,_IONBF,0);
	  fd = open(pathBuffer, (O_RDWR | O_CREAT), 0644);
	  dup2(fd, STDOUT_FILENO);
	  dup2(fd, STDERR_FILENO);
	  printf("Here is dll init, redirect stdout and stderr to logfile\n");
	  return 0;
	} 
	
假如现在要切换为原来的终端显示的话，又要怎么做呢？看一下[解惑 dup/dup2](http://blog.donews.com/mutecat/archive/2007/09/20/1212178.aspx)或许就可以找到答案了。

文章转自[看雪论坛](http://bbs.pediy.com/showthread.php?t=183190) <b>jerryxjtu</b> 大神写的帖子。

参考链接：

[Linux 管道编程技术：dup 函数，dup2 函数，open 函数详解](http://blog.csdn.net/zhouhong1026/article/details/8151235)


[linux shell 中“2>&1”含义](http://www.cnblogs.com/caolisong/archive/2007/04/25/726896.html)