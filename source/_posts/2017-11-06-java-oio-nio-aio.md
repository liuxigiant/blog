---
layout: post
title: "Java OIO NIO AIO简介"
description: Java OIO NIO AIO简介
date: 2017-11-06 10:53:53 +0800
catalog: true
categories:
- Java IO
tags:
- Java IO
---

随着Java语言的发展，陆续支持多种I/O模型，包括 OIO/BIO、NIO、AIO/NIO2.0，下面主要介绍下 NIO 和 AIO  

# 1 OIO/BIO  

OIO -- Old I/O，相对于后面的 NIO 来说的，也可称之为 BIO（Bolcking I/O 阻塞 I/O），是JAVA语言中的标准I/O，在 java.io 包下  

>基于流Stream的阻塞I/O，用的比较多都比较熟悉，不做过多阐述  

# 2 NIO  

NIO -- New I/O，新I/O，基于I/O多路复用模型，能支持高并发场景下大量I/O的管理。在 java.nio 包下，JDK1.4提供了对NIO的支持   

> NIO既然是New I/O，处理I/O模型的区别，相对于OIO/BIO，数据打包和传输的方式也有所不同。  
> OIO/BIO是以流Stream的方式一个字节一个字节地处理数据，而NIO是以数据块的方式处理数据    

JAVA NIO主要包含三个概念：缓冲区（Buffer）、通道（Channel）和选择器（Selector），下面一一了解  

> **通道**是对原 I/O 包中的流的模拟。到任何目的地(或来自任何地方)的所有数据都必须通过一个 Channel 对象。一个 **Buffer** 实质上是一个容器对象。发送给一个通道的所有对象都必须首先放到**缓冲区**中；同样地，从通道中读取的任何数据都要读到**缓冲区**中。

## 2.1 缓冲区Buffer  

### 2.1.1 概念  

Buffer 是一个对象， 它包含一些要写入或者刚读出的数据。 在 NIO 中加入 Buffer 对象，体现了新库与原 I/O 的一个重要区别。在面向流的 I/O 中，您将数据直接写入或者将数据直接读到 Stream 对象中。

在 NIO 库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的。在写入数据时，它是写入到缓冲区中的。任何时候访问 NIO 中的数据，您都是将它放到缓冲区中。

缓冲区实质上是一个数组。通常它是一个字节数组，但是也可以使用其他种类的数组。但是一个缓冲区不 仅仅 是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。  

### 2.1.2 类型  

既然缓冲区是数据容器，那么就会有类型，和其实就是底层数组的类型    

共8种类型：对应7种（除了boolean类型）基本数据类型和一个MappedByteBuffer（内存映射）  

- ByteBuffer
- MappedByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffe

### 2.2.3 内部细节  

前面有说道，通道Channel可理解为数据交互双方建立的连接信道，而缓冲区Buffer是数据的呈现方式，对通道数据的续写，其实都是通过对缓冲器的读写来完成的。  

JAVA NIO中的Buffer的底层是数据，但是为了满足读写需求，设计了巧妙的状态变量和访问方法  

#### 2.2.3.1 状态变量  

**状态变量**：简要来说就是当前缓冲器区是在读或者写的状态，而状态是由一组变量来体现的。每一个读/写操作都会改变缓冲区的状态。通过记录和跟踪这些变化，缓冲区就可能够内部地管理自己的资源。  

> 对缓冲器的读->写（由读变成写）、写->读（由写变成读）是需要切换缓冲区的状态的  

可以用三个值指定缓冲区在任意时刻的状态：  

- position  
作为一个内存块，Buffer有一个固定的大小值，也叫“capacity”.你只能往里写capacity个byte、long，char等类型。一旦Buffer满了，需要将其清空（通过读数据或者清除数据）才能继续写数据往里写数据。  
当读取数据时，也是从某个特定位置读。当将Buffer从写模式切换到读模式，position会被重置为0. 当从Buffer的position处读取数据时，position向前移动到下一个可读的位置。

- limit  
在写模式下，Buffer的limit表示你最多能往Buffer里写多少数据。 写模式下，limit等于Buffer的capacity。  
当切换Buffer到读模式时， limit表示你最多能读到多少数据。因此，当切换Buffer到读模式时，limit会被设置成写模式下的position值。换句话说，你能读到之前写入的所有数据（limit被设置成已写数据的数量，这个值在写模式下就是position）


- capacity
作为一个内存块，Buffer有一个固定的大小值，也叫“capacity”.你只能往里写capacity个byte、long，char等类型。一旦Buffer满了，需要将其清空（通过读数据或者清除数据）才能继续写数据往里写数据。  

#### 2.2.3.2 访问方法  

**访问方法**：对缓冲区操作的方法。在从通道读取数据时，数据被放入到缓冲区。在有些情况下，可以将这个缓冲区直接写入另一个通道，但是在一般情况下，您还需要查看数据。这是使用 访问方法 get() 来完成的。同样，如果要将原始数据放入缓冲区中，就要使用访问方法 put()

- allocate：buffer分配  

- flip： flip方法将Buffer从写模式切换到读模式。调用flip()方法会将position设回0，并将limit设置成之前position的值 
 
- rewind： Buffer.rewind()将position设回0，所以你可以重读Buffer中的所有数据。limit保持不变，仍然表示能从Buffer中读取多少个元素（byte、char等）  

- clear： 调用的是clear()方法，position将被设回0，limit被设置成 capacity的值。换句话说，Buffer 被清空了。Buffer中的数据并未清除，只是这些标记告诉我们可以从哪里开始往Buffer里写数据  

- compact： compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。limit属性依然像clear()方法一样，设置成capacity。现在Buffer准备好写数据了，但是不会覆盖未读的数据  

- mark与reset： 通过调用Buffer.mark()方法，可以标记Buffer中的一个特定position。之后可以通过调用Buffer.reset()方法恢复到这个positio  

- get() 方法  
ByteBuffer 类中有四个 get() 方法：  
byte get();  
ByteBuffer get( byte dst[] );  
ByteBuffer get( byte dst[], int offset, int length );  
byte get( int index );  
第一个方法获取单个字节。第二和第三个方法将一组字节读到一个数组中。第四个方法从缓冲区中的特定位置获取字节。那些返回 ByteBuffer 的方法只是返回调用它们的缓冲区的 this 值。  
此外，我们认为前三个 get() 方法是相对的，而最后一个方法是绝对的。 相对 意味着 get() 操作服从 limit 和 position 值 ― 更明确地说，字节是从当前 position 读取的，而 position 在 get 之后会增加。另一方面，一个 绝对 方法会忽略 limit 和 position 值，也不会影响它们。事实上，它完全绕过了缓冲区的统计方法。  
上面列出的方法对应于 ByteBuffer 类。其他类有等价的 get() 方法，这些方法除了不是处理字节外，其它方面是是完全一样的，它们处理的是与该缓冲区类相适应的类型。

- put()方法  
ByteBuffer 类中有五个 put() 方法：  
ByteBuffer put( byte b );  
ByteBuffer put( byte src[] );  
ByteBuffer put( byte src[], int offset, int length );  
ByteBuffer put( ByteBuffer src );  
ByteBuffer put( int index, byte b );  
第一个方法 写入（put） 单个字节。第二和第三个方法写入来自一个数组的一组字节。第四个方法将数据从一个给定的源 ByteBuffer 写入这个 ByteBuffer。第五个方法将字节写入缓冲区中特定的 位置 。那些返回 ByteBuffer 的方法只是返回调用它们的缓冲区的 this 值。  
与 get() 方法一样，我们将把 put() 方法划分为 相对 或者 绝对 的。前四个方法是相对的，而第五个方法是绝对的。  
上面显示的方法对应于 ByteBuffer 类。其他类有等价的 put() 方法，这些方法除了不是处理字节之外，其它方面是完全一样的。它们处理的是与该缓冲区类相适应的类型。

#### 2.2.3.3 简单示例  

channel的读写和buffer的读写方向是正好相反的  

```java
while (true) {
     buffer.clear();  // 重设缓冲区以便接收更多字节  缓冲区由读切换为写
     int r = fcin.read( buffer );  //读取通道fcin  写缓冲区buffer

     if (r==-1) {
       break;
     }
 
     buffer.flip(); // 准备读取缓冲区数据到通道  缓冲区由写切换为读
     fcout.write( buffer );  //写入通道fcout   读取缓冲区buffer
}
```

#### 2.2.3.4 其它  

缓冲区还有一些其它的灵活、高端的特性  

- 直接缓冲区  
缓冲区分配时候调用`ByteBuffer.allocateDirect`。直接缓冲区 是为加快 I/O 速度，而以一种特殊的方式分配其内存的缓冲区。(使用NIO可能导致直接内存溢出异常，就在于直接缓冲区的内存分配不在JVM堆上，在直接内存中分配)  

- 只读缓冲区  
创建缓冲区的直接副本`buffer.asReadOnlyBuffer()`，用于将缓冲区给其他方法使用，又不免被其他方法修改  

- 内存映射  
前面提到的8中buffer类型中比较特殊的一种MappedByteBuffer，用于内存和文件的映射，提高文件读写速度（并不是通过将文件读入内存，而是操作系统将磁盘文件映射到内存，JAVA只是提供了对该机制的访问）。  

- 分散和聚集  
通道可以有选择地实现两个新的接口： ScatteringByteChannel 和 GatheringByteChannel。Buffer的分散和聚集相对于单个buffer操作来说，一次操作的是一个buffer数组，读写都是把这个buffer数组当一个整体  

还有其他特性，不做说明了。。。

#### 2.2.3.5 总结  

缓冲区buffer底层是数组，有读和写两种状态，状态切换（由写切换成读或者由读切换成写）需要调用相应的切换方法，两种状态都由三个状态变量（capacity、position和limit）来体现，不同的访问方法（状态切换方法或者读写方法）会影响对应的状态变量  

## 2.2 通道Channel  

### 2.2.1 概念  

Channel是一个对象，可以通过它读取和写入数据。拿 NIO 与原来的 I/O 做个比较，通道就像是流。  

正如前面提到的，所有数据都通过 Buffer 对象来处理。您永远不会将字节直接写入通道中，相反，您是将数据写入包含一个或者多个字节的缓冲区。同样，您不会直接从通道中读取字节，而是将数据从通道读入缓冲区，再从缓冲区获取这个字节。  


### 2.2.2 类型  

通道与流的不同之处在于通道是双向的。而流只是在一个方向上移动(一个流必须是 InputStream 或者 OutputStream 的子类)， 而 通道 可以用于读、写或者同时用于读写。  

因为它们是双向的，所以通道可以比流更好地反映底层操作系统的真实情况。特别是在 UNIX 模型中，底层操作系统通道是双向的。  

常用的通道类如下：  

- FileChannel 从文件中读写数据 （不能切换到非阻塞模式，总是运行在阻塞模式下，所以不能与selector一起使用）  
- DatagramChannel 能通过UDP读写网络中的数据  
- SocketChannel 能通过TCP读写网络中的数据  
- ServerSocketChannel可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel  

## 2.3 选择器Selector  

### 2.3.1 概念  

选择器用于管理多个通道，监听注册通道的事件  

> 可以将选择器Selector与I/O模型中的多路复用模型中的select系统调用结合理解  

### 2.3.2 事件监听  

对于Java网络编程中，一般监听以下四种类型的事件：   
 
- Connect --   SelectionKey.OP_CONNECT
- Accept  --   SelectionKey. OP_ACCEPT
- Read    --   SelectionKey. OP_READ
- Write   --   SelectionKey. OP_WRITE


## 2.4 JAVA NIO示例  

下面给出一个JAVA网络编程中服务端使用NIO的示例  

```java
private void go() throws IOException {
    // Create a new selector
    Selector selector = Selector.open();

    // Open a listener on each port, and register each one
    // with the selector
    for (int i=0; i<ports.length; ++i) {
      ServerSocketChannel ssc = ServerSocketChannel.open();
      ssc.configureBlocking( false );
      ServerSocket ss = ssc.socket();
      InetSocketAddress address = new InetSocketAddress( ports[i] );
      ss.bind( address );

      SelectionKey key = ssc.register( selector, SelectionKey.OP_ACCEPT );

      System.out.println( "Going to listen on "+ports[i] );
    }

    while (true) {
        //我们调用 Selector 的 select() 方法。这个方法会阻塞，直到至少有一个已注册的事件发生。当一个或者更多的事件发生时， select() 方法将返回所发生的事件的数量
      int num = selector.select(); 

      Set selectedKeys = selector.selectedKeys();
      Iterator it = selectedKeys.iterator();

      while (it.hasNext()) {
        SelectionKey key = (SelectionKey)it.next();

        if ((key.readyOps() & SelectionKey.OP_ACCEPT)
          == SelectionKey.OP_ACCEPT) {
          // Accept the new connection
          ServerSocketChannel ssc = (ServerSocketChannel)key.channel();
          SocketChannel sc = ssc.accept();
          sc.configureBlocking( false );

          // Add the new connection to the selector
          SelectionKey newKey = sc.register( selector, SelectionKey.OP_READ );
          it.remove();

          System.out.println( "Got connection from "+sc );
        } else if ((key.readyOps() & SelectionKey.OP_READ)
          == SelectionKey.OP_READ) {
          // Read the data
          SocketChannel sc = (SocketChannel)key.channel();

          // Echo data
          int bytesEchoed = 0;
          while (true) {
            echoBuffer.clear();

            int r = sc.read( echoBuffer );

            if (r<=0) {
              break;
            }

            echoBuffer.flip();

            sc.write( echoBuffer );
            bytesEchoed += r;
          }

          System.out.println( "Echoed "+bytesEchoed+" from "+sc );

          it.remove();
        }

      }

//System.out.println( "going to clear" );
//      selectedKeys.clear();
//System.out.println( "cleared" );
    }
  }

```

# 3 AIO  

AIO -  Asynchronous I/O，也可称之为NIO2.0，是真正的异步I/O模型，引入了异步通道。在java.io包下，JDK7提供了对AIO的支持  

AIO 包括 Sockets 和 Files 两部分的异步通道接口及其实现  


## 3.1 异步通道  

AIO中引入了异步通道，重点关注下网络编程中服务端和客户端异步通道  

- 服务端  AsynchronousServerSocketChannel  
- 客户端  AsynchronousSocketChannel

## 3.2 异步回调  

异步I/O的核心在于I/O操作完成后，能触发回调函数，JAVA AIO 中对异步通道的 I/O 操作，提供了两种回调函数的触发方式的支持：  

- Future 方式  
调用方法返回Future，通过Future完成回调函数的处理（future方式需要考虑get方法是同步的）  

- Callback 方式  
在调用方法时候，直接传递Callback函数（一般是实现CompletionHandler接口）  

可以对照`AsynchronousSocketChannel#read`的两种方法签名来理解以上两种方式  

```java
//Future 方式
public abstract Future<Integer> read(ByteBuffer dst)

//Callback 方式
public final <A> void read(ByteBuffer dst,
                               A attachment,
                               CompletionHandler<Integer,? super A> handler)
```

> 对于**Future方式**需要轮询I/O操作完成后，再调用回调函数处理，更像是兼容阻塞I/O模型和I/O多路复用模型的处理方式；  
> 而对于**Callbac方式**直接传递回调函数，则更像是异步I/O模型中回调函数的触发方式，无需显示调用回调处理函数     

## 3.3 NIO与AIO对比  

- NIO 是将通道Channel和需要关注的事件注册到选择器Selector上，选择器Selector同步轮询事件，然后当事件触发时调用处理函数  

> NIO其实还是会在读写数据过程中阻塞，读写完毕才能执行其他操作  
> 选择器的select方法是同步阻塞的  
> 虽然select方法是同步阻塞的，但是可以将select操作交由单独的线程处理，而回调函数的处理由其他线程完成，做到伪异步化（Reactor线程模型，后续再讨论）  

  
- AIO 是异步的，不用注册事件，是在IO操作时候直接附加一个回调，IO操作完成后，触发回调，AIO不会在读写数据过程中阻塞，读写完毕才回调  

> 在API使用的方式上的区别在于：NIO是注册事件和处理事件；AIO是调用I/O操作，传递回调函数。  

## 3.4 示例  

略，可参照下面参考资料中的链接    

# 4 The End  

最后说明下，本文其实是本人的一个学习笔记，内容参考自其他文章，主要是做一个系统的归纳，不免有拼凑别人文章之嫌 -_-  

**参考文章：**  

[http://ifeve.com/java-nio-all/]()  
[https://www.ibm.com/developerworks/cn/education/java/j-nio/j-nio.html]()  
[http://colobu.com/2014/11/13/java-aio-introduction/]()  
[http://blog.csdn.net/anxpp/article/details/51512200]()  包含AIO示例  
[https://www.ibm.com/developerworks/cn/java/j-lo-nio2/index.html]()  包含AIO中两种回调方式  






