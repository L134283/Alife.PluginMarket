# Alife Plugin Market

这里是 Alife 插件市场的插件注册表仓库，存放了所有插件的元数据。

仓库支持自动合并PR，只要你是的新建文件，或者修改的文件没有冲突且是是上次提交者即可。

## 插件与模块

在 Alife 中，插件和模块是分开的两种概念。不过要注意的是，一般常说的插件在 Alife 其实对应模块，因此如果有人说想做个插件，那可能时想通过模块来调整 Alife 的功能，而不是发布一个插件。

### 模块（Module）

1. 模块是 Alife 中的功能单元，是运行时注入到 Agent 环境的实例。
2. 只要是被打上`[Module]`标签的类即可被识别为模块，并被显示在功能模块页面。
4. 模块通过 ModuleSystem 热编译加载，因此可以是 cs 或 dll。
5. 模块存放在 Storage/Plugins，但本身与插件无关，存放其中的 cs 或 dll 可直接被识别加载。

### 插件（Plugin）

1. 插件在 Alife 中代表 Storage/Plugins 中的一个文件夹，并且有在插件市场进行注册。
2. 插件是用于分发模块的载体，且由于注册信息的存在，额外支持安装环境依赖的能力。

插件本身只用于分发和依赖描述，不负责编译。编译流程实际是 PluginMarketService 在插件变动后，将环境信息传递给 ModuleSystem，然后 ModuleSystem 将 Plugins 文件夹整体性的编译加载。

## 编写一个模块

Alife 采用全模块化架构，内置的所有核心功能（如视觉、语音、浏览器等）均通过模块实现。开发者可以参考 Alife 源码仓库 `Sources/Alife.Function` 中的项目，这是最快的学习方式。

### DLL 工作流

1. 新建一个Dll工程，引用 Alife.Framework 项目。
2. 创建一个类，继承 `InteractiveModule<T>`，并添加 `[Module]` 属性，此时这个类就代表一个功能模块（插件中的功能单元）。
3. 打包出Dll后将其放入到`{存储目录}/Plugins`文件夹，当触发插件加载时，该Dll中的插件类就会增加到插件选项中。

### CS 工作流（推荐）

Alife 的一个特点就是支持热编译，因此允许直接在插件文件夹，编写原始的 cs 文件。编译时，系统会自动收集客户端 dll 、插件文件夹 dll、插件依赖的 NuGet，然后将插件文件夹中的所有的 cs 文件，组合后热编译成 dll ，再加载。

### 常见需求

- 热编译重载：插件支持热编译重载，替换dll或修改cs后直接点击插件页面上的刷新按钮即可重新编译加载插件文件夹中的插件。
- 函数调用：实现函数调用有两种途径，基于XmlFunctionCaller或SemanticKernel（不推荐），然后在插件的AwakeAsync事件中注册即可。
- LLM通讯：llm在Alife中被封装成了ChatBot对象，其提供了Poke（排队式）和Chat（打断式）两种通讯方式，此外还提供事件让你能监听这个过程。
- 提示注入：Alife全程都始终维护一个上下文，其对应ChatHistory对象。可以直接修改其内容，来改变上下文，其中System类型信息专用于系统提示词。
- 配置文件：通过实现各种接口，插件就可以接入各种扩展功能，比如实现IConfigurable后，即可为插件编辑配置文件，接着系统会在Awake前注入配置对象。

### 代码示例 (C#)

节选自 Alife.Demo.Plugin 项目

```csharp
public class MyModuleData
{
    public int DefaultMax { get; set; } = 120;
}

[Module(
    "我的功能模块", "一个示例功能模块",
    EditorUI = typeof(MyModuleUI)/*支持用razor自定义模块界面*/
)]//只要被打上Module标签的类就会被认为是功能模块，可以让用户勾选，或者也可以通过`角色文件夹/index.json`中的`Modules`属性来编辑启用的模块。
public class MyModule(
    XmlFunctionCaller functionService,//可直接在构造函数申请其他模块，系统会自动通过依赖注入填充，此外XmlFunctionCaller提供函数调用的能力，是非常常用的基础模块
    ILogger<MyModule> logger//也支持申请专用的logger，以及各种全局系统，具体可见 ChatActivitySystem 的创建过程
) :
    InteractiveModule<MyModule>,/*封装好地模块基类，便于快速开发*/
    IConfigurable<MyModuleData>/*通过实现IConfigurable接入配置功能*/
{
    [XmlFunction(FunctionMode.OneShot)]// 表明该函数支持让AI通过Xml函数调用且格式为自闭合标签
    [Description("随机生成一个数字")]// 提供给AI的函数描述
    public Task Rand([Description("随机的最大范围")] int? max = null/*支持任何可被字符串转换的参数，包括默认值可选这些特性*/)
    {
        if (max == null)
            max = Configuration!.DefaultMax;//配置在模块构造后立即注入，故系统事件期间都是不为空的
        if (max < 0)
            throw new Exception("最大值必须大于 0");//可以正常抛出异常

        int value = Random.Shared.Next(max.Value);
        Poke("随机数结果：" + value);//向AI反馈结果(可选，如果函数的功能不需要返回结果，可以去除)
        logger.LogInformation($"调用 {nameof(Rand)} 结果 {value}");//支持依赖注入的Logger

        return Task.CompletedTask;//如果有需要你可以使用异步代码
    }

    public MyModuleData? Configuration { get; set; }

    public override async Task AwakeAsync(AwakeContext context)
    {
        await base.AwakeAsync(context);

        //注册函数调用
        XmlHandler xmlHandler = new(this);
        functionService.RegisterHandlerWithoutDocument(xmlHandler);
        //添加自定义提示词
        Prompt($"""
                此服务可以为你提供一个生成随机数的功能。
                ## 提供工具
                {xmlHandler.FunctionDocument()}
                """);
    }
}
```

## 贡献一个插件

当你有了一些不错的模块后，可以考虑分发他们，将他们整合为一个插件。而且借此，你还可以实现自动化的环境和依赖管理。插件市场可以自动合并多个插件的环境依赖，判断冲突和安装，包括将其他插件作为依赖项，而实现连锁安装。

贡献插件非常简单，只需在本仓库以插件 ID 为名，提交一个 json 文件，用于描述插件信息即可。

```json
{
  "id": "MyPlugin.Example",//插件唯一标识，与文件夹名一致
  "name": "示例插件",//插件显示名称
  "author": "作者名",//插件作者
  "description": "插件功能描述",//插件显示描述
  "tags": ["视觉模型", "官方"],//插件标签，用于分类筛选（可选）
  "source": "https://github.com/xxx",//插件主页或联系方式（可选）
  "dependencies": { //依赖的其他插件
    "{PluginID}": "{VersionDescription}" //版本描述采用pip格式，支持`>=`,`<=`,`==`,留空
  },
  "environments": { //依赖的环境（支持pip和nuget）
    "nuget": {
      "{PackageName}": "{VersionDescription}" 
    },
    "pip": {
      "{PackageName}": "{VersionDescription}" 
    }
  },
  "releases": {//版本发行信息
    "1.0.0": {
      "date": "2026-06-12",
      "note": "初始版本",
      "file": "https://MyPlugin.Example/1.0.0.zip" //存放你的模块cs,dll的zip网址，这些内容会被实际解压到 Plugins/{PluginID} 目录下
    }
  }
}
```

## 相关仓库

- [Alife](https://github.com/BDFFZI/Alife) - 主项目
- [Alife.OfficialPluginStorage](https://github.com/BDFFZI/Alife.OfficialPluginStorage) - 官方插件包存储
