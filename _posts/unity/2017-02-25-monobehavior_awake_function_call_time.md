---
title: 'MonoBehaviour的Awake的调用时机'
date: 2017-02-25 13:56:36
type: "tags"
tags: unity
comments: false
---

对MonoBehaviour的Awake之前有一个一直错误的理解:如果一个GameObject被添加到scene中，就算Active为false只要场景运行，此GameObejct的Awake函数就会调用。此理解是看了[官方文档](https://docs.unity3d.com/ScriptReference/MonoBehaviour.Awake.html)自己得出的结论:

> Awake is called when the script instance is being loaded.

昨天同事问到这个问题的时候才自己测试了一遍，才发现我的理解是错误的。测试发现Awake只有在Active为true的时候会被调用一次。此外在查找资料的时候发现了Unity论坛里面有人在讨论这个问题[Something like Awake that runs even if disabled?](http://answers.unity3d.com/questions/942915/something-like-awake-that-runs-even-if-disabled.html)。有答案说构造方法会在GameObject的Active为false的时候会被调用，测试了一下却是如此。

继续浏览了Monobehaviour的文档，发现文档里面也着重写了这个问题:

> Note: The checkbox for disabling a MonoBehavior (on the editor) will only prevent Start(), Awake(), Update(), FixedUpdate(), and OnGUI() from executing. If none of these functions are present, the checkbox is not displayed.

<!-- more --> 

总结: 看文档要仔细，理论的知识都要自己实践并验证一下。