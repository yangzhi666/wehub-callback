## faq(持续更新中)

-  1.如何查看wehub与 回调接口之间的数据通讯?

```
方法1:安装fiddler,该软件可以很直接的观察到wehub 的所有的http通讯
    在本机上开发和部署回调接口服务的开发者,需下载最新版wehub,打开设置界面,切换到"其他设置"-->设置http代理为127.0.0.1:8888(保持和fiddler默认的代理设置一致),然后就可以在fiddler中查看wehub发送的http request和接收到的respone
	
方法2:Wehub在 C:\Users\xxxxxx\AppData\Roaming\WeHub\system\log 目录下会产生log文件   
log的配置文件为C:\Users\xxxxx\AppData\Roaming\WeHub\system\cfg\log4cxx.properties,    
若要看很详细的log,请提高loglevel:
用记事本打开log配置文件,将第1行"log4j.rootLogger = DEBUG,logFile"中的"DEBUG" 修改为"TRACE",
将第6行的"log4j.appender.logFile.Threshold = DEBUG"中的"DEBUG" 修改为"TRACE",然后重启wehub.
之后log文件会详细记录wehub 发送出去的http request和回调接口返回的respone
```

通过log文件来排查常见的问题
```
错误1:log中出现  "unknow format reply data,error = xxxxx"
原因: 回调接口收到wehub发送的http request后,返回的http respone不是json格式,wehub无法解析或解析出错

错误2:log中出现"HubLogic OnReplyError, replay error = xxx", xxx的值是下方NetworkError枚举类型的值
原因:回调接口的服务端运行不正常(比如服务没有开启,回调接口的域名无法解析等等)

enum  NetworkError {
        NoError = 0,
// network layer errors [relating to the destination server] (1-99):
        ConnectionRefusedError = 1,
        RemoteHostClosedError,
        HostNotFoundError,
        TimeoutError,
        OperationCanceledError,
        SslHandshakeFailedError,
        TemporaryNetworkFailureError,
        NetworkSessionFailedError,
        BackgroundRequestNotAllowedError,
        TooManyRedirectsError,
        InsecureRedirectError,
        UnknownNetworkError = 99,

        // proxy errors (101-199):
        ProxyConnectionRefusedError = 101,
        ProxyConnectionClosedError,
        ProxyNotFoundError,
        ProxyTimeoutError,
        ProxyAuthenticationRequiredError,
        UnknownProxyError = 199,

        // content errors (201-299):
        ContentAccessDenied = 201,
        ContentOperationNotPermittedError,
        ContentNotFoundError,
        AuthenticationRequiredError,
        ContentReSendError,
        ContentConflictError,
        ContentGoneError,
        UnknownContentError = 299,

        // protocol errors
        ProtocolUnknownError = 301,
        ProtocolInvalidOperationError,
        ProtocolFailure = 399,

        // Server side errors (401-499)
        InternalServerError = 401,
        OperationNotImplementedError,
        ServiceUnavailableError,
        UnknownServerError = 499
    };
```

- 2 wehub和回调接口的交互数据的编码方式

```
wehub 将utf-8编码的json格式的request以post方式发往回调接口(http 的ContentType 为application/json)
回调接口的respone也必须为utf-8的编码的json格式数据,否则wehub无法正确解析
```

- 3.wehub是否可以多开?

```
0.1.4之前的wehub不支持多开微信,之后的wehub已支持多开微信,请升级到最新版的微信.
多开方式:先开启一个wehub和微信并登陆;再开启第二个wehub和微信并登陆;重复以上步骤
如果这个过程中wehub没有自动开启新的微信客户端,手动点击wehub界面上的"打开微信"按钮.
```

- 4.回调接口该如何处理上报的聊天消息?

```
wehub提供基础的微信聊天消息上报的能力,不过滤发送者的wxid(但会过滤公众号推送的内容)
因此回调接口在做自动回复的时候,需要过滤自己的微信号发的消息.否则会陷入"回调接口下发自动 
回复内容--->wehub发消息--->微信消息事件回调--->wehub上报刚才自己发的消息--->回调接口又下发聊天任务"的死循环
```
```
一旦陷入死循环,容易导致wehub高频率地发消息,这极易触发微信系统的安全提醒甚至被封号
如何过滤自己发的消息?
若使用的wehub版本<0.1.4,回调接口在收到report_new_msg时,判断msg中的wxid是否为这个wehub上登陆的wxid. 若是,则不要下发task_type=1的发消息的任务
若使用的wehub版本>=0.1.4,回调接口在收到report_new_msg时,判断msg中的wxid_from是否为这个wehub上登陆的wxid.若是,则不要下发task_type=1的发消息的任务
```

- 5.为什么有时候wehub无法监测到微信?

```
1. 确定当前wehub 支持的微信版本号:
   方法:  打开wehub的安装目录,进入WeChatVersion 子目录,会发现有以微信版本号命名的子文件夹如2.6.3.78,2.6.4.38,2.6.4.56等等,他们分别代表着wehub所支持的微信的版本号. 若你当前正在使用的微信的版本号不在上述之列,则wehub无法支持你当前运行的微信版本. 这时你必须卸载当前的微信,重新安装我们支持的微信版本.
      每当微信出新版本时,我们会第一时间推出新的wehub版本以适配新的微信,请关注我们的更新.

2. 看系统中是否有僵死的微信进程:
   方法: 打开系统的任务管理器,对所有的进程按"名称" 排序查看,查看当前有多少个微信进程(只有名称为Wechat的进程才是微信进程,其他的如WeChatweb, WeChatStore都不是微信的主进程),同时查看你的任务栏里有多少个微信的聊天或登陆窗口. 比如你的任务管理器中显示有4个"Wechat"进程而在任务栏上你只看到了3个微信的窗口,说明有一个微信进程是僵死的.
    若出现这种情况,请杀掉所有的微信进程,然后重新开启wehub登陆微信
    
3. 看系统是否安装了360等安全软件,或者其他的类似于wehub的微信辅助软件.
   360会干扰wehub的正常的行为甚至会删除wehub的某些关键dll文件,其他的微信辅软件可能会和wehub冲突.
   请彻底卸载掉这些软件(保险起见,最好重新安装wehub)
```



- 6.为什么有时候wehub发图片很慢很慢?

```
目前已知的情况是,运行在阿里云 主机上的wehub 发图片很慢很慢(猜测可能微信对运行阿里云的微信客户端有某些特殊的行为处理),更换成腾讯云的主机后一切都正常了
```

- 7.wehub 开启后,appid无法验证成功?
```
   appid验证失败后会弹出设置界面,并用红色文字标识失败的原因. 
  一般 appid验证失败有两种情况:
  1.appid 过期了需要续费, 这种情况请在微信群里资讯我们的工作人员
  2.wehub 发给回调接口的 login 请求没有正确地返回: 
  	可能你的网络问题,可能你的服务器没有正常启动(看wehub的log). 
  	若开启了安全验证,回调接口在处理logIn时可能没有正确地返回签名.
```
