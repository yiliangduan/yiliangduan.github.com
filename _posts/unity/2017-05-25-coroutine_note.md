---
title: 'Unity Coroutine的实现原理'
date: 2017-05-25 16:47:36
type: "tags"
tags: csharp
comments: false
---


在Unity中我经常用到Coroutine的功能，但是对于Coroutine一直有一些疑问没有得到答案，下面先上一个在项目里面经常使用Coroutine的场景的sample:

```cs
//TestCoroutine.cs

public class TestCoroutine : MonoBehaviour {

    private void Start () 
    {
        StartCoroutine(Test()); 
    }

    private IEnumerator Test()
    {
        Debug.Log(gameObject.name);

        yield return new WaitForSeconds(5);

        Debug.Log(transform.localPosition);

        transform.localScale = Vector3.one;
    }
}

```

* Test中yield语句调用之后为什么可以立即停止，等到 *WaitForSeconds* 的时候接着执行，这是怎么做到的？

* Test的返回值是IEnumerator，但是函数内部的return用了yield来修饰，yield到底做了什么工作。


针对第二个问题我找到了一个比较详细的[解释](https://stackoverflow.com/a/39496), 里面有这样一段话:

> The yield keyword actually does quite a lot here. The function returns an object that implements the IEnumerable interface.

相当于yield关键字会自动生成一个集成IEnumerable的对象,那么这个对象到底是什么样子呢? 首先使用ILSpy工具查看下这段代码代码的C#版本:

```
[DebuggerHidden]
private IEnumerator Test()
{
	TestCoroutine.<Test>c__Iterator0 <Test>c__Iterator = new TestCoroutine.<Test>c__Iterator0();
	<Test>c__Iterator.$this = this;
	return <Test>c__Iterator;
}

```

可以看到通过ILSpy解析之后Test方法已经完全变了，但是也验证了上面的说法。yield关键字会自动转换成一个IEnumerable的对象，这里自动生成了一个名字为TestCoroutine.{Test}c_Iterator0 的类，并且创建了这个类型的 {Test}c_Iterator 对象。

```
代码里面名字是TestCoroutine.<Test>c_Iterator0,但是当前主题<>符号显示不正确，所以改成{}显示。下同
```

那这个对象内部是怎样实现的呢？现在我们得看下TestCoroutine.{Test}c_Iterator0的实现了(这里方法自动加了DebuggerHidden属性，所以只能看IL版本的代码)

```
//TestCoroutine.IL
//整个类的代码比较多，这里只截取我们需要的代码

.class public auto ansi beforefieldinit Test.TestCoroutine
	extends [UnityEngine]UnityEngine.MonoBehaviour
{
	// 嵌套类型
	.class nested private auto ansi sealed beforefieldinit '<Test>c__Iterator0'
		extends [mscorlib]System.Object
		implements [mscorlib]System.Collections.IEnumerator,
		           [mscorlib]System.IDisposable,
		           class [mscorlib]System.Collections.Generic.IEnumerator`1<object>
	{
		.custom instance void [mscorlib]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor() = (
			01 00 00 00
		)
		// 成员
		.field assembly class Test.TestCoroutine $this
		.field assembly object $current
		.field assembly bool $disposing
		.field assembly int32 $PC

		.method public final hidebysig newslot virtual 
			instance bool MoveNext () cil managed 
		{
			// 方法起始 RVA 地址 0x3c7a8
			// 方法起始地址（相对于文件绝对值：0x3aba8）
			// 代码长度 144 (0x90)
			.maxstack 2
			.locals init (
				[0] uint32
			)

			// 0x3ABB4: 02
			IL_0000: ldarg.0
			// 0x3ABB5: 7B C9 07 00 04
			IL_0001: ldfld int32 Test.TestCoroutine/'<Test>c__Iterator0'::$PC
			// 0x3ABBA: 0A
			IL_0006: stloc.0
			// 0x3ABBB: 02
			IL_0007: ldarg.0
			// 0x3ABBC: 15
			IL_0008: ldc.i4.m1
			// 0x3ABBD: 7D C9 07 00 04
			IL_0009: stfld int32 Test.TestCoroutine/'<Test>c__Iterator0'::$PC
			// 0x3ABC2: 06
			IL_000e: ldloc.0
			// 0x3ABC3: 45 02 00 00 00 05 00 00 00 3A 00 00 00
			IL_000f: switch (IL_0021, IL_0056) // 分支，直接执行IL_0021(对应我们cs代码的WaitForSeconds之前的代码，IL_0056对应我们WaitForSeconds之后的代码)

			// 0x3ABD0: 38 6B 00 00 00
			IL_001c: br IL_008c

			// 0x3ABD5: 00
			IL_0021: nop
			// 0x3ABD6: 02
			IL_0022: ldarg.0
			// 0x3ABD7: 7B C6 07 00 04
			IL_0023: ldfld class Test.TestCoroutine Test.TestCoroutine/'<Test>c__Iterator0'::$this
		
			//这里做了 Debug.Log(gameObject.name);
		
			// 0x3ABEB: 02
			IL_0037: ldarg.0
			// 0x3ABEC: 73 1B 02 00 0A
			IL_0038: newobj instance void [UnityEngine]UnityEngine.WaitForSeconds::.ctor(float32) //创建一个WaitForSeconds对象，引用会存放在栈顶。
			// 0x3ABF1: 7D C7 07 00 04
			IL_003d: stfld object Test.TestCoroutine/'<Test>c__Iterator0'::$current // 将栈顶的对象(WaitForSeconds)赋值给current变量
			// 0x3ABF6: 02
			IL_0042: ldarg.0 //载入第0个参数
			// 0x3ABF7: 7B C8 07 00 04
			IL_0043: ldfld bool Test.TestCoroutine/'<Test>c__Iterator0'::$disposing
			// 0x3ABFC: 2D 07
			IL_0048: brtrue.s IL_0051 //判断成员变量disposing是否为true，非空或者非0，如果是则跳转到IL_0051的地址

			// 0x3ABFE: 02
			IL_004a: ldarg.0
			// 0x3ABFF: 17
			IL_004b: ldc.i4.1 //载入值1入栈
			// 0x3AC00: 7D C9 07 00 04
			//把栈顶的值(现在是1)赋值给成员变量PC，这样的话下次调用MoveNext,在IL_000f处的switch的分支就会直接走IL_0056了
			IL_004c: stfld int32 Test.TestCoroutine/'<Test>c__Iterator0'::$PC 

			// 0x3AC05: 38 38 00 00 00
			IL_0051: br IL_008e //IL_008e在当前(MoveNext)函数结尾

			// 0x3AC0A: 02
			IL_0056: ldarg.0
			// 0x3AC0B: 7B C6 07 00 04
			IL_0057: ldfld class Test.TestCoroutine Test.TestCoroutine/'<Test>c__Iterator0'::$this
			// 0x3AC10: 28 4E 01 00 0A
			
			//这里做了Debug.Log(transform.localPosition);

        	//       transform.localScale = Vector3.one;
			
			// 0x3AC39: 02
			IL_0085: ldarg.0
			// 0x3AC3A: 15
			IL_0086: ldc.i4.m1
			// 0x3AC3B: 7D C9 07 00 04
			IL_0087: stfld int32 Test.TestCoroutine/'<Test>c__Iterator0'::$PC

			// 0x3AC40: 16
			IL_008c: ldc.i4.0
			// 0x3AC41: 2A
			IL_008d: ret

			// 0x3AC42: 17
			IL_008e: ldc.i4.1 
			// 0x3AC43: 2A
			IL_008f: ret
		} // 方法 '<Test>c__Iterator0'::MoveNext 结束
}

```

这里只截取了MoveNext方法，因为这个方法里面包含了我们Sample中的Test函数的所有逻辑操作。这段代码我加了注释，通过看代码基本上已经解决了我的第一个疑惑了,下面再做一下分析。Sample中的方法Test在编译之后会转换成大概这个样子(只是模拟):

```
private sealed class <Test>c__Iterator0 : IEnumerator, System.IDisposable
{
	// 用来保存 创建的WaitForSeconds对象的，调用者会根据这个对象的条件是否满足来确定调用MoveNext
	private object current;
	
	// 判断宿主对象是否已经被销毁了,这里就是那个发起StartCoroutine的那个MonoBehaviour对象 
	private bool disposing; 
	
	//保存MoveNext中调用的地址的,用这个来判断是执行yield之前的代码还是之后的代码
	private int PC; 
	
	private bool MoveNext()
	{
		 switch(this.PC)
		 {
		 case IL_0021:
			{
				Debug.Log(gameObject.name);

   		     	this.current = new WaitForSeconds();
        	
	        	PC = IL_0056;
	        	
	        	return true;
			}
   
		 case IL_0056:
			{
				Debug.Log(transform.localPosition);

				transform.localScale = Vector3.one;
			}
		 }
		  
        return false;
	} 
	
	object IEnumerator<object>.Current
   {
		get
		{
			return this.current;
		}
	}
}

```

到这里已经知道了yield关键字所做的操作了，那么现在来看看这个{Test}c_Iterator0对象是怎样被调用的。

首先我们看到我们的Sample中的代码:

```
StartCoroutine(Test()); 
```

这说明创建的{Test}c_Iterator0对象传入的StartCoroutine方法中了，浏览了StartCoroutine的实现，这个方法直接绑定的C++层MonoBehaviour的StartCoroutine方法，这里简述下里面的实现(不贴Unity代码了):

* 调用StartCoroutine方法，该方法会以传入的IEnumerator参数(这里是{Test}c_Iterator0对象)创建一个C++的Coroutine对象
* 这个对象保存会保存参数IEnumerator对象，并且会先获取出IEnuerator的MoveNext和Current方法。这两个方法也是IEunerator最关键的方法
* 创建好之后这个Coroutine对象会保存在MonoBehaviour一个成员变量List中，这样使得MonoBehaviour具备StopCoroutine功能，StopCoroutine能够找到对应Coroutine并停止
* Coroutine对象会调用成员方法run，启动这个Coroutine

这个步骤大概是这样的(代码只表现大概逻辑，不能正确执行):

```
// C#层IEnumerator被传入C++层的时候会被C++的类ScriptingObjectPtr封装并绑定
ScriptingObjectPtr MonoBehaviour::StartCoroutine(ScriptingObjectPtr userCoroutine)
{
	Coroutine* coroutine = new Coroutine ();
	
	//获取C#层对应的方法，拿到方法的地址。通过Mono就可以直接调用了
	ScriptingMethodPtr moveNext =  scripting_object_get_virtual_method(userCoroutine, MONO_COMMON.IEnumerator_MoveNext, GetScriptingMethodRegistry());
	ScriptingMethodPtr current = scripting_object_get_virtual_method(userCoroutine, MONO_COMMON.IEnumerator_Current, GetScriptingMethodRegistry());
	
	...
	coroutine->SetMoveNextMethod(moveNext);
	coroutine->SetCurrentMethod(current);
	
	coroutine->m_Behaviour = this;
	...
	
	m_ActiveCoroutines.push_back(coroutine); 
	
	coroutine.Run();
	
}

```

那么现在生成了C++层的Coroutine对象了，再分析Coroutine现在作为执行者它是怎么实现的。Coroutine里面针对yield对象(Sample里面是WaitForSeconds)类型做了处理:

* WaitForSeconds
* WaitForFixedUpdate
* WaitForEndOfFrame
* Coroutine (C#层)
* WWW
* AsyncOperation

这些类型处理的方式是定义了一个类似定时调用的管理类DelayedCallManager。比如我的条件是WaitForSceonds(5), 那么Coroutine里面会创建一个CallDelayed，把时间设置魏5秒，然后DelayedCallManager的Update里面会直接算时间，到时间了就会回调Coroutine。WWW类型比较特殊它本身做了类似的处理，它提供了一个方法CallWhenDone，当它完成的时候直接回调Coroutine。

这个步骤大概是这样的(代码只表现大概逻辑，不能正确执行):

```
//Coroutine.cpp

// Run其实是一个递归的操作，在CallDelayed之后回调coroutineCallback里面会根据条件再次进入Run，
// 直到MoveNext返回false，条件结束了。
void Coroutine::Run (...)
{
	//根据IEnumerator的特性，首先得调用下MoveNext，所以这里进入Run之后首先会调用一次MoveNext(), 
	//就是<Test>c__Iterator0的MoveNext。这样就给current赋值了
	ScriptingInvocation invocation(MoveNext);
	...
 	ScriptingObjectPtr monoWait = invocation.Invoke(&exception)
	
	//调用Current
	ScriptingInvocation invocation(Current);
	...
 	ScriptingObjectPtr monoWait = invocation.Invoke(&exception);
 	
 	
	//可以在Sample里面看到Current函数在调用过第一次MoveNext之后被赋值为WaitForSeconds对象，
	//所以这里waitClass就可以获取WaitForSeconds对象的等待时间了
	ScriptingClassPtr waitClass = scripting_object_get_class (monoWait, GetScriptingTypeRegistry());

	if (scripting_class_is_subclass_of (waitClass, classes.waitForSeconds))
	{
		float waitTime = 0;//通过waitClass获取WaitForSeconds对象的等待时间
		MarshallManagedStructIntoNative(monoWait, &waitTime);
	
		// 这里添加到DelayedCallManager里面
		// coroutineCallback里面处理了我们在一个方法里面多次yield return的操作，
		// 这个方法里面会递归调用Run直到所有的操作处理完成
		CallDelayed(coroutineCallback, monobehaviour, waitTime, , ,);
		return;
	}
}


//DelayedCallManager.cpp

void DelayedCallManager::Update()
{
	float time = GetCurTime();
	int frame = GetTimeManager().GetFrameCount();
	
	Container::iterator iterator = m_CallObjects.begin (); //m_CallObjects保存了所有注册的Coroutine对象
	
	// iterator->time 在注册的时候赋值是: 当前时间 + 等待的时间(new WaitForSeconds(5),那么就是5秒)
	// iterator->time <= time 这个条件判断了iterator的定时时间是否满足了
	// 比如上面我们加入定义new WaitForSeconds(5)，
	// 满足的条件时就当当前时间time要大于iterator满足的时间的时候，则进入这个while循环内
	while (iterator !=  m_CallObjects.end () && iterator->time <= time) 
	{
		//判断帧是否满足,加入用到了new WaitForFixedUpdate()之类的
		if (cb.timeStamp != m_TimeStamp && cb.frame <= frame) 		{
			//调用CoroutineCallback了
		}	
	}
}

```

其他的的类型会直接在下一帧调用,比如yield return 0

整个过程粗略的看大概就是这个样子。上面的分析没有深入到每个条件判断之类的，但是已经够了解Coroutine的全貌了。有错误的地方欢迎指出来，非常感谢。

