title: 一个简单的定时器源Demo
date: 2016-04-23 11:04:44
categories: ios 基础
tags: [ios,定时器]
---


GCD 定时器基于分派队列，而不是像 N STimer 那样基于运行循环，这意味着把它们用在多个线程应用中要容易得多。GCD 定时器用块而不是选择器，所以不需要独立的方法处理定时器，这样也更容易避免重复 GCD 定时器的循环保留。

<!-- more -->

下面的这个 Demo 使用 GCD 实现了一个简单的定时器。

主要做到了一下几点：

Simple GCD-based timer based on NSTimer. It starts immediately and stops when released. This avoids many of the typical problems with NSTimer:

- RNTimer runs in all modes (unlike NSTimer)

- RNTimer runs when there is no runloop (unlike NSTimer)

- Repeating RNTimers can easily avoid retain loops (unlike NSTimer)

** RNTimer.h **

		//  RNTimer
		
		#import <Foundation/Foundation.h>
		
		/** Simple GCD-based timer based on NSTimer.
		
		 Starts immediately and stops when deallocated. This avoids many of the typical problems with NSTimer:
		
		 * RNTimer runs in all modes (unlike NSTimer)
		 * RNTimer runs when there is no runloop (unlike NSTimer)
		 * Repeating RNTimers can easily avoid retain loops (unlike NSTimer)
		*/
		
		@interface RNTimer : NSObject
		
		/**---------------------------------------------------------------------------------------
		 @name Creating a Timer
		 -----------------------------------------------------------------------------------------
		*/
		
		/** Creates and returns a new repeating RNTimer object and starts running it
		
		 After `seconds` seconds have elapsed, the timer fires, executing the block.
		 You will generally need to use a weakSelf pointer to avoid a retain loop.
		 The timer is attached to the main GCD queue.
		
		 @param seconds The number of seconds between firings of the timer. Must be greater than 0.
		 @param block Block to execute. Must be non-nil
		
		 @return A new RNTimer object, configured according to the specified parameters.
		*/
		+ (RNTimer *)repeatingTimerWithTimeInterval:(NSTimeInterval)seconds block:(dispatch_block_t)block;
		
		
		/**---------------------------------------------------------------------------------------
		 @name Firing a Timer
		 -----------------------------------------------------------------------------------------
		*/
		
		/** Causes the block to be executed.
		
		 This does not modify the timer. It will still fire on schedule.
		*/
		- (void)fire;
		
		
		/**---------------------------------------------------------------------------------------
		 @name Stopping a Timer
		 -----------------------------------------------------------------------------------------
		*/
		
		/** Stops the receiver from ever firing again
		
		 Once invalidated, a timer cannot be reused.
		
		*/
		- (void)invalidate;
		@end



** RNTimer.m **

		#import "RNTimer.h"
		
		@interface RNTimer ()
		@property (nonatomic, readwrite, copy) dispatch_block_t block;
		@property (nonatomic, readwrite, assign) dispatch_source_t source;
		@end
		
		@implementation RNTimer
		@synthesize block = _block;
		@synthesize source = _source;
		
		+ (RNTimer *)repeatingTimerWithTimeInterval:(NSTimeInterval)seconds
		                                      block:(void (^)(void))block {
		  NSParameterAssert(seconds);
		  NSParameterAssert(block);
		
		  RNTimer *timer = [[self alloc] init];
		  timer.block = block;
		  timer.source = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER,
		                                        0, 0,
		                                        dispatch_get_main_queue());
		  uint64_t nsec = (uint64_t)(seconds * NSEC_PER_SEC);
		  dispatch_source_set_timer(timer.source,
		                            dispatch_time(DISPATCH_TIME_NOW, nsec),
		                            nsec, 0);
		  dispatch_source_set_event_handler(timer.source, block);
		  dispatch_resume(timer.source);
		  return timer;
		}
		
		- (void)invalidate {
		  if (self.source) {
		    dispatch_source_cancel(self.source);
		    dispatch_release(self.source);
		    self.source = nil;
		  }
		  self.block = nil;
		}
		
		- (void)dealloc {
		  [self invalidate];
		}
		
		- (void)fire {
		  self.block();
		}
		
		
		@end


拓展链接：[使用 GCD 的信号量和队列关联数据 Demo](https://github.com/iosptl/ios7ptl/tree/master/ch23-AdvGCD/ProducerConsumer)