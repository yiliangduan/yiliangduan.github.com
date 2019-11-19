---
title: MonoBehaviour的Awake,Start等函数的调用原理
date: 2017-03-08 18:12:40
type: "tags"
comments: false
tags: unity
---

在用Unity的时候一直有一个比较困惑问题，继承自MonoBehaviour脚本的Awake(),Start(),Update()等这些函数是怎样被调用的。今天阅读了下Unity的代码了解了其中的调用原理,然后自己写了一个sample来模拟Unity中的调用方式。

Unity的引擎是用C++实现的，自己实现的MonoBehaviour的脚本却是CS脚本。所以这里就涉及到C++调用CS函数的过程。Unity中用的Mono的Framework来实现在C++调用CS脚本，不多说直接上sample就懂了。

> 测试环境: macOS 10.12.3 , mono 4.8.0


有一个注意的地方就是必须设置好**PKG_CONFIG_PATH**环境变量，不然gcc在链接exe文件时候会报错,我是这样配置的: 
export PKG_CONFIG_PATH=/Library/Frameworks/Mono.framework/Versions/Current/lib/pkgconfig/

```cs
\\MonoBehaviour.cs
\\这里模拟我们在CS中MonoBehaviour的类
using System;
namespace Unity
{
    public class MonoBehaviour
    {
        private void Awake()
        {
			Console.WriteLine("Awake");
		}
		protected void OnEnable()
		{
			Console.WriteLine("OnEnable");
		}
		public void Start()
		{
			Console.WriteLine("Start");
		}
	}
	public class Test
	{
		static void Main () {}
	}
}

```

编译MonoBehaviour.cs生成MonoBehaviour.exe 或者 MonoBehaviour.exe.dll (此处我直接生成exe文件)

> mcs MonoBehaviour.cs

然后实现C++层调用代码(此处是根据mono的[官方文档](http://www.mono-project.com/docs/advanced/embedding/)和官方的sample来写的)

```cpp
\\InvokeMonoBehaviour.c
\\这里用C语言写的，C++同理
\\这个脚本模拟Unity引擎内部的调用方式

#include <mono/jit/jit.h>
#include <mono/metadata/object.h>
#include <mono/metadata/environment.h>
#include <mono/metadata/assembly.h>
#include <mono/metadata/debug-helpers.h>
#include <string.h>
#include <stdlib.h>

static void call_methods (MonoObject *obj)
{
	MonoClass *klass;
	MonoDomain *domain;
	MonoMethod *awake = NULL, *m = NULL, *onenable = NULL, *start = NULL, *mvalues;
	MonoProperty *prop;
	MonoObject *result, *exception;
	MonoString *str;
	char *p;
	void* iter;
	void* args [2];
	int val;

	klass = mono_object_get_class (obj);
	domain = mono_object_get_domain (obj);

	/* retrieve all the methods we need */
	iter = NULL;

	//从MonoBehaviour.exe中查找MonoBehaviour.cs中的Awake，OnEnable，Start方法。
	//通过这种方式不管在cs中的访问权限是private,protected还是public都可以获取得到
	
	while ((m = mono_class_get_methods (klass, &iter))) {
		if (strcmp (mono_method_get_name (m), "Awake") == 0) {
			awake = m;
		} else if (strcmp (mono_method_get_name (m), "OnEnable") == 0) {
			onenable = m;
		} else if (strcmp (mono_method_get_name (m), "Start") == 0) {
			start = m;
		} 
	}

	//这里调用了MonoBehaviour的Awake函数
	mono_runtime_invoke (awake, obj, NULL, NULL);

	//这里调用了MonoBehaviour的OnEnable函数
	mono_runtime_invoke (onenable, obj, NULL, NULL);

	//这里调用了MonoBehaviour的Start函数
	mono_runtime_invoke (start, obj, NULL, NULL);
}

static void create_object (MonoDomain *domain, MonoImage *image)
{
	MonoClass *klass;
	MonoObject *obj;

	klass = mono_class_from_name (image, "Unity", "MonoBehaviour");
	if (!klass) {
		fprintf (stderr, "Can't find MyType in assembly %s\n", mono_image_get_filename (image));
		exit (1);
	}

	obj = mono_object_new (domain, klass);
	/* mono_object_new () only allocates the storage: 
	 * it doesn't run any constructor. Tell the runtime to run
	 * the default argumentless constructor.
	 */
	mono_runtime_object_init (obj);

	call_methods (obj);
}

static void main_function (MonoDomain *domain, const char *file, int argc, char **argv)
{
	MonoAssembly *assembly;

	/* Loading an assembly makes the runtime setup everything
	 * needed to execute it. If we're just interested in the metadata
	 * we'd use mono_image_load (), instead and we'd get a MonoImage*.
	 */
	assembly = mono_domain_assembly_open (domain, file);
	if (!assembly)
		exit (2);
	/*
	 * mono_jit_exec() will run the Main() method in the assembly.
	 * The return value needs to be looked up from
	 * System.Environment.ExitCode.
	 */
	//mono_jit_exec (domain, assembly, argc, argv);

	create_object (domain, mono_assembly_get_image (assembly));
}


int main (int argc, char* argv[]) {

	const char *file = "MonoBehaviour.exe";

	/*
	 * mono_jit_init() creates a domain: each assembly is
	 * loaded and run in a MonoDomain.
	 */
	MonoDomain *domain = mono_jit_init (file);

	main_function (domain, file, argc - 1, argv + 1);

	int retval = mono_environment_exitcode_get ();
	
	mono_jit_cleanup (domain);
	return retval;
}

```

编译InvokeMonoBehaviour.c生成了a.out文件

> gcc InvokeMonoBehaviour.c \`pkg-config --cflags --libs mono-2\`

然后执行a.out文件

>./a.out

可以看到terminal中输出:

```
Awake
OnEnable
Start
```

当我们编译了InvokeMonoBehaviour.c之后我们再次编辑MonoBehaviour.cs就不需要再编译InvokeMonoBehaviour.c文件就可以加载最新编译出的exe文件,这样还是比较方便的。

整个调用过程就是这样的。在Unity中我们编写的CS脚本被编译成dll存放在项目根目录的Library/ScripAssemblies目录下面(Editor模式),Unity引擎内部封装了一个Manager来专门管理和加载这些dll。加载的方式和上面sample的方式是一样的。