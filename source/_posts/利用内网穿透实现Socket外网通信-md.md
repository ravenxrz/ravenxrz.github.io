---
title: 利用内网穿透实现Socket外网通信
date: 2019-07-09 13:58:41
categories: 杂文
toc: true
tags: Socket
---

## 1. 前言

> 需求提出：还有两三天就要参加毕业设计答辩了。一直在考虑怎么样给老师做展示才好。由于毕设做的是基于深度学习的去雾研究。只有一台PC，也就意味着只能录屏？感觉录屏不好啊，所以想了下要不**做个Android App展示去雾**？可是研究了下**把Keras模型移植到Android上**似乎也不太轻松，因为我的模型中有些自定义层，不是标准的模型。**再来就想要不把模型放到服务器上**？但是我的服务器性能这么垃圾，而且服务器在国外，上传下载速度也不行。最后，要不就拿现在这台PC来做？可是问题来了，我怎么才能**走广域网连接PC**呢，毕竟答辩不可能在寝室。
> OK，百度一波，还是选择内网穿透吧。
>
> 第一次接触内网穿透是在去年10月还是11月的**[“搭建aria2远程下载机”](https://www.ravenxrz.ink/2019/05/03/reuse-of-old-mobile-phonea-downloader-based-on-arria2.html)**。所以对内网穿透这个名词还不算陌生。（看样子曾经无聊搞的东西还是可能在未来帮到自己啊）
<!-- more -->
[post cid="36" /]


好了，废话了一堆，本文要做的功能有：

1. 以台式机作为服务器，等待客户端上传图片，服务器将图片去雾后回传给客户端。
2. 既然台式机要做服务器，那就必须来个内网穿透了，不然外网的手机是不能连接到PC的。
3. 服务器与客户端的通信使用的Socket

## 2. 正文“斯达头”（start）

### 2.1 材料准备

1. PC机
3. 花生壳账号

### 2.2 花生壳注册与端口映射

登陆官网，并注册：https://hsk.oray.com/ 

然后进入控制台。进入域名注册：

![](https://pic.superbed.cn/item/5cfc9796451253d178e53687.jpg)

![](https://pic.superbed.cn/item/5cfc9814451253d178e53c32.jpg)

自己注册一个域名吧，用不了多少钱。

然后打开内网穿透，新增映射。

![](https://cy-pic.kuaizhan.com/g3/73/28/ed54-5ad8-4b9e-aae1-38bb14fd4a1132)

填写你的相关信息。

![](https://cy-pic.kuaizhan.com/g3/ff/3c/1bfb-b4b7-45a5-9763-763a31e99c8758)

这里再说一下如何查IP。

方法一：如果使用了路由器，可以打开路由器查：

![](https://ae01.alicdn.com/kf/HTB1SV79b8Kw3KVjSZFOq6yrDVXaB.jpg)

方法二：直连网线或wifi的，在cmd窗口用ipconfig命令查看。

![](https://ae01.alicdn.com/kf/HTB1T4Xacbys3KVjSZFnq6xFzpXaK.jpg)

注意你的连接方式，我是有线所以找的以太网适配器。如果是无线，记得找无线适配器的地址。

最后下载花生壳的内网穿透客户端到PC上，记得**下载3.0版本**，5.0 beta版经测试还是有bug。

花生壳3.0 版本：https://hsk.oray.com/download/download?id=hskddns_310

**下载好后，安装并登陆打开。**

### 2.3 测试是否映射成功

打开在线端口映射工具：http://tool.chinaz.com/port/

![](https://ae01.alicdn.com/kf/HTB1y6XXcbus3KVjSZKbq6xqkFXa7.jpg)

记住外网的域名和端口，依此填入即可。

![](https://ae01.alicdn.com/kf/HTB1I2g3b25G3KVjSZPxq6zI3XXab.jpg)

显示“开启”字样，则表示穿透成功。

## 3. 测试Socket通信

在我的测试中，python作为Socket服务端，Android作为Socket客户端。对应代码如下

### 3.1 Socket服务端

```python
def socket_service():
    """
    开启socket服务
    :return:
    """
    # 开启socket服务
    try:
        s = socket.socket()
        s.bind(('192.168.1.104', 6666))
        s.listen(10)
    except socket.error as msg:
        print(msg)
        sys.exit(1)

    print("Wait")

    def save_haze_file(sock):
        """
        从sock中获取数据，并保存下来
        :param sock:
        :return:
        """
        with open(haze_path, 'wb') as f:
            print('file opened')
            while True:
                data = sock.recv(1024)
                # print()
                if not data:
                    break
                elif 'EOF' in str(data):
                    f.write(data[:-len('EFO')])
                    break
                # write data to a file
                f.write(data)
        # sock.close()
        print('pic received finished')

    def send_img(sock):
        # 发送处理后的图片
        with open(dehaze_path, 'rb') as f:
            for data in f:
                sock.send(data)
        print('send finished')

    # 等待连接并处理
    idx = 0
    while True:
        sock, _ = s.accept()
        # idx += 1
        # if idx == 1:
        #     save_haze_file(sock)
        # elif idx == 2:
        #     idx = 0
        try:
            save_haze_file(sock)
            dehaze()
            send_img(sock)
        except Exception as reason:
            print(reason)
        finally:
            sock.close()

```

上述代码，就是建立Socket服务端，服务端的IP为内网IP。



### 3.2 Socket客户端

```java
 private final String IP = "xxx";		// 这个地方的IP为内网穿透时给的ip
private final int PORT= xxx;					// 同理为给的端口

private void sendImg(String path) throws IOException {
    // 创建一个Socket对象，并指定服务端的IP及端口号
    socket = new Socket(IP, PORT);
    InputStream inputStream = new FileInputStream(path);
    // 获取Socket的OutputStream对象用于发送数据。
    OutputStream outputStream = socket.getOutputStream();
    // 创建一个byte类型的buffer字节数组，用于存放读取的本地文件
    byte buffer[] = new byte[10 * 1024];
    int temp = 0;
    // 循环读取文件
    while ((temp = inputStream.read(buffer)) != -1) {
        // 把数据写入到OuputStream对象中
        outputStream.write(buffer, 0, temp);
        // 发送读取的数据到服务端
        outputStream.flush();
    }
    socket.close();
}
```

注意要修改的地址为IP和PORT。

查看方式如下：

![](https://ae01.alicdn.com/kf/HTB1W7M3b4iH3KVjSZPfq6xBiVXay.jpg)

![](https://ae01.alicdn.com/kf/HTB1KzSeXvBj_uVjSZFpq6A0SXXaS.jpg)

**红框表明的即为需要修改的IP，端口为“映射”字段后写的端口。**



## 4. 结语

至此，已经实现了内网穿透并且通过Socket通信了。其余应用功能只用映射为其它端口即可。

本文涉及的所有代码已放于GIthub:https://github.com/ravenxrz/DeBlurGanToDehaze
