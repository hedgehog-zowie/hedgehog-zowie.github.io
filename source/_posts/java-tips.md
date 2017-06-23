---
title: java-tips
date: 2016-09-23 08:50:52
toc: true
tags:
- java
categories:
- java
---

记录使用js的过程中一些技巧、方法、注意事项。

# java线程个数的确定

[参考链接](http://bbs.csdn.net/topics/390940782)

## 确定最佳线程数量
首先确定应用是CPU密集型 （例如分词，加密等），还是耗时io（ 网络，文件操作等）
CPU密集型：： 最佳线程数等于cpu核心数或稍微小于cpu核心数。。。具体数值要以jvm图形线程监控显示繁忙情况为依据。。

耗时io型：： 最佳线程数一般会大于cpu核心数很多倍。。一般是io设备延时除以cpu处理延时，得到一个倍数，我的经验数值是20--50倍*cpu核心数，，具体数值也是要以jvm图形线程监控显示繁忙情况为依据。。保证线程空闲可以衔接上。。。

`对于耗时io型，一个简单的算法：：最佳线程数==单个线程的黄色时间块长度（空闲） / 绿色时间块长度（繁忙） * cpu核心数`

`线程图形化监控工具:: 可以用jprofile  ，，，磁盘队列图形化监控工具：：：任务管理器》》资源监视器》》磁盘队列深度`

最佳线程数量也与机器配置（内存，磁盘速度）有关，如果cpu，内存，磁盘任何一个达到顶点，就需要适当减少线程数。。

## 使用多线程的原因

1.防止界面卡死.
提高用户的用户体验
对单核CPU，对客户端软件，采用多线程，主要是 创建多线程将一些计算放在后台执行，而不影响用户交互操作。（用户界面 & 其他计算 并行进行）提高用户的操作性能！

2.耗时的操作(io,网络io等)使用线程，提高cpu使用率..
I/O操作不仅包括了直接的文件、网络的读写，还包括数据库操作、Web Service、HttpRequest以及.net Remoting等跨进程的调用。
要是不使用多线程,你回发现cpu使用率很空闲..

3．多CPU(核心)中，使用线程提高CPU利用率
 使多CPU系统更加有效
操作系统会保证当线程数不大于CPU数目时，不同的线程运行于不同的CPU上。
要是不使用多线程,你回发现仅仅一个cpu很忙碌的,其他cpu使用率很空闲..

## 不适用多线程的情况,

1.你的代码是cpu密集型,在单核cpu上..
2.单核cpu上,线程的使用（滥用）会给系统带来上下文切换的额外负担。并且线程间的共享变量可能造成死锁的出现。
3.当需要执行I/O操作时，使用异步操作常常比使用线程+同步I/O操作更合适。

#　HttpClient连接池的使用

[参考链接：使用httpclient必须知道的参数设置及代码写法、存在的风险](http://jinnianshilongnian.iteye.com/blog/2089792)

详细代码：

```
public class PostUtil {

    /**
     * 日志记录器
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(PostUtil.class);

    /**
     * 连接池
     */
    private PoolingHttpClientConnectionManager connectionManager = null;
    /**
     * http客户端建造器
     */
    private HttpClientBuilder httpClientBuilder = null;
    /**
     * 请求配置
     */
    private RequestConfig requestConfig = null;

    /**
     * socket超时
     */
    private static final int SOCKET_TIME_OUT = 5000;
    /**
     * 连接超时
     */
    private static final int CONNECTION_TIME_OUT = 5000;
    /**
     * 请求超时
     */
    private static final int REQUEST_TIME_OUT = 5000;

    /**
     *
     * @param maxConnection 最大连接数
     */
    public PostUtil(final int maxConnection) {
        connectionManager = new PoolingHttpClientConnectionManager();
        // MaxTotal是整个连接池的最大连接数
        connectionManager.setMaxTotal(maxConnection);
        // DefaultMaxPerRoute是根据连接到的主机对MaxTotal的一个细分；比如：
        // MaxtTotal=400 DefaultMaxPerRoute=200
        // 当只连接到http://baidu.com时，到这个主机的并发最多只有200；而不是400；
        // 当连接到http://baidu.com 和 http://qq.com时，到每个主机的并发最多只有200；即加起来是400（但不能超过400）；所以起作用的设置是DefaultMaxPerRoute。
        connectionManager.setDefaultMaxPerRoute(maxConnection);

        // 为指定站点设置最大连接数
//        HttpHost target = new HttpHost(IP, PORT);
//        connectionManager.setMaxPerRoute(new HttpRoute(target), 20);

        httpClientBuilder = HttpClients.custom();
        httpClientBuilder.setConnectionManager(connectionManager);

        // 设置http的状态参数
        requestConfig = RequestConfig.custom()
                .setSocketTimeout(SOCKET_TIME_OUT)
                .setConnectTimeout(CONNECTION_TIME_OUT)
                .setConnectionRequestTimeout(REQUEST_TIME_OUT)
                .build();
    }

    /**
     * post方式请求接口
     *
     * @param url    请求URL
     * @param params 请求参数
     * @return 服务器返回的字符串
     * @throws Exception 抛出异常不处理，由调用者自行处理异常
     */
    public String post(final String url, final List<NameValuePair> params) throws Exception {
        HttpClient httpClient = httpClientBuilder.build();

        HttpPost request = new HttpPost(url);
        request.setConfig(requestConfig);

        HttpEntity httpEntity = new UrlEncodedFormEntity(params, "UTF-8");
        request.setEntity(httpEntity);

        HttpResponse response;
        response = httpClient.execute(request);
        String strResult;
        // 请求发送成功，并得到响应
        if (response.getStatusLine().getStatusCode() == HttpStatus.SC_OK) {
            // 读取服务器返回过来的json字符串数据
            strResult = EntityUtils.toString(response.getEntity(), Charsets.UTF_8);
            LOGGER.info("response = {}", strResult);
            ReturnResult returnResult = JsonUtil.fromJson(strResult, ReturnResult.class);
            if (!"0".equals(returnResult.getCode())) {
                throw new RuntimeException("数据上报接口返回失败");
            }
        } else {
            strResult = EntityUtils.toString(response.getEntity());
            throw new RuntimeException("数据上报接口异常，" + strResult);
        }
        return strResult;
    }

}
```
