# m3u8 视频提取工具（下载web中的ts文件）

### 开发背景

- m3u8视频格式简介（借用他人描述）
  - m3u8视频格式原理：将完整的视频拆分成多个 .ts 视频碎片，.m3u8 文件详细记录每个视频片段的地址。
  - 视频播放时，会先读取 .m3u8 文件，再逐个下载播放 .ts 视频片段。
  - 常用于直播业务，也常用该方法规避视频窃取的风险。加大视频窃取难度。
- 鉴于 m3u8 以上特点，无法简单通过视频链接下载，需使用特定下载软件。（借用他人描述）
  - 但软件下载过程繁琐，试错成本高。
  - 使用软件的下载情况不稳定，常出现浏览器正常播放，但软件下载速度慢，甚至无法正常下载的情况。
  - 软件被编译打包，无法了解内部运行机制，不清楚里面到底发生了什么。

- 搜索网上并不能完全符合我的功能，有时需要解密对应M3U8视频文件的C#代码 ，固写了对应的C#简易代码。

### 工具特点

* 利用C# 语言开发 类库的方式支持调用
* 参数简单 只需要提供对应的 M3u8文件下载地址 和对应文件的存放路径就会自动下载
* 支持单和多线程的自主选择 同时会对下载失败的文件自动重新下载一次

### 功能说明

​	输入对应M3U8文件的下载地址和保存的文件夹路径，就会开始下载

### 使用说明

开发者可利用相关代码或手工获取到对应的M3U8📃的下载地址，创建TsHelper对象，调用ReuqestAndSave（）方法。即可完成下载。下载的日志可以通过打开DebugView查看。

```c#
TsHelper tsHelper = new TsHelper(url,@"C:\temp\存放所有视频的base文件夹","我的测试视频1文件夹");
bool flag = tsHelper.ReuqestAndSave();
```

### 实现思路

⏬下载解析M3U8文件：

1.首先通过给定url请求对应的m3u8文件-->结果写入M3U8Response。

2.请求后的结果可能会得到多码率的m3u8文件的新的指向，这里默认请求第一个码率的url，然后请求结果覆盖M3U8Response。

3.对请求的结果M3U8Response进行解析：

> 3.1 有ASE-128 加密 --》请求加密的Key文件获取密钥
>
> 3.2无加密 

4.然后请求m3u8文件给出的所有的ts文件，下载结束后判读加密🔐，如果有加密就通过DealTS()这个函数，对下载的字节流解码。

5.保存所有的ts文件，最后生成一个新的m3u8文件在ts的目录下面名为：\__main__.m3u8

### 核心代码如下：

```csharp
/// <summary>
/// 请求m3u8地址
/// </summary>
private void RequestM3U8()
{
	M3U8Response = RequestM3U8(this.M3U8Url);
}

/// <summary>
/// 解析m3u8 并保存成自己的
/// </summary>
private void AnalysisM3U8()
{
  //判断是否  多码率适配流 并重新请求
  IsMutiple();

  //如果需要自己重新加密的 那么需要生成对应的key
  if (outFileConfig.saveAsEncrpy) GenerateAESKey();

  Logger.LogDebug("分解出所有的ts文件" + M3U8Response);
  //分解出所有的ts文件
  SpliteM3U8(M3U8Response);

  //检测视频 AES 加密
  if (M3U8Response.IndexOf("#EXT-X-KEY") != -1)
  {
    //.*METHOD=([^,]+) #EXT-X-KEY:METHOD=AES-128,
    var method = ReGeXHelper.GetRes(M3U8Response, @".*METHOD=([^,]+)");
    aesConf.method = method != null ? method.Groups[1].Value : "";

    //.*URI="([^"]+)  URI="key.key" 
    var uri = ReGeXHelper.GetRes(M3U8Response, @".*URI=""([^""]+)");
    aesConf.uri = uri != null ? uri.Groups[1].Value : "";

    //.*IV=([^,]+) 
    var iv = ReGeXHelper.GetRes(M3U8Response, @".*IV=([^,]+)");
    aesConf.iv = iv != null ? iv.Groups[1].Value : "";

    aesConf.uri = ApplyURL(aesConf.uri, M3U8Url);

    Logger.LogDebug("解密" + aesConf.uri);
    GetAES();
  }
  else
  {
    DownloadTS();
  }
  Logger.LogDebug("保存新的m3u8");
  OutSaveEnd();
}
```

一个是请求m3u8文件的函数，一个是分析请求后的响应函数

#### End：

* 可能代码不规范，写的复杂，不易理解。各位学长学姐讲武德啊。



