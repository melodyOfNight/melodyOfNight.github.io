title: 关于 NSBundle 路径遇到的一点问题
date: 2015-11-19 21:31:16
categories: ios 基础
tags: 路径
---
### [1] 想要解决的问题
记忆蝰蛇音效最后加载的 *.irs* 或者 *.wav* 路径。

### [2] 解决过程

[2.1] 能够想到的最简单的解决办法就是直接持久化保存即可。可以写到 plist 文件或者使用*NSUserDefaults*保存。最后选择了后者。*.irs* 文件放在工程 *[NSBundle mainBundle]*下的 irs 文件夹里面。
<!-- more -->
保存

	[[NSUserDefaults standardUserDefaults] setObject:path forKey:@"lastViperirPath"];
	[[NSUserDefaults standardUserDefaults] synchronize];

取出


	NSString *lastViperirPath = [[NSUserDefaults standardUserDefaults] objectForKey:@"lastViperirPath"];


很简单的事情，应该可以了，恩，测试一下。
运行，切换音效，强制退出，再次运行，断点看到路径确实保存下来了，很奇怪的是设置进去的路径和再次运行后拿出来的路径是不一样的。
这样子去设置路径的


	NSString *path = [[NSBundle mainBundle] pathForResource:@"极致HiFi" ofType:@"irs"];


debug 看到保存在 NSUserDefaults 的路径为：


	/Users/aaron/Library/Developer/CoreSimulator/Devices/13743822-80BB-451A-821A-81BCC341E1D7/data/Containers/Bundle/Application/9B60A637-70DF-42FB-BA57-9C0393FD7AA5/kugou.app/极致HiFi.irs


再次运行 *[NSBundle mainBundle]* 的路径却不是上面那一串了，而是


	/Users/aaron/Library/Developer/CoreSimulator/Devices/13743822-80BB-451A-821A-81BCC341E1D7/data/Containers/Bundle/Application/90ABB2E4-8E08-4335-8FAA-684581FF58D5/kugou.app/极致HiFi.irs


*.../Bundle/Application/xxxx-xx-xxx* 这一串改变了。结果你拿上次退出的那一串路径设置进去它发现是非法的路径，不存在。

OK,OK 那我不把 *.irs* 放在 mianBundle 下，直接放在沙盒下算了，这总可以了吧，想到以后可能还会不断添加新的 *.irs* 文件进来，又不想每次都去打开沙盒目录，干脆就在应用启动的时候将 mainBundle irs文件夹下的所有 *.irs* 和 *.wav* 文件都 copy 到沙盒的 Document 目录下的 irs 文件夹里，以后有新加的音效文件就直接拉到工程的irs目录下就好了，也比较方便直观。



				+ (void)copyViperirIRSFileToDocument {
				    NSArray *searchPaths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask,YES);
				    NSString *irsPath = [NSString stringWithFormat:@"%@/irs",searchPaths[0]];
				    
				    BOOL isDirectory;
				    NSError *error = nil;
				    NSFileManager *fileManager = [NSFileManager defaultManager];
				    BOOL isFileExist = [fileManager fileExistsAtPath:irsPath isDirectory:&isDirectory];
				    if (isFileExist && isDirectory) {
				        
				    }else {
				        [fileManager createDirectoryAtPath:irsPath withIntermediateDirectories:YES attributes:nil error:&error];
				        if (error) {
				            NSLog(@"create irs directory error:%@",error.description);
				        }
				    }
				    
				    
				    NSArray *arr = [[NSBundle mainBundle] pathsForResourcesOfType:nil inDirectory:DIRECTORY_NAME];
				    for (NSString *originPath in arr) {
				        NSString *destinationPath = [NSString stringWithFormat:@"%@/%@/%@",searchPaths[0],DIRECTORY_NAME,[originPath lastPathComponent]];
				        if (originPath && ![_irsFileManager isFileExistAtPath:destinationPath]) {
				            NSError *error = nil;
				            [[NSFileManager defaultManager] copyItemAtPath:originPath toPath:destinationPath error:&error];
				            if (error) {
				                NSLog(@"++++++ move %@ to irs directory error:%@",[originPath lastPathComponent],error.description);
				            }
				        }
				    }
				}
				
				- (BOOL)isFileExistAtPath:(NSString*)path {
				    BOOL isDirectory = NO;
				    BOOL isFileExist = [[NSFileManager defaultManager] fileExistsAtPath:path isDirectory:&isDirectory];
				    if (isFileExist && !isDirectory) {
				        return YES;
				    }
				    return NO;
				}


其中遇到了一个问题，NSArray *arr = [[NSBundle mainBundle] pathsForResourcesOfType:nil inDirectory:DIRECTORY_NAME]; 返回的数组个数为0，不对啊，正确应该是4个的。stackoverflow 上找了一下，才知道将文件夹 add 到工程的时候要选择 *Create folder references* 而不是 *Create groups* 这样才能被识别出来。选择 *Create folder references* note:添加进来的文件夹是蓝色的而不是黄色的。

再次 debug ,保存进去的路径是这样的：
*/Users/aaron/Library/Developer/CoreSimulator/Devices/13743822-80BB-451A-821A-81BCC341E1D7/data/Containers/Data/Application/[118BAD92-DC85-4486-B47A-4FBB52C56AE10]()/Documents/irs/极致HiFi.irs*
停止运行，然后再次运行，打印出 Document 的路径：

*/Users/aaron/Library/Developer/CoreSimulator/Devices/13743822-80BB-451A-821A-81BCC341E1D7/data/Containers/Data/Application/[00F18198-FC6B-47E9-BFAD-E4B866430E7F](0)/Documents*

尼玛，Application/后面那一串又是不一样的，摔~
```才发现原来每次一重新运行，无论是 mianBundle 还是沙盒路径里的 Application/xxx 后面那一串都是会改变的```。

[2.2] 细想一下，既然这样的话，那我从一开始就直接保存 ```.irs``` 或者 ```.wav``` 的名字不就好了，为什么一定要完整的路径呢？然后再下一次运行的时候再取出 mainBundle 或者是沙盒的路径在拼接 ```.irs``` 的名字不就得到了完整的路径了，擦，绕着绕着有回到使用 ```NSBundle```上来了。
### [3]解决办法
保存

	[[NSUserDefaults standardUserDefaults] setObject:[path lastPathComponent] forKey:@"lastViperirName"];
	            [[NSUserDefaults standardUserDefaults] synchronize];

取出

	NSString *lastViperirName = [[NSUserDefaults standardUserDefaults] objectForKey:@"lastViperirName"];
	        NSString *lastViperirPath = [NSString stringWithFormat:@"%@/%@",[[NSBundle mainBundle] bundlePath],lastViperirName];
	        
需要注意的是：上面把irs文件夹以```Create folder references``` 的形式 add 到了工程中了，现在要改回 ```Create groups``` 形式 add 回去，要不然 NSBundle 是找不到该路径的。

### [4]总结

敲代码之前先好好考虑清楚，有没有更好的办法，不要浪费无谓的时间。当然，做其他事情也是一样。
