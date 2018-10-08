title: 使用fishhook hook NSLog 函数
date: 2017-02-24 15:51:40
tags:
---

使用的是 facebook 的 [fishhook](https://github.com/facebook/fishhook) 进行 C 函数的 hook 的。
<!-- more -->

```
//
//  main.m
//  FishTest
//
//  Created by aaron.zheng on 2017-02-22.
//  Copyright © 2017 aaron.zheng. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "fishhook.h"
#import <dlfcn.h>

static int (*orig_close)(int);
static int (*orig_open)(const char *, int, ...);

static void (*orig_nslog)(NSString *format, ...);


int my_close(int fd) {
    printf("Calling real close(%d)\n", fd);
    return orig_close(fd);
}

int my_open(const char *path, int oflag, ...) {
    va_list ap = {0};
    mode_t mode = 0;
    
    if ((oflag & O_CREAT) != 0) {
        // mode only applies to O_CREAT
        va_start(ap, oflag);
        mode = va_arg(ap, int);
        va_end(ap);
        printf("Calling real open('%s', %d, %d)\n", path, oflag, mode);
        return orig_open(path, oflag, mode);
    } else {
        printf("Calling real open('%s', %d)\n", path, oflag);
        return orig_open(path, oflag, mode);
    }
}


void my_nslog(NSString *format, ...) {
    printf("my_nslog\n");
    
    /*方法一*/
    va_list vl;
    va_start(vl, format);
    NSString* str = [[NSString alloc] initWithFormat:format arguments:vl];
    va_end(vl);
    orig_nslog(str);
    
    /*方法二*/
    va_list va;
    va_start(va, format);
    NSLogv(format, va);
    va_end(va);
    
    
}




int main(int argc, const char * argv[]) {
    @autoreleasepool {
//        rebind_symbols((struct rebinding[2]){{"close", my_close, (void *)&orig_close}, {"open", my_open, (void *)&orig_open}}, 2);
//        
//        // Open our own binary and print out first 4 bytes (which is the same
//        // for all Mach-O binaries on a given architecture)
//        int fd = open(argv[0], O_RDONLY);
//        uint32_t magic_number = 0;
//        read(fd, &magic_number, 4);
//        printf("Mach-O Magic Number: %x \n", magic_number);
//        close(fd);
        
        struct rebinding nslog_rebinding = {"NSLog",my_nslog,(void*)&orig_nslog};
        rebind_symbols((struct rebinding[1]){nslog_rebinding}, 1);
        
        NSLog(@"hello word! %@,%d",@"ss",123);
        
    }
    return 0;
}

```

拓展链接：

- [动态修改 C 语言函数的实现](http://www.jianshu.com/p/625a61dfe039)
- [趣探 Mach-O: FishHook 解析](http://www.open-open.com/lib/view/open1487057519754.html)