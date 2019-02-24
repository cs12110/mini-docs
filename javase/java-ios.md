# IOs

当二进制流在时间的长河里面流动,有无数的机械臂在搅动着这条长河,不是I,就是O.

小指头对TCP说: 混乱就是阶梯.

电脑越来越慢了, 泪目.


---

## 1. BIO

之前一直不太了解NIO/和IO的区别,所以花了一点时间去查资料,然后还是不明白. orz

但可以得出的初步结论如下:**传统io单线程处理的弊端,一个服务端无法为多个客户端提供服务.**

### 1.1 测试代码

```java
package com.pkgs;

import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * TODO:
 *
 * @author cs12110 create at: 2019/2/24 13:57
 * Since: 1.0.0
 */
public class SocketServerApp {

    public static void main(String[] args) {
        try {
            int port = 7799;
            ServerSocket serverSocket = new ServerSocket(port);
            System.out.println("Start socket at port: " + port);

            while (true) {
                Socket accept = serverSocket.accept();
                System.out.println("Accept: " + accept.getPort());
                handler(accept);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void handler(Socket socket) {
        try {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            byte[] arr = new byte[1024];
            InputStream inputStream = socket.getInputStream();
            int len;

            StringBuilder builder = new StringBuilder();
            while (-1 != (len = inputStream.read(arr))) {
                builder.append(new String(arr, 0, len));
            }
            inputStream.close();
            System.out.println(sdf.format(new Date()) + " - " + builder);

            // we take 2 seconds to consumer this message
            Thread.sleep(2000);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                socket.close();
            } catch (Exception e) {
                // do nothing
            }
        }
    }
}
```

```java
package com.pkgs;

import java.io.OutputStream;
import java.net.Socket;

/**
 * TODO:
 *
 * @author cs12110 create at: 2019/2/24 14:09
 * Since: 1.0.0
 */
public class ClientSocketApp1 {

    public static void main(String[] args) {
        String msg = ClientSocketApp1.class.getName();
        try {
            int times = 4;
            int index = 0;
            while (index++ < times) {
                System.out.println("Sending: " + index);

                Socket socket = new Socket("127.0.0.1", 7799);
                OutputStream outputStream = socket.getOutputStream();
                outputStream.write((index + ":" + msg).getBytes());
                outputStream.flush();
                outputStream.close();

                System.out.println("Sending: " + index + " is done");
                socket.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 1.2 测试结果

```java
Sending: 1
Sending: 1 is done
Sending: 2
Sending: 2 is done
Sending: 3
Sending: 3 is done
Sending: 4
Sending: 4 is done
```

```java
Start scoket at port: 7799
Accept: 53275
2019-02-24 21:33:23 - 1:com.pkgs.ClientSocketApp1
Accept: 53276
2019-02-24 21:33:25 - 2:com.pkgs.ClientSocketApp1
Accept: 53277
2019-02-24 21:33:27 - 3:com.pkgs.ClientSocketApp1
Accept: 53278
2019-02-24 21:33:29 - 4:com.pkgs.ClientSocketApp1
```

上面可以看出,同一时间服务端只能处理一个连接.

这个处理方式用在高并发下,只能删库跑路了,babe.


### 1.3 演变

Q: 老师,老师,那我可不可以为每一个socket都开一个线程处理呀?就像下面这个. 轻蔑一笑.jpg(气氛突然变得有点紧张起来)

```java
package com.pkgs;

import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * TODO:
 *
 * @author cs12110 create at: 2019/2/24 13:57
 * Since: 1.0.0
 */
public class SocketServerApp {

    public static void main(String[] args) {
        try {
            int port = 7799;
            ServerSocket serverSocket = new ServerSocket(port);
            System.out.println("Start socket at port: " + port);

            // 线程池
            ExecutorService threadPool = Executors.newCachedThreadPool();

            while (true) {
                Socket accept = serverSocket.accept();
                System.out.println("Accept: " + accept.getPort());

                // 使用新线程处理socket
                threadPool.submit(new SocketHandler(accept));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static class SocketHandler implements Runnable {
        private Socket socket;

        SocketHandler(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            handler(socket);
        }
    }

    private static void handler(Socket socket) {
        try {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            byte[] arr = new byte[1024];
            InputStream inputStream = socket.getInputStream();
            int len;

            StringBuilder builder = new StringBuilder();
            while (-1 != (len = inputStream.read(arr))) {
                builder.append(new String(arr, 0, len));
            }
            inputStream.close();
            System.out.println(sdf.format(new Date()) + " - " + builder);

            // we take 2 seconds to consumer this message
            Thread.sleep(2000);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                socket.close();
            } catch (Exception e) {
                // do nothing
            }
        }
    }
}
```

```java
Start socket at port: 7799
Accept: 53479
Accept: 53480
Accept: 53481
Accept: 53482
2019-02-24 21:42:10 - 3:com.pkgs.ClientSocketApp1
2019-02-24 21:42:10 - 1:com.pkgs.ClientSocketApp1
2019-02-24 21:42:10 - 2:com.pkgs.ClientSocketApp1
2019-02-24 21:42:10 - 4:com.pkgs.ClientSocketApp1
```

A: 这样子的确做到了处理多个socket了.如果并发10k的话,就有10k线程了,上下文切换会对资源产生很大的消耗.如果你家有矿的话,的确可以...,但是都有矿了,为什么还要和代码过不去呀,求你了,放过代码吧.

---

## 2. NIO

这时候,NIO面对高并发的时候,一手把前面的BIO推开说:垃圾,闪开,让我来.

写不下去了,痛哭流涕.

### 2.1 

---

## 参考资料

a. [传统io和NIO详细比较](https://blog.csdn.net/qq_22933035/article/details/79967791)