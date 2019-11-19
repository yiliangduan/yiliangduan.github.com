---
title: Unity文件下载分析
date: 2016-12-28 23:39
type: "tags"
tags: unity
comments: false
---

在项目里面有时需要从服务器上下载比较大的文件，比如我们在做更新的时候需要下载AssetBundle文件。Unity3D中提供了WWW接口可以用来下载文件，但是使用之后发现这个接口在下载比较大的文件的时候非常占用内存资源。然后我用C#的Webclient接口来下载同样的文件，发现内存占用情况存在和WWW的完全不一样。本文来就来探讨一下使用WWW和Webclient两个接口下载文件造成内存占用的原因。

<!-- more --> 
### 测试

>首先我们通过下面一个简单的demo来测试下分别使用WWW和Webclient两个API来实际下载下载文件内存使用情况。在demo中我们分别测试了两个下载接口下载文件到内存和下载文件到本地存储卡的情况。([全部代码及完整内存截图](https://pan.baidu.com/s/1qYq8xtE) 密码n3pv)

```cs
...

public class Test : MonoBehaviour {

	//50.9M
	private const string uri = "http://sw.bos.baidu.com/sw-search-sp/software/98087379aca67/QQMusic_12.97.3627.1201.exe";

	private HttpDataDownLoader downloader;

	...

	//@1 直接下载文件到内存中
	private IEnumerator DownloadByWWWInternal()
	{
		Debug.Log("Start DownloadByWWWInternal");

		WWW www = new WWW(uri);

		yield return www;

		Debug.Log("download completed& www.error: " + www.error);
	}

	//@2 下载数据到内存中然后保存到保存到本地(WWW下载文件必须全部下载下来才能获取下载的数据)
	private IEnumerator DownloadByWWWFileInternal()
	{
		Debug.Log("Start DownloadByWWWFile");

		WWW www = new WWW(uri);

		yield return www;

		Debug.Log("download completed& www.error: " + www.error);

		File.WriteAllBytes(Application.persistentDataPath + "/test.exe", www.bytes);

		www.Dispose();

		Debug.Log("write complete!");
	}

	//@3 下载文件到内存中(HttpDataDownLoader是对Webclient的一个非常小的封装，具体可以打开全部代码查看)
	public void DownloadByWebclient()
	{
		Debug.Log("Start DownloadByWebclient");

		downloader = new HttpDataDownLoader(uri, 100000, false);
	}

	//@4 下载文件到本地，不保存到内存中
	public void DownloadByWebclientSaveData()
	{
		Debug.Log("Start DownloadByWebclientFile");

		Debug.Log(Application.dataPath);

		downloader = new HttpDataDownLoader(uri, 100000, true, Application.persistentDataPath + "/test2.exe");
	}
}

```

下面是demo分别测试四种不同下载方式的内存消耗情况:
>测试环境: Unity3D 5.4.1f1 macOS 10.12.1 XCode 8.0


![](/images/unity_download/1.png)

---
### 分析

观察上面实际测的运行内存情况可以产生一个疑问:
* 为什么使用WWW下载(@1@2)则是内存慢慢增长的?
* 为什么使用WWW下载文件并保存到本地这种方式(@2)会出现内存突然增长大概一倍的情况?
* 为什么使用Webclient方法下载文件到本地(@4)内存暂用如此之小?(下载的目标文件大小为50.9M)

接下来我们分别对WWW和Webclient进行分析，来得出上面三个问题得答案:


#### WWW

WWW创建对象之后我们就是可根据对象的IsDone属性来判断是否下载完成，那么我们看看WWW内部到底是怎样下载:

在C#层一般情况下我们是这样来创建一个WWW对象来进行下载的:

```
WWW www = new WWW(uri);
```

通过观察WWW类C#代码我们可发现所有的WWW构造函数最终都会去调用 **InitWWW** 这个函数:

```
public WWW (string url)
{
	this.InitWWW (url, null, null);
}

[WrapperlessIcall]
[MethodImpl (MethodImplOptions.InternalCall)]
public extern void InitWWW (string url, byte[] postData, string[] iHeaders);

```
而InitWWW是绑定的一个C++的函数,那么看看C++的实现部分:

```
	CUSTOM void InitWWW( string url , byte[] postData, string[] iHeaders ) {
		...
		
		WWW* www = WWW::Create (cpp_string.c_str(), rawPostDataPtr, rawPostDataLength, headers);
		...
	}
```

在C++层的InitWWW函数中C++层的WWW类会根据平台来创建一个相应的WWW派生类对象,由于我们现在测试的是iOS环境所以我们只看iOS环境的代码:

```cpp
static WWW* CreatePlatformWWWBackend (const char* url, const char* postDataPtr, int postDataLength, const WWW::WWWHeaders& headers, bool cached, int cacheVersion, UInt32 crc )
{
	...
	#elif UNITY_IPHONE
	return new iPhoneWWW(url, postDataPtr, postDataLength, headers, cached, cacheVersion, crc);
	...
}
```

iPhoneWWW是一个Objective-c类，这个类在构造方法调用之后会调用iOS的系统接口 **NSURLConnection** 来建立网络连接然后开始下载文件。我们重点看下载过程中接受数据的方法，这个方法可以让我们看到下载的文件数据的保存情况:

```oc
- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)ldata
{
	...

	int bytes = [ldata length];

	if(size+bytes > alloc_size)
	{
		const UInt32 estimatedDownloadSize = www->GetEstimatedDownloadSize();
		if (alloc_size+bytes <= estimatedDownloadSize)
			alloc_size = estimatedDownloadSize;
		else
			alloc_size = (alloc_size * 1.5) + bytes;
		data = (UInt8*)realloc((void*)data, alloc_size);
		if (!data)
		{
			ErrorString("WWW: out of memory");
			[connection cancel];
			return; // this will stop the transfer - not all data is loaded
		}
	}

	[ldata getBytes:data+size length:bytes];
	size += bytes;

	...
}
```
这个函数我们可以看到iPhoneWWW首先会分配一个预估大小的数组data来存储下载的数据，connection这个方法是在下载过程中多次回调的，每隔一段时间会接受一部分数据，然后方法里面判断data的存储大小是否足够，如果不够则重新分配之前的1.5倍大小内存空间。接着把新下载的ldata数据存到data里面。也就是说直到文件下载结束data会存储下载的文件的全部数据，但是data得大小和文件的数据的大小不是相同的(存在相同的可能)。这也是为什么@1和@2方法下载过程中所消耗的内存慢慢的增长到近似文件大小的内存值了。

WWW解析到这里还没有解释一个问题，那就是为什么@2方法下载过程中那个内存的峰值是怎么出现的。再来回顾我们的demo的这一段保存下载的数据到本地代码:

```
File.WriteAllBytes(Application.persistentDataPath + "/test.exe", www.bytes);
```

答案就在这里,细心的人很快会产生一个疑问，iPhoneWWW下载的数据保存到了OC分配的内存空间中，那么我们这里在C#中通过 www.bytes 怎样获取到的下载数据的呢？那就来看下这个方法的实现吧


```
CUSTOM_PROP byte[] bytes {
	...

	return CreateScriptingArray<UInt8>( www.GetData(), www.GetSize(), GetScriptingManager().GetCommonClasses().byte );
}

ScriptingArrayPtr CreateScriptingArray (ScriptingClassPtr klass, int count)
{
	...
	return mono_array_new(mono_domain_get(),klass,count);
	...
}

```
看到了么？当我们在C#层调用WWW的bytes属性的时候会重新再C#(或者说Mono)层重新分配一个和OC层保存的下载数据同样大小的内存空间，然后会把下载的数据拷贝到这个Mono层的数组中，此时此刻WWW消耗的内存突然增加了一倍，然后我们把这部分数据通过File.WriteAllBytes写入到文件中然后我们调用了WWW.Dispose方法内存就降下来了。这就是我们为什么会看到那个内存突然出现一个峰值的原因了。

#### Webclient

分析完了WWW我们再来分析下Webclient，Webclient到底怎样实现的可以让下载文件到本地只消耗如此之小的内存，那我们来看看Webclient的下载文件的方法DownloadFileAysnc，Webclient的实现要比WWW看起来要稍微轻松一点。

```
public void DownloadFileAsync (Uri address, string fileName, object userToken)
{
	...
	DownloadFileCore ((Uri) args [0], (string) args [1], args [2]);
						OnDownloadFileCompleted (
							new AsyncCompletedEventArgs (null, false, args [2]));
	...
}

void DownloadFileCore (Uri address, string fileName, object userToken)
{
	WebRequest request = null;
			
	using (FileStream f = new FileStream (fileName, FileMode.Create)) {
	
	...
		request = SetupRequest (address);
		WebResponse response = request.GetResponse ();
		Stream st = ProcessResponse (response);
					
		int cLength = (int) response.ContentLength;
		int length = (cLength <= -1 || cLength > 32*1024) ? 32*1024 : cLength;
		byte [] buffer = new byte [length];
					
		int nread = 0;
		
		...				
		while ((nread = st.Read (buffer, 0, length)) != 0){
		...
				f.Write (buffer, 0, nread);
			}
		...
	}
}

```
为了看不来不那么累，我尽量只保留有用的代码，其他的都省略。看上面两个方法，其实就已经有答案了。Webclient下载文件其实和WWW下载文件一个道理，不同的时Webclient每接受32k大小的数据之后就直接写入到了文件当中去，这部分内存数据并不常驻内存而是随着方法调用结束会被C#的GC自动回收掉。这就是为什么Webclient下载文件保存到本地内存可以消耗这么低的原因了。

### 总结
在Unity中如果是需要把文件下载在本地则建议使用Webclient来下载,如果是单纯的下载到内存当中则两个接口都可以使用