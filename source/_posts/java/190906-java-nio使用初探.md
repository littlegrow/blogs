---
title: Java NIO 使用初探
date: 2019-9-6 19:00:00
description: "NIO三大核心部分Channel，Buffer，Selector"
categories:
- NIO
---

### 概述

<br>

NIO主要有三大核心部分：Channel(通道)，Buffer(缓冲区), Selector。传统IO基于字节流和字符流进行操作，而NIO基于Channel和Buffer(缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector(选择区)用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个线程可以监听多个数据通道。

<br>

#### Channel

<br>

Channel和IO中的Stream差不多。只是Stream是单向的，而Channel是双向的，既可以用来进行读操作，又可以用来进行写操作。
NIO中的Channel的主要有：

    FileChannel               文件IO
    DatagramChannel           UDP
    SocketChannel             TCP Client
    ServerSocketChannel       TCP server

<br>

#### Buffer

<br>

NIO中的关键Buffer实现有：ByteBuffer, CharBuffer, DoubleBuffer, FloatBuffer, IntBuffer, LongBuffer, ShortBuffer，分别对应基本数据类型: byte, char, double, float, int, long, short。
NIO中还有MappedByteBuffer, HeapByteBuffer, DirectByteBuffer.

<br>

#### Selector

<br>

Selector运行单线程处理多个Channel，如果你的应用打开了多个通道，但每个连接的流量都很低，使用Selector就会很方便。例如在一个聊天服务器中。要使用Selector, 得向Selector注册Channel，然后调用它的select()方法。这个方法会一直阻塞到某个注册的通道有事件就绪。一旦这个方法返回，线程就可以处理这些事件，事件的例子有如新的连接进来、数据接收等。

<br>

### 文件读取

<br>

#### 传统IO读取文件

<br>

使用 InputStream 读取文件：

```
void readFileByOriginIo(String filePath) {
    try {
        InputStream inputStream = new FileInputStream(filePath);
        byte[] buffer = new byte[1024];
        int readSize = inputStream.read(buffer);
        while (readSize != -1) {
            for (int i = 0; i < readSize; i++) {
                System.out.print((char) buffer[i]);
            }
            readSize = inputStream.read(buffer);
        }
        inputStream.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

<br>

#### NIO读取文件

<br>

使用 Channel、Buffer读取文件内容：

```
void nioFileChannelTest(String filePath) {
    RandomAccessFile randomAccessFile = null;
    try {
        randomAccessFile = new RandomAccessFile(filePath, "r");
        FileChannel fileChannel = randomAccessFile.getChannel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int readSize = fileChannel.read(buffer);
        while (readSize != -1) {
            buffer.flip();
            while (buffer.hasRemaining()) {
                System.out.print((char) buffer.get());
            }
            buffer.compact();
            readSize = fileChannel.read(buffer);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (randomAccessFile != null) {
            try {
                randomAccessFile.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

<br>

### Socket通信

<br>

#### 客户端

<br>

```
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class NioClient {

    private static void client() {
        Random random = new Random(System.currentTimeMillis());
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        SocketChannel socketChannel = null;
        try {
            socketChannel = SocketChannel.open();
            socketChannel.configureBlocking(false);
            socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
            if (socketChannel.finishConnect()) {
                TimeUnit.MILLISECONDS.sleep(random.nextInt(500));
                String info = Thread.currentThread().getName() + "information";
                buffer.clear();
                buffer.put(info.getBytes());
                buffer.flip();
                socketChannel.write(buffer);
            }
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        } finally {
            try {
                if (socketChannel != null) {
                    System.out.println("close socket");
                    socketChannel.shutdownInput();
                    socketChannel.shutdownOutput();
                    socketChannel.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        AtomicInteger integer = new AtomicInteger(0);
        ExecutorService executorService = Executors.newCachedThreadPool(r -> new Thread(r, "client-" + integer.addAndGet(1)));
        for (int i = 0; i < 10; i++) {
            executorService.submit(NioClient::client);
        }
    }
}
```

<br>

#### 服务端

<br>

```
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;

public class NioServer {
    private static final int BUFFER_SIZE = 1024;
    private static final int PORT = 8080;
    private static final int TIMEOUT = 300000;

    public static void main(String[] args) {
        selector();
    }

    private static void selector() {
        Selector selector = null;
        ServerSocketChannel serverSocketChannel = null;

        try {
            selector = Selector.open();
            serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.socket().bind(new InetSocketAddress(PORT));
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

            for (; ; ) {
                if (selector.select(TIMEOUT) == 0) {
                    System.out.println("==");
                    continue;
                }
                Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                try {
                    while (iterator.hasNext()) {
                        SelectionKey selectionKey = iterator.next();
                        if (!selectionKey.isValid()) {
                            System.out.println(":invalid");
                            iterator.remove();
                            continue;
                        }
                        if (selectionKey.isAcceptable()) {
                            System.out.println(":accept");
                            handleAccept(selectionKey);
                        }
                        if (selectionKey.isReadable()) {
                            System.out.println(":read");
                            handleRead(selectionKey);
                        }
                        if (selectionKey.isWritable()) {
                            System.out.println(":write");
                            handleWrite(selectionKey);
                        }
                        if (selectionKey.isConnectable()) {
                            System.out.println("is connecting");
                        }
                        iterator.remove();
                    }
                } catch (CancelledKeyException e) {
                    e.printStackTrace();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (selector != null) {
                    selector.close();
                }
                if (serverSocketChannel != null) {
                    serverSocketChannel.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private static void handleAccept(SelectionKey selectionKey) throws IOException {
        ServerSocketChannel serverSocketChannel = (ServerSocketChannel) selectionKey.channel();
        SocketChannel socketChannel = serverSocketChannel.accept();
        System.out.println("Accept: " + socketChannel.getRemoteAddress());
        socketChannel.configureBlocking(false);
        socketChannel.register(selectionKey.selector(), SelectionKey.OP_READ, ByteBuffer.allocateDirect(BUFFER_SIZE));
    }

    private static void handleRead(SelectionKey selectionKey) throws IOException {
        SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
        ByteBuffer buffer = (ByteBuffer) selectionKey.attachment();
        while (socketChannel.read(buffer) > 0) {
            buffer.flip();
            while (buffer.hasRemaining()) {
                System.out.print((char) buffer.get());
            }
            System.out.println();
            buffer.clear();
        }
    }

    private static void handleWrite(SelectionKey selectionKey) throws IOException {
        ByteBuffer buf = (ByteBuffer) selectionKey.attachment();
        buf.flip();
        SocketChannel sc = (SocketChannel) selectionKey.channel();
        while (buf.hasRemaining()) {
            sc.write(buf);
        }
        buf.compact();
    }
}
```

<br>

参考资料：
[Java NIO](https://blog.csdn.net/u011381576/article/details/79876754)

<br><br><br><br>
