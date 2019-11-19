---
title: Unity GameObject深度剖析
date: 2016-05-09 17:44:40
type: "tags"
comments: false
tags: unity
---

**创建一个GameObject实例的过程是怎样的?**

UnityEngine的Mono对象 全部是用C++实现的，我们在Mono层的使用的如GameObject、MonoBehaviour这些对象都是Mono层绑定的C++对象。Mono层对这些C++对象做了简单的封装。下图是Mono层GameObject类的构造函数


```cs
//mono
public GameObject(string name)
{
    GameObject.Internal_CreateGameObject(this, name);
}

public GameObject()
{
    GameObject.Internal_CreateGameObject(this, null);
}

public GameObject(string name, params Type[] components)
{
    GameObject.Internal_CreateGameObject(this, name);
    for (int i = 0; i < components.Length; i++)
    {
        Type componentType = components[i];
        this.AddComponent(componentType);
    }
}
```

可以看到GameObject的所有构造函数都调用了Internal_CreateGameObject静态方法, 接着我们再看这个方法的走向

```cpp
	//cpp
	CUSTOM private static void Internal_CreateGameObject ([Writable]GameObject mono, string name)
	{
		DISALLOW_IN_CONSTRUCTOR
		GameObject* go = MonoCreateGameObject (name.IsNull() ? NULL : name.AsUTF8().c_str() ); 
		ConnectScriptingWrapperToObject (mono.GetScriptingObject(), go); 
	}
```

通过这个方法Internal_CreateGameObject方法是CShape创建并绑定C++对象的方法，其中很重的是MonoCreateGameObject方法，此方法创建了一个C++的GameObject对象，然后Scripting::ConnectScriptingWrapperToObject把新创建的GameObject的C++对象绑定到了Mono层的GameObject对象，这样我们就可以通过Mono层的GameObject对象访问到C++层的GameObject对象了(Scripting作用空间下定义了很多全局的静态方法，用于Mono和C++交互)。接着我们再看看MonoCreateGameObject方法

```cpp
//cpp
GameObject* MonoCreateGameObject (const char* name)
{
	string cname;
	if (!name)
	{
		cname = "New Game Object";
	}
	else
	{
		cname = name;
	}
	return &CreateGameObject (cname, "Transform", NULL);
}

//cpp
GameObject& CreateGameObject (const string& name, const char* componentName, ...)
{
	// Create game object with name!
	GameObject &go = *NEW_OBJECT (GameObject);

	ActivateGameObject (go, name);

	// Add components with class names!
	va_list ap;
	va_start (ap, componentName);
	AddComponentsFromVAList (go, componentName, ap);
	va_end (ap);

	return go;
}

```

最终我们看到通过在CreateGameObject方法中分配的内存创建的对象, 这是我们从Mono层创建一个GameObject对象的过程, 通过这个过程我们解释了问题一。

**通过GameObject对象获取到自身任何Component的内部实现是怎样的？**

Transform只是一个普通的Component和其他继承自Component的组建不同的是，在Mono层的GameObject对象包含了一个Transform的 property, 如下:

```cs
//mono
namespace UnityEngine
{
	public sealed class GameObject : Object
	{
		public extern Transform transform
		{
			[WrapperlessIcall]
			[MethodImpl(MethodImplOptions.InternalCall)]
			get;
		}
		...
	}
}

```

```cpp

//cpp
CUSTOM_PROP Transform transform { FASTGO_QUERY_COMPONENT(Transform) }

 #define FASTGO_QUERY_COMPONENT(x) GameObject& go = *self; return Scripting::GetComponentObjectToScriptingObject (go.QueryComponent (x), go, ClassID (x));

```
可以看到Mono的GameObject transform属性的get函数内部是通过GameObject的QueryComponent函数来得到C++层的Transform组件，再通过Scripting::GetComponentObjectToScriptingObject返回C++组建对应的Mono层Transform组件。现在我们知道其实Mono层的GameObject并没有持有任何Transform应用了。但是在C++呢？我们在看下面代码

```cpp
//cpp  GameObject.h
class EXPORT_COREMODULE GameObject : public EditorExtension
{
	public:

	typedef std::pair<SInt32, ImmediatePtr<Component> > ComponentPair;
	typedef UNITY_VECTOR(kMemBaseObject, ComponentPair)	Container;
	...
	private:
	Container	m_Component;
	...
}

```

```cpp
//GameObject.cpp
...
void GameObject::AddComponentInternal (Component* com)
{
	AssertIf (com == NULL);
	{
		m_Component.push_back (std::make_pair (com->GetClassID (), ImmediatePtr<Component> (com)));
	}
	...
}
...
void GameObject::AddComponentInternal (GameObject& gameObject, Component& clone)
{
	SET_ALLOC_OWNER(&gameObject);
	Assert(clone.m_GameObject == NULL);
	gameObject.m_Component.push_back(make_pair(clone.GetClassID(), &clone));
	clone.m_GameObject = &gameObject;
}
..
```

C++层的GameObject有一个成员变量m_Component。m_Component的作用可以通过代码中的两个方法AddComponentInternal方法可知，m_Component是用来存储添加到GameObject中的Component的集合。另外我还发现Container包装的Component用ImmediatePtr修饰了，那么ImmediatePtr有什么作用呢？通过查看代码我总结如（感兴趣的同学可以查看BaseObject.h/cpp文件)）: **Unity Engine在c++层封装了两个对象引用的包装类ImmediatePtr和PPtr。用ImmediatePtr包装的对象引用属于弱引用，用PPtr包装的对象引用属于强引用**。变量m_Component为什么要定义成弱类型的呢？稍后再讲。现在GameObject的m_Component保存所有的添加的Component，那就解释了问题二。

**通过Component(Component的派生类，如transform)获取到对应的Gameobject对象的内部实现是怎样的?**

我们看下面代码

```cs
//Mono
public class Component : Object
{
		public extern GameObject gameObject
		{
			[WrapperlessIcall]
			[MethodImpl(MethodImplOptions.InternalCall)]
			get;
		}
		...
}

```

```cpp
//cpp

class EXPORT_COREMODULE Component : public EditorExtension
{
	private:

	ImmediatePtr<GameObject>	m_GameObject;
	...
}
```
通过上面代码可以看出，Component持有了GameObject的引用，这样就解释了问题三了。此外我们还发现了m_GameObject用ImmediatePtr修饰了，现在我们应该明白了为什么要用ImmediatePtr来修饰GameObject的成员m_Component了，因为GameObject和Component两个类互相持有引用，造成循环引用了。所以Unity 自己封装了保持弱引用功能的ImmediatePtr类解决了这个问题。