---
title: Unity脚本自动构建IPA包
date: 2017-06-18 23:30
type: "tags"
comments: false
tags: unity
---


Unity项目在iOS平台打包是一件比较费时间的事情，所以写了脚本来实现自动话打包生成ipa文件，避免每次去到处xcode工程然后配置证书之类的设置。

>测试环境 Unity5.5.1f3, macOS 10.12.4, Xcode 8.3.2

#### 工具使用介绍

打包的配置用到了很重要的[XUPorter](https://github.com/onevcat/XUPorter)插件，打包的代码[UnityIPABuilder](https://github.com/elangduan/UnityIPABuilder)。下面简单讲解一下里面的具体实现

Xcode打包主要有两个步骤，第一步是导出Xcode工程，第二步是构建Xcode工程并且导出ipa包。这里两种方式启动打包步骤。

*第一种*在Editor模式下打包。制作了一个Editor界面如下:

![](/images/unity_ipa_builder/ipa_builder_window.jpeg)

里面的Export按钮的功能是直接导出一个配置号参数的Xcode工程，Build按钮的功能是首先会进行Export操作，然后对Export出的工程进行Build直接生成IPA包。整个界面比较简单，如果不满足自己的需求的话可以通过在[BuildWindow](https://github.com/elangduan/UnityIPABuilder/blob/master/Assets/Editor/ProjectBuilder/BuilderWindow.cs)里面添加。

*第二种*不启动Editor，直接用脚本打包。这种模式下我们直接启动项目打包脚本[cmd_start_unity_ipa_builder](https://github.com/elangduan/UnityIPABuilder/blob/master/Builder/Script/cmd_start_unity_ipa_builder.sh)加上对应的参数即可。例如sample工程的打包直接运行一下命令(先配置好*xcode_basic.cfg*)即可得到最终的ipa文件:

```
./cmd_start_unity_ipa_builder.sh -v 1.0.0 -b 100 -c QA

```
这个脚本会静默启动Unity来执行*ProjectBuilder*的打包方法。并且会把打包的日志存放在{$ProjectPath}/iOSBuild/build/log目录下。

#### 导出Xcode工程部分

这一步可以通过Unity的内置API完成。在这一步中我们还需要做的是设置好导出的Xcode的工程的各种配置(证书，版本号，Framework等)。导出Xcode工程直接只用Unity的API

```
BuildPipeline.BuildPlayer(buildPlayerOptions);
```
这个API可以通过BuildOptions配置在Editor中的*Player Setttings*里面的各种配置。我的工具里面只添加了ScriptingImpletion、Architecture、渠道、Version和Build五个参数。这是根据我自己的需求来添加的，其他的设置在*Palyer Setting*设置好了基本上就不需要动态改变了，这几个参数是每次打包改动比较频繁的选项。


先看下我们的整体构建步骤:

```
  public static void BuildForiOS(ProjectBuildData data)
  {
  		//导出Xcode工程
      ExportForiOS(data);
  
  	   //构建Xcode工程
      BuildiOSPorject();
  }

  public static void ExportForiOS(ProjectBuildData data)
  {
      //根据data数据来导出对应的Xcode工程
	  ....
		
	   //配置Xcode工程
      XCodePostProcess.OnPostProcessBuild(BuildTarget.iOS, BuildPaths.ProjectPath, data.Channel, data);
  }

```

导出Xcode工程之后我们要做的就是配置Xcode工程。这里我用的是XUPorter插件来进行修改的，这个插件可以很方便的帮助我们自动把依赖的Framework、静态库、文件等添加到Xcode工程，这样可以解决我们在使用第三方插件的时候依赖的库每次打包需要手动添加的问题(5.5.1f1版本支持添加iOS系统自带的Framwork，但是不支持添加第三方的库，比如libsqlite3.dylib文件)。另外为了配置为了配置证书和Info.plist文件，稍微改动了一下里面的代码。

```
		public static void OnPostProcessBuild(BuildTarget target, string xcodeProjectPath, PackageChannel channel, ProjectBuildData data)
		{
			if (target != BuildTarget.iOS) {
				Debug.LogWarning("Target is not iPhone. XCodePostProcess will not run");
				return;
			}

			//这里来动态XUPorter的配置文件来自动添加Framework等一些XCode的配置
			XCProject project = new XCProject(xcodeProjectPath + "Unity-iPhone.xcodeproj");

			string[] files = Directory.GetFiles( Application.dataPath, "*.projmods", SearchOption.AllDirectories );
			foreach( string file in files ) {
				project.ApplyMod( file );
			}

			//读取我们自己的配置文件
			ProjectBasicConfig config = new ProjectBasicConfigLoader().GetConfig(channel);
		
			//这里配置证书
			OverWriteBuildSetting(config, project);
		
		   //这里配置Info.plist文件，比如加权限描述，修改版本号之类的
			OverWriteInfoPlist(config, xcodeProjectPath, data);
		
			project.overwriteBuildSetting ("ENABLE_BITCODE","NO");
			
			//保存修改
			project.Save();
		}

```

OnPosProcessBuild函数在XUPorter插件里面[XCodePostProcess](https://github.com/elangduan/UnityIPABuilder/blob/master/Assets/Editor/ProjectBuilder/XCodePostProcess.cs)文件里面的一个函数,这个函数本来是加了*[PostProcessBuild(999)]*属性，加了这个属性的函数是Unity在BuildPlayer调用之后会主动调用的函数。但是这里我们去掉这个属性，我们不需要Unity来主动调用这个函数。因为我们的构建的步骤是: BuildPlayer导出Xcode工程，XUPorter配置XCode工程，shell脚本构建XCode工程。XCodePostProcess属于第二步XUPorter配置XCode工程这一步，如果让Unity来主动调用XCodePostProcess的话在不修改XUPorter的情况下我们无法得知XUPorter执行完毕的时间。所以我们自己在调用完BuildPlayer之后接着调用XCodePostProcess然后再调用shell构建，这样顺序来执行。

对于配置XUPorter的参数，大家可以参考工程的XUPorter插件下面的Bugly.projmods配置文件。详细的解释可以参考作者的[原文介绍](https://onevcat.com/2012/12/xuporter/)。至于ProjectBasicConfig配置，这个直接共用的shell脚本使用的配置文件，构建部分详解下。

至此我们可以得到一个配置号所有参数的Xcode工程。

#### 构建Xcode工程部分

构建Xcode部分我设计的是不依赖Export部分，也就是说我们不需要在Export部分配置好证书或者版本号等这些参数([xcode_basic.cfg](https://github.com/elangduan/UnityIPABuilder/blob/master/BuildScript/xcode_basic.cfg)文件包含的参数)，对于任何Xcode工程我们都可以通过这个脚本进行Build生成IPA包。构建脚本只有[auto_build_xcode_project](https://github.com/elangduan/UnityIPABuilder/blob/master/Builder/Script/auto_build_xcode_project.sh)一个文件，使用方法我们可以通过执行命令查阅:

```
sh auto_build_xcode_project.sh --help

```
我们可以先对shell脚本添加可执行权限，之后运行以来就不需要sh命令了

```
chmod +x auto_build_xcode_project.sh

./auto_build_xcode_project.sh --help
```

当然，我们在使用这个脚本之前请先配置这个脚本对应的配置文件[xcode_basic.cfg](https://github.com/elangduan/UnityIPABuilder/blob/master/Builder/Script/xcode_basic.cfg)，这个配置文件里面包含的参数是我们在使用这个脚本自动打包可以动态改变的所有参数。如果还需要修改额外的参数，那就需要自己另外添加了。整个配置文件我分了三个渠道，分别是Debug，QA和Release。每个渠道的配置的参数是一样的:

```
channel=Debug

#证书
code_signing_identity=xxx

#旧版描述文件
provisioning_profile=xxx

#新版本描述文件
provisioning_profile_specifier=xxx

#账号的TeamID
team_id=xxx

#包ID
bundle_identity=xxx

#包安装之后的显示名字
display_name=xxx
```
打包脚本会自动读取相应的渠道的配置，这个配置的参数是有序的，不要改变他们之间的顺序。这里和大家说下怎么获取得到描述文件和证书的指。

描述文件在Xcode里面的*Build Settings*里面Signing里面先选择对应的描述文件，然后在选项中选择*other*就会选择我们描述文件的值:

![](/images/unity_ipa_builder/provisining_profile_get_method.png)

证书直接在系统应用*钥匙串访问里面查看*,双击点开一个钥匙串可以看得到:

![](/images/unity_ipa_builder/code_signing_identity_get_method.jpeg)

直接复制里面的*常用名称*对应的值即可。


其实Xcode的自动构建比较简单，主要是通过XcodeBuild来构建，构建好了之后会生成一个app包，然后再通过xcrun命令把这个app包压缩\转换成ipa包即可。

```
...

//构建xcode，生成app包
xcodebuild  -project $xcode_project_filepath \
            -configuration $build_mode \
            -target $project_target \
            PROVISIONING_PROFILE="$provisioning_profile" \
            CODE_SIGN_IDENTITY="$code_signing_identity" \
            PRODUCT_BUNDLE_IDENTIFIER=$bundle_identity \
            PRODUCT_NAME=$product_name \
            DISPLAY_NAME=$display_name

...

转换app成ipa文件
xcrun -sdk iphoneos PackageApplication -v $app_package_path -o $package_unique_path

```

至此我们就得到了ipa包了。如果需要自动上传到发布平台(比如fir.im)，可以直接在生成ipa包之后调用发布平台提供的上传命令直接上传即可，这样整个过程就完全自动化了。