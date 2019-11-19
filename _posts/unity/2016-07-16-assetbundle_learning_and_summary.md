---
title: 我所理解的Unity AssetBundle
date: 2016-07-16 16:44:10
type: "tags"
comments: false
tags: unity
---


本文是学习[Unity官方教程](https://unity3d.com/learn/tutorials/topics/best-practices)、[Unity官方的API](https://docs.unity3d.com/ScriptReference/)和阅读Unity代码的总结

#### AssetBundle基本知识

**1.** AssetBundle由两部分组成:头部(header)数据部分(data segment)

* **头部**保存了版本号、数据类型、<font color=#FF0000>文件信息</font>、是否压缩等这些描述信息。
	* 文件信息记录了数据部分里面的所有单个资源的文件名以及在整个AssetBundle中文件offset和size,通过这个信息可以直接获取到AssetBundle中的某一个文件的数据。从Unity5.3版本开始这部分数据会单独生成一个AssetBundle同名的.manifest文件。
* **数据块**保存了在build AssetBundle时候标记的textures、prefabs、materials等这些资源文件

<!-- more --> 

**2.** 生成的AssetBundle内容文件可选的格式
* 未压缩
* LZMA
* LZ4 (Unity 5.3版本开始支持)

**3.** 项目里面尽量通过AssetBundle的方式来分发资源,不推荐把资源文件全部放到Resources目录中。对比如下:
* 大量的资源放到Resources目录中当要加载其中某些资源的时候会造成更长的搜索资源的时间
* AssetBundle文件可以按需来加载和卸载便于管理内存
* AssetBundle文件可实现在线下载,便于资源更新
	
	
#### 生成AssetBundle文件

生成AssetBundle Unity提供的API如下:

```cs
public static AssetBundleManifest BuildAssetBundles (string outputPath, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform){}
```

其中我们主要看第二个参数BuildAssetBundleOptions,它定义如下:

```cs
public enum BuildAssetBundleOptions
{
	None = 0,
	UncompressedAssetBundle = 1,
	
	...
	
	ChunkBasedCompression = 256,
	StrictMode = 512,
	OmitClassVersions = 1024
}
```

这几种类型说明:

* **ChunkBasedCompression**这个选项在Unity5.3版本开始添加了,它支持把AssetBundle中每个组成对象build为LZ4格式的数据(5.3版本之前是不支持的)。这样的话就带来了一个很大的好处,我们在加载AssetBundle的时候不需要解压整个AssetBundle来读取我们需要加载AssetBundle中的某个对象的数据,可以加快读取Asset的速度同时也节省了很多内存。

* **UncimpressedAssetBundle**这个选项使用为压缩的格式。这样省去了加载AssetBundle或者实例化Asset的时候解压缩的时间,但是build出来的AssetBundle的文件大小会比较大

* **其他选项**使用LZMA格式的压缩格式来生成AssetBundle文件。这种格式其实压缩率是最大的,build出来的AssetBundle文件大小最小,但是这种格式的AssetBundle的读取效率(时间,内存)比不上使用LZ4格式生成的AssetBundle。不过在5.3版本之前只有这一种压缩格式可选

**总结:** 如果使用的Unity是5.3版本或更高级版本,推荐使用ChunkBasedCompression压缩格式


#### 加载AssetBundle对象

在Unity5.3版本Unity对加载AssetBundle对象的API做了修改, 主要有三种方式加载/生成AssetBundle的API(前两种这里省去带crc参数的API):

```cs

//@1
public static AssetBundle LoadFromMemory(byte[] binary);
public static extern AssetBundleCreateRequest LoadFromMemoryAsync (byte[] binary);

//@2
public static AssetBundle LoadFromFile (string path);
public static AssetBundleCreateRequest LoadFromFileAsync (string path);

//@3
public static WWW LoadFromCacheOrDownload(string url, int version, uint crc = 0);

```

##### LoadFromMemory(Async)

这个API在Unity5.3版本名字改为CreateFromMemory。虽然名字改了但是它的功能没有变化,先看下怎么使用(方法直接拷贝[Unity官方的demo](https://docs.unity3d.com/ScriptReference/AssetBundle.CreateFromMemory.html)):

```cs

WWW www = new WWW (url);
yield return www;   

AssetBundleCreateRequest assetBundleCreateRequest = AssetBundle.CreateFromMemory (www.bytes);
yield return assetBundleCreateRequest;

AssetBundle assetBundle = assetBundleCreateRequest.assetBundle;

```

想象一下CreateFromMemory方法接受的参数是一个AssetBundle的原始数据然后返回了一个AssetBundleCreateRequest包装的AssetBundle。那么AssetBundle是直接的数据是成员变量的引用指向www.bytes数据吗(显然这样的话是最省内存的)? 还是会自己拷贝一份www.bytes数据。显然使用成员变量引用指向www.bytes是行不通的,因为www的内存只要一释放那么AssetBundle这个对象肯定就使用不了了。进一步证实我们可以看CreateFromMemory这个方式的实现:

```cs
[WrapperlessIcall]
[MethodImpl(MethodImplOptions.InternalCall)]
public static extern AssetBundleCreateRequest CreateFromMemory(byte[] binary);

```

看到方法很显然它真正调用的是C++层的CreateFromMemory方法,那我们继续看C++层


```cpp
//cpp
CUSTOM static AssetBundleCreateRequest CreateFromMemory(byte[] binary)
{
	if (!binary)
		return SCRIPTING_NULL;

	int dataSize = GetScriptingArraySize(binary);
	UInt8* dataPtr = Scripting::GetScriptingArrayStart<UInt8>(binary);

	AsyncOperation* req = new AssetBundleCreateRequest(dataPtr, dataSize);


	return CreateScriptingObjectFromNativeStruct(MONO_COMMON.assetBundleCreateRequest,req);
}

```

binary(也就是www.bytes数据)数据最终被C++层的AssetBundleCreateRequest对象包裹,然后再通过CreateScriptingObjectFromNativeStruct方法转换成Mono对象返回给了Mono层。在继续跟踪binary数据的去向的时候我们发现最终Unity通过这个数据创建了一个UnityWebStream对象,而这个对象是最为AssetBundleCreateRequest的成员变量存在的。在创建这个UnityWebStream对象的过程中Unity终于使用了binary数据，首先解析了头部数据然后再把数据部分(data segment)通过memorycpy拷贝到了UnityWebStream对象在非托管内存中申请的内存空间上,而且如果数据是LZMA格式的话,Unity还会对其进行解压。

```cpp
//cpp
while(bytesCopied < size)
{
	...
	
	int bytesToCopy = std::min<int>(size - bytesCopied, kDownloadBufferSize - buf.bytesUsed);
	memcpy(buf.mem + buf.bytesUsed, (UInt8*)data + bytesCopied, bytesToCopy);
	buf.bytesUsed += bytesToCopy;
	
	...
}

```

上面data数据就是我们传入的binary数据。进过上面分析我们可以确认使用CreateFromMemory这个方法来创建AssetBundle肯定会消耗两倍与AssetBundle数据的内存空间。

##### LoadFromFile

这个API在Unity5.3之前的版本方法名为CreateFromFile,同样这个API的功能并没有变。这个API支持读取未压缩的和LZ4格式的AssetBundle文件,但是LZMA格式文件经过加压之后任然可以读取。官方文档中有一句话是这样描述的:

```
This is the fastest way to load an AssetBundle
```

使用LoadFromFile方法的时候在手机或者Pad设备上的加载AssetBundle的时候仅仅加载AssetBundle的头信息,至于AssetBundle的数据部分(data segment)并不会加载到内存中,然后当我们使用Load(Async)来加载AssetBundle中的某一个Asset的时候找到这个Asset对应在AssetBundle文件中的位置并仅读取这部分数据。这样的话就可以节省非常多的内存空间加载的速度自然会变快了。

##### LoadFromCacheOrDownload

使用这种加载的话会缓存一份解压之后的AssetBundle数据缓存到本地(不管是从网络下载还是加载本地的AssetBundle都会缓存)。当第二次加载的时候在版本号不变的情况下LoadFromCacheOrDownload会加载本地的缓存，但是只会读取BundleHeader和FullHeader数据。iOS平台下AssetBundle缓存的路径:

```cs
application.persistentdatapath + "../Library/UnityCache/Shared/*"
```

当存在了缓存文件之后再次使用LoadFromCacheOrDownload方法加载的时候它的工作方法其实和LoadFromFile的工作方式一样了,它只会把AssetBundle数据的头部信息加载到内存中,等我们要加载AssetBundle中的某个Asset的时候才会去磁盘上读取对应的部分的数据。那么它和LoadFromFile的不同之处在哪里呢?不同之处就在于这是一个WWW的方法。和LoadFromMemory一样它的原始数据来自WWW, WWW在加载这个原始数据时同样会拷贝一份数据在非托管的内存空间上。这样的话就会造成两倍的内存开销,不过幸好LoadFromCacheOrDownload第一次加载会缓存到本地之后就没有这个额外的内存消耗了。

#####总结

如果使用的Unity是5.3之前的版本那么最好使用LoadFromCacheOrDownload来加载AssetBundle。但是如果使用的是5.3版本及高版本则最好使用LoadFromFile(Async)这个方法来加载AssetBundle。

#### 卸载AssetBundle对象

##### 卸载的方法
AssetBundle有一个Unload的成员方法来卸载AssetBundle资源,首先我们看下Unload的方法声明:

```
public extern void Unload (bool unloadAllLoadedObjects);
```

Unload方法带有一个bool类型的unloadAllLoadedObjects参数,这儿参数的作用Unity官方的API也有详细说明。当unloadAllLoadedObjects为true的时候所有的从AssetBundle中加载的对象和AssetBundle本身将会被卸载。

##### 实例化的资源清理问题
当unloadAllLoadedObjects为false的时候只会卸载AssetBundle本身,从AssetBundle中实例化的对象是不会卸载的。这些已经实例化的对象在依赖的AssetBundle被Unload(false)卸载的时候它们之间引用也就断了,这个时候如果Load之前被卸载的那个AssetBundle的重新实例化里面的对象和之前的对象就没有任何关系了,就算之前的对象还存在于内存当中。举个例子:现在有AssetBundle文件A,A里面包含了一个material m。然后我们进行一下步骤:

* Load AssetBundle A得到对象A1,从A1中实例化m得到m1
* 通过Unload(false)卸载A1 (此时m1还在内存当中)
* 再次Load AssetBundle A得到对象A2,从A2中实例化m。此时会得到新的一个实例化对象m2就算m1还是内存当中,因为此时m1和A2没有任何关系。那么m1这个对象就需要我们自己去销毁了。

从上面例子我们可以看出使用Unload(false)卸载AssetBundle的话很容易造成内存当中同一个Asset会有几份拷贝,这样会浪费一定的内存。

##### 总结
推荐使用Unload(true)来卸载AssetBundle