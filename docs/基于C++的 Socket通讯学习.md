# Socket通讯、管道、命令执行 [C++]

## 前言

​	数据传输是病毒木马的必备技术之一，而数据回传也成为了病毒木马的一个重要特征，我们就尝试自己写一个程序来实现数据的传输，本文尝试通过c++来进行套接字(socket)的实现。

​	Socket又称套接字，应用程序通常通过套接字向网络发出请求或者应答网络请求。Socket的本质还是API，是对TCP/IP的封装

## 基础知识

### socket缓冲区

​	每个 socket 被创建后，都会分配两个缓冲区，输入缓冲区和输出缓冲区。

​	write()/send() 并不立即向网络中传输数据，而是先将数据写入缓冲区中，再由TCP协议将数据从缓冲区发送到目标机器。一旦将数据写入到缓冲区，函数就可以成功返回，不管它们有没有到达目标机器，也不管它们何时被发送到网络，这些都是TCP协议负责的事情。

​	TCP协议独立于 write()/send() 函数，数据有可能刚被写入缓冲区就发送到网络，也可能在缓冲区中不断积压，多次写入的数据被一次性发送到网络，这取决于当时的网络情况、当前线程是否空闲等诸多因素，不由程序员控制。

read()/recv() 函数也是如此，也从输入缓冲区中读取数据，而不是直接从网络中读取，如下图所示：

​	![image-20211108091935665](https://imagesolo.oss-cn-beijing.aliyuncs.com/image-20211108091935665.png)

这些I/O缓冲区特性如下：

- I/O缓冲区在每个TCP套接字中单独存在；
- I/O缓冲区在创建套接字时自动生成；
- 即使关闭套接字也会继续传送输出缓冲区中遗留的数据；
- 关闭套接字将丢失输入缓冲区中的数据。

### 阻塞模式

对于TCP套接字（默认情况下），当使用 write()/send() 发送数据时：

- 1.首先会检查缓冲区，如果缓冲区的可用空间长度小于要发送的数据，那么 write()/send() 会被阻塞（暂停执行），直到缓冲区中的数据被发送到目标机器，腾出足够的空间，才唤醒 write()/send() 函数继续写入数据。
- 2.如果TCP协议正在向网络发送数据，那么输出缓冲区会被锁定，不允许写入，write()/send() 也会被阻塞，直到数据发送完毕缓冲区解锁，write()/send() 才会被唤醒。
- 3.如果要写入的数据大于缓冲区的最大长度，那么将分批写入。
- 4.直到所有数据被写入缓冲区 write()/send() 才能返回。

当使用 read()/recv() 读取数据时：

- 1.首先会检查缓冲区，如果缓冲区中有数据，那么就读取，否则函数会被阻塞，直到网络上有数据到来。
- 2.如果要读取的数据长度小于缓冲区中的数据长度，那么就不能一次性将缓冲区中的所有数据读出，剩余数据将不断积压，直到有 read()/recv() 函数再次读取。
- 3.直到读取到数据后 read()/recv() 函数才会返回，否则就一直被阻塞。

这就是TCP套接字的阻塞模式。所谓阻塞，就是上一步动作没有完成，下一步动作将暂停，直到上一步动作完成后才能继续，以保持同步性。

### TCP的粘包问题

​	上面提到了socket缓冲区和数据的传递过程，可以看到数据的接收和发送是无关的，read()/recv() 函数不管数据发送了多少次，都会尽可能多的接收数据。也就是说，read()/recv() 和 write()/send() 的执行次数可能不同。

​	例如，write()/send() 重复执行三次，每次都发送字符串"abc"，那么目标机器上的 read()/recv() 可能分三次接收，每次都接收"abc"；也可能分两次接收，第一次接收"abcab"，第二次接收"cabc"；也可能一次就接收到字符串"abcabcabc"。

​	假设我们希望客户端每次发送一位学生的学号，让服务器端返回该学生的姓名、住址、成绩等信息，这时候可能就会出现问题，服务器端不能区分学生的学号。例如第一次发送 1，第二次发送 3，服务器可能当成 13 来处理，返回的信息显然是错误的。

​	这就是数据的“粘包”问题，客户端发送的多个数据包被当做一个数据包接收。也称数据的无边界性，read()/recv() 函数不知道数据包的开始或结束标志（实际上也没有任何开始或结束标志），只把它们当做连续的数据流来处理。

​	在实际状况来说，客户端连续三次向服务器端发送数据，但是服务器端却一次性接收到了所有数据，这就是TCP的粘包问题。

### TCP传输详解

​	TCP（Transmission Control Protocol，传输控制协议）是一种面向连接的、可靠的、基于字节流的通信协议，数据在传输前要建立连接，传输完毕后还要断开连接。

​	客户端在收发数据前要使用 connect() 函数和服务器建立连接。建立连接的目的是保证IP地址、端口、物理链路等正确无误，为数据的传输开辟通道。

TCP建立连接时要传输三个数据包，俗称三次握手（Three-way Handshaking）。

​	来看一下TCP数据包的结构:

![image-20211108092533432](https://imagesolo.oss-cn-beijing.aliyuncs.com/image-20211108092533432.png)

带阴影的几个字段需要重点说明一下：

1.序号：Seq（Sequence Number）序号占32位，用来标识从计算机A发送到计算机B的数据包的序号，计算机发送数据时对此进行标记。

2.确认号：Ack（Acknowledge Number）确认号占32位，客户端和服务器端都可以发送，Ack = Seq + 1。

3.标志位：每个标志位占用1Bit，共有6个，分别为 URG、ACK、PSH、RST、SYN、FIN，具体含义如下：

- URG：紧急指针（urgent pointer）有效。
- ACK：确认序号有效。
- PSH：接收方应该尽快将这个报文交给应用层。
- RST：重置连接。
- SYN：建立一个新连接。
- FIN：断开一个连接。

使用 connect() 建立连接时，客户端和服务器端会相互发送三个数据包

![image-20211108093638877](https://imagesolo.oss-cn-beijing.aliyuncs.com/image-20211108093638877.png)

​	客户端调用 socket() 函数创建套接字后，因为没有建立连接，所以套接字处于`CLOSED`状态；服务器端调用 listen() 函数后，套接字进入`LISTEN`状态，开始监听客户端请求。

​	这个时候，客户端开始发起请求：

​	1.当客户端调用 connect() 函数后，TCP协议会组建一个数据包，并设置 SYN 标志位，表示该数据包是用来建立同步连接的。同时生成一个随机数字 1000，填充“序号（Seq）”字段，表示该数据包的序号。完成这些工作，开始向服务器端发送数据包，客户端就进入了`SYN-SEND`状态。

​	2.服务器端收到数据包，检测到已经设置了 SYN 标志位，就知道这是客户端发来的建立连接的“请求包”。服务器端也会组建一个数据包，并设置 SYN 和 ACK 标志位，SYN 表示该数据包用来建立连接，ACK 用来确认收到了刚才客户端发送的数据包。

​	服务器生成一个随机数 2000，填充“序号（Seq）”字段。2000 和客户端数据包没有关系。

​	服务器将客户端数据包序号（1000）加1，得到1001，并用这个数字填充“确认号（Ack）”字段。

​	服务器将数据包发出，进入`SYN-RECV`状态。

​	1.客户端收到数据包，检测到已经设置了 SYN 和 ACK 标志位，就知道这是服务器发来的“确认包”。客户端会检测“确认号（Ack）”字段，看它的值是否为 1000+1，如果是就说明连接建立成功。

​	接下来，客户端会继续组建数据包，并设置 ACK 标志位，表示客户端正确接收了服务器发来的“确认包”。同时，将刚才服务器发来的数据包序号（2000）加1，得到 2001，并用这个数字来填充“确认号（Ack）”字段。

​	客户端将数据包发出，进入`ESTABLISED`状态，表示连接已经成功建立。

​	1.服务器端收到数据包，检测到已经设置了 ACK 标志位，就知道这是客户端发来的“确认包”。服务器会检测“确认号（Ack）”字段，看它的值是否为 2000+1，如果是就说明连接建立成功，服务器进入`ESTABLISED`状态。

​	至此，客户端和服务器都进入了`ESTABLISED`状态，连接建立成功，接下来就可以收发数据了

​	三次握手的关键是要确认对方收到了自己的数据包，这个目标就是通过“确认号（Ack）”字段实现的。计算机会记录下自己发送的数据包序号 Seq，待收到对方的数据包后，检测“确认号（Ack）”字段，看`Ack = Seq + 1`是否成立，如果成立说明对方正确收到了自己的数据包。

## 实现原理

​	我们知道数据传输肯定是有一个发送端和一个接收端的，这里我们可以称之为服务器端和客户端，这两个都需要初始化`Winsock`服务环境

​	这里简单说一下`Winsock`

​	Winsock是windows系统下利用Socket套接字进行网络编程的相关函数，是Windows下的网络编程接口。

​	Winsock在常见的Windows平台上有两个主要的版本，即Winsock1和Winsock2。编写与Winsock1兼容的程序你需要引用头文件WINSOCK.H，如果编写使用Winsock2的程序，则需要引用WINSOCK2.H。此外还有一个MSWSOCK.H头文件，它是专门用来支持在Windows平台上高性能网络程序扩展功能的。使用WINSOCK.H头文件时，同时需要库文件WSOCK32.LIB，使用WINSOCK2.H时，则需要WS2_32.LIB，如果使用MSWSOCK.H中的扩展API，则需要MSWSOCK.LIB。正确引用了头文件，并链接了对应的库文件，你就构建起编写WINSOCK网络程序的环境了。

​	服务端在初始化`Winsock`环境过后，便调用`Socket`函数创建流式套接字，然后对`sockaddr_in`结构体进行设置，设置服务器绑定的IP地址和端口等信息并调用`bind`函数来绑定。绑定成功后，就可以调用`listen`函数设置连接数量，并进行监听。直到有来自客户端的连接请求，服务器便调用`accept`函数接受连接请求，建立连接，与此同时，便可以使用`recv`函数和`send`函数与客户端进行数据收发

​	客户端初始化环境后，便调用`Socket`函数同样创建流式套接字，然后对`sockaddr_in`结构体进行设置，这里与服务器端不同，它不需要用`bind`绑定，也不需要`listen`监听，他直接使用`connect`等待服务器端发送是数据，建立连接过后，也是使用`recv`和`send`函数来进行数据接收

## 实现过程

项目结构如下：

![image-20211108100749202](https://imagesolo.oss-cn-beijing.aliyuncs.com/image-20211108100749202.png)

给出核心代码：

Server端：

```C++
#include "stdafx.h"
#include "TcpServer.h"
#include <stdio.h>

// 服务端套接字
SOCKET g_ServerSocket;
// 客户端套接字
SOCKET g_ClientSocket;


// 绑定端口并监听
BOOL SocketBindAndListen(char *lpszIp, int iPort)
{
	// 初始化 Winsock 库
	WSADATA wsaData = {0};
	::WSAStartup(MAKEWORD(2, 2), &wsaData);

	// 创建流式套接字
	g_ServerSocket = ::socket(AF_INET, SOCK_STREAM, 0);
	if (INVALID_SOCKET == g_ServerSocket)
	{
		return FALSE;
	}
	// 设置服务端地址和端口信息
	sockaddr_in addr;
	addr.sin_family = AF_INET;
	addr.sin_port = ::htons(iPort);
	addr.sin_addr.S_un.S_addr = ::inet_addr(lpszIp);
	// 绑定IP和端口
	if (0 != ::bind(g_ServerSocket, (sockaddr *)(&addr), sizeof(addr)))
	{
		return FALSE;
	}
	// 设置监听
	if (0 != ::listen(g_ServerSocket, 1))
	{
		return FALSE;
	}

	// 创建接收数据多线程
	::CreateThread(NULL, NULL, (LPTHREAD_START_ROUTINE)RecvThreadProc, NULL, NULL, NULL);

	return TRUE;
}


// 发送数据
void SendMsg(char *pszSend)
{
	// 发送数据
	::send(g_ClientSocket, pszSend, (1 + ::lstrlen(pszSend)), 0);
	printf("[send]%s\n", pszSend);
}


// 接受连接请求 并 接收数据
void AcceptRecvMsg()
{
	sockaddr_in addr = { 0 };
	// 注意：该变量既是输入也是输出
	int iLen = sizeof(addr);   
	// 接受来自客户端的连接请求
	g_ClientSocket = ::accept(g_ServerSocket, (sockaddr *)(&addr), &iLen);
	printf("accept a connection from client!\n");

	char szBuf[MAX_PATH] = { 0 };
	while (TRUE)
	{
		// 接收数据
		int iRet = ::recv(g_ClientSocket, szBuf, MAX_PATH, 0);
		if (0 >= iRet)
		{
			continue;
		}
		printf("[recv]%s\n", szBuf);
	}
}


// 接收数据多线程
UINT RecvThreadProc(LPVOID lpVoid) 
{
	// 接受连接请求 并 接收数据
	AcceptRecvMsg();
	return 0;
}

```

Client:

```C++
#include "stdafx.h"
#include "TcpClient.h"


// 客户端套接字
SOCKET g_ClientSocket;


// 连接到服务器
BOOL Connection(char *lpszServerIp, int iServerPort)
{
	// 初始化 Winsock 库
	WSADATA wsaData = { 0 };
	::WSAStartup(MAKEWORD(2, 2), &wsaData);
	// 创建流式套接字
	g_ClientSocket = ::socket(AF_INET, SOCK_STREAM, 0);
	if (INVALID_SOCKET == g_ClientSocket)
	{
		return FALSE;
	}
	// 设置服务端地址和端口信息
	sockaddr_in addr = { 0 };
	addr.sin_family = AF_INET;
	addr.sin_port = ::htons(iServerPort);
	addr.sin_addr.S_un.S_addr = ::inet_addr(lpszServerIp);
	// 连接到服务器
	if (0 != ::connect(g_ClientSocket, (sockaddr *)(&addr), sizeof(addr)))
	{
		return FALSE;
	}
	// 创建接收数据多线程
	::CreateThread(NULL, NULL, (LPTHREAD_START_ROUTINE)RecvThreadProc, NULL, NULL, NULL);

	return TRUE;
}


// 发送数据
void SendMsg(char *pszSend)
{
	// 发送数据
	::send(g_ClientSocket, pszSend, (1 + ::lstrlen(pszSend)), 0);
	printf("[send]%s\n", pszSend);
}


// 接收数据
void RecvMsg()
{
	char szBuf[MAX_PATH] = { 0 };
	while (TRUE)
	{
		// 接收数据
		int iRet = ::recv(g_ClientSocket, szBuf, MAX_PATH, 0);
		if (0 >= iRet)
		{
			continue;
		}
		printf("[recv]%s\n", szBuf);
	}
}


// 接收数据多线程
UINT RecvThreadProc(LPVOID lpVoid) 
{
	// 接受连接请求 并 接收数据
	RecvMsg();
	return 0;
}

```

实现效果如下：

![image-20211108101418590](https://imagesolo.oss-cn-beijing.aliyuncs.com/image-20211108101418590.png)

### 命令执行回显

​	在Windows系统中，很多WIN32 API函数可以执行CMD命令，例如system、WinExec、CreateProcess等，但是 这些函数均不能获取执行后的结果，所有这里通过管道的方式实现：

**函数介绍**

CreatePipe：创建一个匿名管道，并返回管道读写端的句柄。

```c++
BOOL CreatePipe(
  [out]          PHANDLE               hReadPipe,
  [out]          PHANDLE               hWritePipe,
  [in, optional] LPSECURITY_ATTRIBUTES lpPipeAttributes,
  [in]           DWORD                 nSize
);
```

**参数**

```
[out] hReadPipe
```

指向接收管道读取句柄的变量的指针。

```
[out] hWritePipe
```

指向接收管道写句柄的变量的指针。

```
[in, optional] lpPipeAttributes
```

指向[SECURITY_ATTRIBUTES](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/aa379560(v=vs.85))结构的指针，该 结构确定返回的句柄是否可以被子进程继承。如果*lpPipeAttributes*为**NULL**，则不能继承句柄。

该结构的**lpSecurityDescriptor**成员为新管道指定了一个安全描述符。如果*lpPipeAttributes*为**NULL**，则管道获取默认安全描述符。管道的默认安全描述符中的 ACL 来自创建者的主要令牌或模拟令牌。

```
[in] nSize
```

管道缓冲区的大小，以字节为单位。尺寸只是一个建议；系统使用该值来计算适当的缓冲机制。如果此参数为零，则系统使用默认缓冲区大小。

**实现原理**

​	管道是一种在进程间共享数据的机制，其实质是一段共享内存。Windos系统为这段共享的内存设计使用数据流I/O的方式来访问。一个进程读，一个进程写，这类似于一个管道的两端，因此这种进程间的通信方式成为“管道”。

​	管道分为匿名管道和命名管道。匿名管道只能在父子进程之间通信，不能再网络间通信，而且数据传输是单向的，只能一端写，一端读。命名管道可以在任意进程间通信，通信是双向的，任意一端都可以进行读写，但在同一时间只能由一端读，一端写。

​	创建命名管道的方式可以获取CMD执行结果的输出内容，具体实现流程如下：

​	首先初始化匿名管道的安全属性结构体SECURITY_ATTRIBUTES，使用匿名管道默认的缓冲区大小，并调用函数CreatePipe创建匿名管道，获取管道数据读取句柄和管道数据写入句柄。

​	 对即将创建的进程结构体STARTUPINFO 进行初始化，隐藏进程窗口，并把上面的管道数据写入句柄赋值给新进程控制台的缓存句柄，这样，新进程就会把窗口缓存的输出数据写入到匿名管道中。

​	 调用CreateProcess函数创建新进程。执行CMD命令，并调用WaitForSingleObject 等待命令执行完毕。如果不等待执行完成就获取执行结果的话，在获取结果数据的时候CMD可能为执行完毕，那么获取的就不是完整的结果。

​	命令执行完毕后，边调用ReadFIle函数根据匿名管道的数据读取句柄从匿名管道的缓冲区中读取数据，这个数据就是新进程执行命令返回的结果。

​	经过上面的操作后，CMD的执行结果就获取成功了，接着关闭句柄，释放资源。

附上核心代码：

```c++
BOOL PipeCmd(char* pszCmd, char* pszResultBuffer, DWORD dwResultBufferSize)
{
	HANDLE hReadPipe = NULL;
	HANDLE hWritePipe = NULL;
	SECURITY_ATTRIBUTES securityAttributes = { 0 };
	BOOL bRet = FALSE;
	STARTUPINFO si = { 0 };
	PROCESS_INFORMATION pi = { 0 };

	// 设定管道的安全属性
	securityAttributes.bInheritHandle = TRUE;
	securityAttributes.nLength = sizeof(securityAttributes);
	securityAttributes.lpSecurityDescriptor = NULL;
	// 创建匿名管道
	bRet = ::CreatePipe(&hReadPipe, &hWritePipe, &securityAttributes, 0);
	if (FALSE == bRet)
	{
		printf("创建管道失败！\n");
		return FALSE;
	}
	// 设置新进程参数
	si.cb = sizeof(si);
	si.dwFlags = STARTF_USESHOWWINDOW | STARTF_USESTDHANDLES;
	si.wShowWindow = SW_HIDE;
	si.hStdError = hWritePipe;
	si.hStdOutput = hWritePipe;
	// 创建新进程执行命令, 将执行结果写入匿名管道中
	bRet = ::CreateProcess(NULL, pszCmd, NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi);
	if (FALSE == bRet)
	{
		printf("创建进程失败,没有该命令！\n");
		return false;
	}
	// 等待命令执行结束
	::WaitForSingleObject(pi.hThread, INFINITE);
	::WaitForSingleObject(pi.hProcess, INFINITE);
	// 从匿名管道中读取结果到输出缓冲区
	::RtlZeroMemory(pszResultBuffer, dwResultBufferSize);
	::ReadFile(hReadPipe, pszResultBuffer, dwResultBufferSize, NULL, NULL);
	// 关闭句柄, 释放内存
	
	::CloseHandle(pi.hThread);
	::CloseHandle(pi.hProcess);
	::CloseHandle(hWritePipe);
	::CloseHandle(hReadPipe);
	

	return TRUE;
}

```

![image-20211110101257715](https://imagesolo.oss-cn-beijing.aliyuncs.com/image-20211110101257715.png)

