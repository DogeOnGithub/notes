# Java NIO

---

[参考链接：Java NIO？看这一篇就够了！](https://blog.csdn.net/forezp/article/details/88414741)

[参考链接：NIO buffer中clear、compact方法的区别](https://blog.csdn.net/weixin_40581455/article/details/83143634)

---

## 概述

+ NIO主要有三大核心部分：`Channel(通道)`，`Buffer(缓冲区)`, `Selector`
+ 传统IO基于字节流和字符流进行操作，而NIO基于Channel和Buffer(缓冲区)进行操作，`数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中`
+ Selector(选择区)用于监听多个通道的事件（比如：连接打开，数据到达）
+ NIO单个线程可以监听多个数据通道

+ NIO和传统IO（一下简称IO）之间第一个最大的区别是，IO是面向流的，`NIO是面向缓冲区的`
+ Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方
+ Java IO中stream不能前后移动流中的数据，如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区
+ NIO中数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的灵活性
+ IO的各种流是阻塞的，这意味着，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入，该线程在此期间不能再干任何事情了
+ NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它`仅能得到目前可用的数据`，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变得可以读取之前，该线程可以继续做其他的事情，非阻塞写也是如此，一个线程请求写入一些数据到某通道，但`不需要等待它完全写入`，这个线程同时可以去做别的事情，线程通常将非阻塞IO的空闲时间用于在其它通道上执行IO操作，所以一个单独的线程现在可以管理多个输入和输出通道（channel）

## Channel

+ NIO中的channel基本等价于传统IO的stream，或者说是处于IO操作中的等价地位，但stream是单向的，而channel这是双向的（即可以进行读操作和写操作）
+ NIO中的Channel的主要实现有：`FileChannel`，`DatagramChannel`，`SocketChannel`，`ServerSocketChannel`
+ 以上实现分别可以对应文件IO、UDP和TCP（Server和Client）

## Buffer

NIO中的关键Buffer实现有：`ByteBuffer`，`CharBuffer`，`DoubleBuffer`，`FloatBuffer`，`IntBuffer`，`LongBuffer`，`ShortBuffer`，分别对应基本数据类型: byte, char, double, float, int, long, short
NIO中还有`MappedByteBuffer`，`HeapByteBuffer`，`DirectByteBuffer`

## Selector

+ Selector运行单线程处理多个Channel
+ 如果应用打开了多个通道，但每个连接的流量都很低，使用Selector就会很方便（例如在一个聊天服务器中，要使用Selector，得向Selector注册Channel，然后调用它的select()方法，这个方法会一直阻塞到某个注册的通道有事件就绪，一旦这个方法返回，线程就可以处理这些事件，事件的例子有如新的连接进来、数据接收等）

## FileChannel

```java
package top.tjsanshao;

import java.io.File;
import java.io.FileInputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.Charset;
import java.nio.charset.CharsetDecoder;
import java.nio.charset.StandardCharsets;

/**
 * file channel
 *
 * @author TjSanshao
 * @date 2020-06-12 14:11
 */
public class FileChannelTest01 {
    public static void main(String[] args) {
        try {
            File file = new File("D:\\projects\\thread-interupt-test\\src\\main\\java\\top\\tjsanshao\\Test01.java");
            FileChannel channel = new FileInputStream(file).getChannel();
            // allocate后，position=1,limit=64,capacity=64,mark=-1
            ByteBuffer buffer = ByteBuffer.allocate(64);
            int read = channel.read(buffer);
            while (read != -1) {
                // flip方法后，limit=position，同时position归0，mark=-1
                buffer.flip();
                while (buffer.hasRemaining()) {
                    System.out.println(StandardCharsets.UTF_8.decode(buffer));
                    // 上面使用了整个buffer来decode，因此，这里输出的position是64，但如果使用buffer.get()方法，则position是会逐渐递增1的
                    System.out.println(buffer.position());
                }
                // 如果使用buffer.clear()方法，position将被设回0，limit设置成capacity，换句话说，Buffer被清空了，但实际上buffer中仍然还有数据，但已经无法读取，因为position已经为0
                // 如果Buffer中仍有未读的数据，且后续还需要这些数据，但是此时想要先写些数据，那么使用compact()方法
                // compact()方法将所有未读的数据拷贝到Buffer起始处，然后将position设到最后一个未读元素正后面
                // limit属性依然像clear()方法一样，设置成capacity
                // 现在Buffer准备好写数据了，但是不会覆盖未读的数据
                buffer.compact();
                read = channel.read(buffer);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## SocketChannel Selector

```java
package top.tjsanshao;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectableChannel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;

/**
 * nio client demo
 *
 * @author TjSanshao
 * @date 2020-06-15 14:10
 */
public class NIOClientDemo02 {
    private Selector selector;

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                final NIOClientDemo02 client = new NIOClientDemo02();
                client.connect("127.0.0.1", 8080);
                client.listen();
            }).start();
        }
    }

    public void connect(String host, int port) {
        try {
            final SocketChannel channel = SocketChannel.open();
            this.selector = Selector.open();
            channel.configureBlocking(false);
            channel.register(this.selector, SelectionKey.OP_CONNECT);
            channel.connect(new InetSocketAddress(host, port));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void listen() {
        while (true) {
            try {
                final int events = this.selector.select();
                if (events > 0) {
                    final Iterator<SelectionKey> iterator = this.selector.selectedKeys().iterator();
                    while (iterator.hasNext()) {
                        final SelectionKey next = iterator.next();
                        iterator.remove();
                        if (next.isConnectable()) {
                            final SocketChannel channel = (SocketChannel) next.channel();
                            if (channel.isConnectionPending()) {
                                channel.finishConnect();
                            }
                            channel.configureBlocking(false);
                            channel.register(this.selector, SelectionKey.OP_READ);
                            channel.write(ByteBuffer.wrap(("From client message.").getBytes()));
                        } else {
                            if (next.isReadable()) {
                                final SocketChannel channel = (SocketChannel) next.channel();
                                final ByteBuffer buffer = ByteBuffer.allocate(1024);
                                channel.read(buffer);
                                buffer.flip();
                                System.out.println("From server:" + StandardCharsets.UTF_8.decode(buffer));
                            }
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
package top.tjsanshao;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectableChannel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.CompletableFuture;

/**
 * nio server demo
 *
 * @author TjSanshao
 * @date 2020-06-15 13:03
 */
public class NIOServerDemo01 {
    public static void main(String[] args) {
        nioServer();
    }

    private static void nioServer() {
        try (final ServerSocketChannel channel = ServerSocketChannel.open(); final Selector selector = Selector.open()) {
            channel.configureBlocking(false);
            channel.bind(new InetSocketAddress(8080));
            channel.register(selector, SelectionKey.OP_ACCEPT);

            while (true) {
                final int select = selector.select();
                if (select > 0) {
                    final Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                    while (iterator.hasNext()) {
                        final SelectionKey next = iterator.next();
                        iterator.remove();
                        if (next.isAcceptable()) {
                            accept(next);
                        } else {
                            CompletableFuture.runAsync(() -> {
                               if (next.isReadable()) {
                                   try {
                                       SocketChannel socketChannel = (SocketChannel) next.channel();
                                       final ByteBuffer buffer = ByteBuffer.allocate(1024);
                                       socketChannel.read(buffer);
                                       buffer.flip();
                                       System.out.println(StandardCharsets.UTF_8.decode(buffer));
                                   } catch (Exception e) {
                                       e.printStackTrace();
                                   }
                               }
                            });
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void accept(SelectionKey key) {
        try {
            final ServerSocketChannel channel = (ServerSocketChannel) key.channel();
            final SocketChannel accept = channel.accept();
            accept.configureBlocking(false);
            accept.register(key.selector(), SelectionKey.OP_READ);
            System.out.println("accepted");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
