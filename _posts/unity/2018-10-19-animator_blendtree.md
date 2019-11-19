---
title: Unity的Animator BlendTree
date: 2018-10-19 20:26:10
type: "tags"
tags: unity
comments: false
---

> unity 5.6.6f1

在游戏中角色的两个动作直接切换过度效果上，如果两个动作没有很好的衔接会出现一定程度上的僵硬现象。这种情况可以用BlendTree来进行融合。比如这样:

![](/images/unity_blendtree/2.png)

这里配置了三个Motion动作，三个动作之间的切换利用Controller的Parameter来控制，这个Parameter同时可作为BlendTree的Thresholds。Thresholds是用来配置各个Motion的触发条件的。

<!-- more --> 

配置好这个BlendTree之后，这个BlendTree可以作为AnimatorController的State来触发，如图:

![](/images/unity_blendtree/5.png)

这样当Animator的state进入到RelaxState时就自动根据Thresholds来播放对应的Motion了。基于这套流程可以很方便的应用到游戏里的AI怪物之类行为单位。下面举个例子。

现在有个带Fly, Run,Walk,Idle动画的模型，针对这个模型来做个AnimatorController并且驱动这个Controller来使得这个模型随机做动作。这里设计这个模型每次Fly动作一段时间然后随机执行其余三个动作的其中一个，根据这个设想创建了如下的控制器(可以根据脚本来创建，代码在最后)，RelaxState配置包含三个Motion(Run, Walk, Idle)的BlendTree，FlyState配置FlyMotion。

![](/images/unity_blendtree/1.png)

配置State之间的控制条件BehaviourId

![](/images/unity_blendtree/4.png)

然后配置好BlendTree的Thresholds参数RelaxRandom。配置好控制器之后，然后在脚本里面按照自己设定的规律来控制BehaviourId和RelaxRandom就可以驱动这个模型来做对应的动作了，这样很方便。

```csharp
    public enum AnimState
    {
        Relax,
        Fly
    }

    public void Update()
    {
        //一段时间之后进入Relax状态
        if (mAnimState == AnimState.Fly)
        {
            if (mFlyTimer.IsTimeUp)
            {
                EnterRelaxState();
            }
        }
        //一段时间之后进入Fly状态
        else if (mAnimState == AnimState.Relax)
        {
            if (mRelaxTimer.IsTimeUp)
            {
                EnterFlyState();
            }
        }
    }

    public void EnterRelaxState()
    {
        //这里的范围固定是[0,100)不管是多少个Motion,所有motion的阀值加起来等于100即可.
        int randomValue = Random.Range(0, 100);

        TargetAnimator.SetInteger(mAnimArgument_BehaviourId, 2);
        //随机一个BlendTree的Thresholds,让单位的行为不规律
        TargetAnimator.SetFloat(mAnimArgument_RelaxRandom, randomValue);
        mAnimState = AnimState.Relax;

        mRelaxTimer.Reset(RelaxDuration);
    }

    public void EnterFlyState()
    {
        TargetAnimator.SetInteger(mAnimArgument_BehaviourId, 1);
        mAnimState = AnimState.Fly;

        mFlyTimer.Reset(FlyDuration);
    }
```

针对于上面配置Controller的各个手动操作也可以通过脚本来一次生成。代码如下:

```csharp
public class AnimatorControllerGenerator : Editor
{
    #region 拷贝AnimationClip
    private const string AnimationClipDir = "Assets/AnimClips/";
    [MenuItem("Assets/DuplicateMotion")]
    static void DuplicateMotion()
    {
        Object activeObject = Selection.activeObject;

        if (null != activeObject)
        {
            if (!Directory.Exists(AnimationClipDir))
            {
                Directory.CreateDirectory(AnimationClipDir);
            }

            AnimationClip activeAnimClip = activeObject as AnimationClip;
            AnimationClip newObject = Object.Instantiate<AnimationClip>(activeAnimClip);

            AssetDatabase.CreateAsset(newObject, AnimationClipDir + activeObject.name.Replace("|", string.Empty) + ".asset");

            AssetDatabase.SaveAssets();
            AssetDatabase.Refresh();
        }
    }

    [MenuItem("Assets/DuplicateMotion", true)]
    static bool DuplicateMotionValidation()
    {
        return Selection.activeObject.GetType() == typeof(AnimationClip);
    }
    #endregion

    #region 生成AnimationController
    private const string ControllerFileDir = "Assets/Mecanim/";


    private const string FlyMotionPath = "Assets/AnimClips/ArmatureFly_New.asset";

    private static string[] RelaxMotions = { "Assets/AnimClips/ArmatureIdel_New.asset",
                                            "Assets/AnimClips/ArmatureRun_New.asset",
                                            "Assets/AnimClips/ArmatureWalk_New.asset"};

    private class ControllerParameter
    {
        public const string RelaxRandom = "RelaxRandom";
        public const string BehaviourId = "BehaviourId";
    }

    [MenuItem("Assets/CreateDragonController")]
    static void CreateController()
    {
        if (!Directory.Exists(ControllerFileDir))
        {
            Directory.CreateDirectory(ControllerFileDir);
        }

        string controllerPath = ControllerFileDir + "StateMachineTransitions.controller";
        
        // Creates the controller
        AnimatorController controller = AnimatorController.CreateAnimatorControllerAtPath(controllerPath);

        controller.AddParameter(ControllerParameter.BehaviourId, AnimatorControllerParameterType.Int);
        controller.AddParameter(ControllerParameter.RelaxRandom, AnimatorControllerParameterType.Float);

        // Add StateMachines
        AnimatorStateMachine rootStateMachine = controller.layers[0].stateMachine;

        //Add States
        AnimatorState stateFly = rootStateMachine.AddState("Fly");
        stateFly.motion = AssetDatabase.LoadAssetAtPath<AnimationClip>(FlyMotionPath);

        AnimatorState stateRelax = rootStateMachine.AddState("Relax");
        BlendTree blendTree = new BlendTree()
        {
            name = "RelaxTree",
            blendParameter = ControllerParameter.RelaxRandom,
            useAutomaticThresholds = false
        };

        stateRelax.motion = blendTree;

        int blendTreePartFactor = 100 / RelaxMotions.Length;
        int curMotionThreshold = 0;

        for (int i=0; i< RelaxMotions.Length; ++i)
        {
            blendTree.AddChild(AssetDatabase.LoadAssetAtPath<AnimationClip>(RelaxMotions[i]), curMotionThreshold);
            curMotionThreshold += blendTreePartFactor;
        }

        AnimatorStateTransition flyToRelax = stateFly.AddTransition(stateRelax);
        flyToRelax.AddCondition(AnimatorConditionMode.Equals, 2, ControllerParameter.BehaviourId);

        AnimatorStateTransition relaxToFlyTransition = stateRelax.AddTransition(stateFly);
        relaxToFlyTransition.AddCondition(AnimatorConditionMode.Equals, 1, ControllerParameter.BehaviourId);
    }
    #endregion
}
```

*End*