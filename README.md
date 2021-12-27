# rtspscan
## 原理分析

RTSP（Real Time Streaming Protocol）是由Real Network和Netscape共同提出的如何有效地在IP网络上传输流媒体数据的应用层协议。RTSP对流媒体提供了诸如暂停，快进等控制，而它本身并不传输数据，RTSP的作用相当于流媒体服务器的远程控制。服务器端可以自行选择使用TCP或UDP来传送串流内容，它的语法和运作跟HTTP 1.1类似，但并不特别强调时间同步，所以比较能容忍网络延迟。

RTSP五步会话原理如下

```
step1:
C->S:OPTION request //询问S有哪些方法可用
S->C:OPTION response //S回应信息中包括提供的所有可用方法

step2:
C->S:DESCRIBE request //要求得到S提供的媒体初始化描述信息
S->C:DESCRIBE response //S回应媒体初始化描述信息，主要是sdp

step3:
C->S:SETUP request //设置会话的属性，以及传输模式，提醒S建立会话
S->C:SETUP response //S建立会话，返回会话标识符，以及会话相关信息

step4:
C->S:PLAY request //C请求播放
S->C:PLAY response //S回应该请求的信息
S->C:发送流媒体数据

step5:
C->S:TEARDOWN request //C请求关闭会话
S->C:TEARDOWN response //S回应该请求
```

wireshark实战抓包，获取最开始的几步认证即可。

```
OPTIONS rtsp://207.62.193.8:8554 RTSP/1.0
CSeq: 1
User-Agent: Lavf58.29.100

RTSP/1.0 200 OK
Server: StarDot H264 Server
CSeq: 1
Public: DESCRIBE,OPTIONS,PLAY,SETUP,TEARDOWN

DESCRIBE rtsp://207.62.193.8:8554 RTSP/1.0
Accept: application/sdp
CSeq: 2
User-Agent: Lavf58.29.100

RTSP/1.0 200 OK
Server: StarDot H264 Server
CSeq: 2
Cache-Control: no-cache
Content-Length: 289
Content-Type: application/sdp
```

## 利用方式

观察一个比较完整rtsp格式

```
rtsp://user:pass@111.111.111.111/xxx/xxx
```

因此检测的时候主要分几种类型：

1、rtsp未授权访问

2、404需要爆破路径

3、401需要帐密认证

工具思路：

1、先对所有地址进行存活检测。

2、对于存活的地址，如果返回200，则存在未授权访问，返回401则进行帐密爆破，返回404则先进行路径爆破，路径爆破成功则进行帐密爆破。

**构造socket包**

```go
func genOPTIONS(url string, seq int, ua string) string {
    msgRet := "OPTIONS " + url + " RTSP/1.0\r\n"
    msgRet += "CSeq: " + strconv.Itoa(seq) + "\r\n"
    msgRet += "User-Agent: " + ua + "\r\n"
    msgRet += "\r\n"
    return msgRet
}

func genDESCRIBLE(url string, seq int, ua string) string {
    msgRet := "DESCRIBE " + url + " RTSP/1.0\r\n"
    msgRet += "CSeq: " + strconv.Itoa(seq) + "\r\n"
    msgRet += "User-Agent: " + ua + "\r\n"
    msgRet += "Accept: application/sdp\r\n"
    msgRet += "\r\n"
    return msgRet
}
```

**socket通信**

```go
func Handler(serv models.Service) int {
    ua := "LibVLC/2.0.3 (LIVE555 Streaming Media v2011.12.23)"
    addr := fmt.Sprintf("%v:%v", serv.Ip, serv.Port)
    conn, err := net.DialTimeout("tcp", addr, vars.Timeout)
    if err != nil {
        return 0
    }
    conn.SetReadDeadline(time.Now().Add(vars.Timeout))
    conn.SetWriteDeadline(time.Now().Add(vars.Timeout))
    seq := 1
    msg := genOPTIONS(serv.Url, seq, ua)
    data := make([]byte, 255)
    _, err = conn.Write([]byte(msg))
    _, err = conn.Read(data)
    if err != nil {
        return 0
    }
    seq += 1
    msg = genDESCRIBLE(serv.Url, seq, ua)
    _, err = conn.Write([]byte(msg))
    _, err = conn.Read(data)
    if err != nil {
        return 0
    }
    if strings.Contains(string(data), "200 OK") {
        return 200
    }
    if strings.Contains(string(data), "401 Unauthorized") {
        return 401
    }
    if strings.Contains(string(data), "404 Not Found") {
        return 404
    }
    return 0
}
```

## 运行截图

![image-20211227095324636](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227095324636.png)

![image-20211227095425403](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227095425403.png)

## 免责声明

本工具仅面向**合法授权**的企业安全建设行为，且仅供学习研究自查使用，切勿用于非法用途，由使用该工具产生的一切风险均与本人无关！
