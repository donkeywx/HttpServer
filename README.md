# HttpServer
##介绍
本项目为c++11实现的web下载服务器，支持在浏览器页面播放音频和视频，以及文件下载。
##技术要点
* 基于one loop per thread的思想实现主框架，同时使用一种one timer per loop的方式踢掉空闲连接。
* event loop使用epoll LT模式加非阻塞I/O实现；
* timer使用STL的multi_map实现，支持心搏时间动态变化，超时时间动态变化； 
* 使用timerfd将定时事件融入epoll系统调用，即统一事件源；
* 实现高效的多缓冲异步日志系统；
* 实现自定义buffer，支持延迟关闭连接，同时使用mmap加快文件读取速度。
* 使用线程池充分利用多核CPU，并避免线程频繁创建销毁的开销；
## 总体框架

## Timer实现要点
## 日志类实现要点
* 一个包含多个线程的进程最好只写一个文件，这样在分析日志时可以避免在多个文件中跳来跳去。
* 业务线程：干事的线程。  
  日志线程：负责收集日志，并写入日志文件。  
  异步日志：日志线程负责从缓冲区收集日志，并写入日志文件。业务线程除了干事之外只管往缓冲区中写日志。  
* 为什么需要异步日志？  
  因为若是由业务线程直接写入日志文件时，会造成业务线程在进行I/O操作时陷入阻塞状态。这可能造成请求方超时，或者耽误发送心跳消息等。<br>
* 多缓冲技术:前端一块buffer，后端多块buffer，前端buffer满后，和后端一块空闲的buffer交换，然后由日志线程将这块满了的buffer写入日志文件。<br>
* 为什么需要多缓冲？
    * 前端不是将日志一条一条的传给后端，而是将多条日志拼成一个大buffer传给后端，减少了后端被唤醒的频率，降低了开销。<br>
    * 前端写日志时，当buffer满时，不必等待写磁盘操作。<br>
* 为了及时将日志消息写入文件，即便前端buffer没满，每隔一段时间也会进行交换操作。
* 日志打印的消息格式如下：[日期 时间.微秒][日志级别][线程id][源文件名:行号][正文]<br>
  为了加快日志打印速度，其中线程id的打印，和时间戳的的打印需要使用一点小技巧。   
  * 打印线程id前会先查看线程id是否已经缓存过，若缓存过则直接打印，没有缓存过才使用系统调用syscall(SYS_gettid)获取全局唯一的线程id。  
  * 时间戳长这个样子20190511 12:43:05.787868，每次打印时间戳前，会查看当前时刻和上一次打印时间戳的时刻是否处于同一秒内，若处于同一秒内，则只格式化微秒部分；不处于同一秒内才调用gmtime_r格式化时间部分。至于如何判断和上一次打印时间戳是否处于同一秒内，是通过gettimeofday做到的，这个函数可以求得距离1970年0时0分0秒的微妙数。由于gettimeofday不是系统调用，不会陷入内核，所以调用时间相当快。
  
`注：本日志类仅实现高效的多缓冲异步日志系统，不支持日志文件的滚动功能。`  
## HttpServer日志类总体流程
前端提供一块buffer供各业务线程写入，后端预先分配两块空闲buffer，使用STL中的双端链表维护。当前端buffer满时，和后端的一块空闲buffer交换(交换的是指针，所以速度很快)。若后端没有空闲buffer则会像系统申请一块新的buffer加入双端链表中，所以，该双端链表是会动态增长的，会自己动态增长到合适的长度；当后端日志线程发现有满的后端buffer时，就开始将该满的后端buffer写入日志文件。  
## 测试环境
unbuntu 16.04 VMware Workstation 14 Player
## 测试方式
和muduo的日志库进行对比测试，写入50w条日志，每条日志长度100字节，统计总的写入时间和写入速度。  
* mudo
测试代码：  
```cpp  
#include <stdio.h>
#include<iostream>
#include<muduo/base/Logging.h>
#include<ctime>
using namespace std;
using namespace muduo;

FILE* g_file;

void dummyOutput(const char* msg, int len)
{
	if (g_file)
	{
		fwrite(msg, 1, len, g_file);
	}
}

void dummyFlush()
{
	fflush(g_file);
}

int main()
{
	g_file = ::fopen("./test.txt", "ae");
	
	Logger::setOutput(dummyOutput);
	Logger::setFlush(dummyFlush);
	cout << "len:"<< strlen("20190510 12:09:47.579382Z 100056 INFO  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx - main.cpp:143")<<endl;
	clock_t start, end;
	start = clock();
	for (int i = 0; i < 500000; i++)
	{
		LOG_INFO << "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
	}
	end = clock();
	double endTime = (double)(end - start) / CLOCKS_PER_SEC;
	double totaltime = endTime * 1000;
	double mBytes = 500000 * 100 / 1024 / 1024;
	double mBytesEachSecond = (mBytes / totaltime) * 1000;
	cout <<"muduo  -"<<"Total time:" << totaltime << "ms" << "|" << "Bytes:" << mBytes << "MB" << "|" << "Rate:" << mBytesEachSecond << "MB/s" << endl;
	::fclose(g_file);
	return 0;
}
```
* HttpServer 
测试代码：  
```cpp
#include<iostream>
#include"base/Logger.h"
#include<ctime>
int main()
{
	Logger::getLogger()->start(true);

	clock_t start, end;
	start = clock();
	cout << "len:" << strlen("[20190511 12:43:05.787868][INFO][108616][/home/hanliu/projects/HttpSever/main.cpp:87][xxxxxxxxxxxxx]") << endl;
	for (int i = 0; i < 500000; i++)
	{
		LogInfo("xxxxxxxxxxxxx");
	}
	end = clock();
	double endTime	 = (double)(end - start) / CLOCKS_PER_SEC;
	double totaltime = endTime * 1000;
	double mBytes    = 500000 * 100 / 1024 / 1024;
	double mBytesEachSecond = (mBytes / totaltime) * 1000;
	cout << "mine -"<< "Total time:" << totaltime << "ms" << "|" << "Bytes:" << mBytes << "MB" << "|" << "Rate:" << mBytesEachSecond << "MB/s" << endl;

	Logger::stop();
	return 0;
}
```  
## 测试结果
muduo:   
![](https://github.com/hanAndHan/HttpServer/blob/master/imge/muduoLogTest.png)  
HttpServer:   
![](https://github.com/hanAndHan/HttpServer/blob/master/imge/httpServerLogTest.png)  由上述结果可见HttpServer中的日志库和muduo中的日志库性能差距不大。
## Buffer


