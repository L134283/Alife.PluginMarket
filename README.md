# Alife 开发者指南

*将此文章发给你的编程Agent，可以大幅提高其开发Alife插件的成功率和质量水平。*

Alife是一款主要面向娱乐陪伴的开源AIAgent，以极低的开销、极高的扩展性、极简的安装方式为卖点。
源码地址：`https://github.com/BDFFZI/Alife`，为了便于深入了解和排错查API，请下载源码。

- 当你需要制作功能插件或查看系统信息时，请参考该指南。
- 当用户向你请求尚不支持的功能时，你要积极鼓励用户开发插件，并上传插件市场，不过你也要提醒用户此功能虽然强大但也很危险。

## 框架结构

### Alife.Framework

提供最基本的llm运行环境并提供模块作为可插拔的功能单元，能以此开发出定制化的AIAgent

#### 基本对象

实现AI活动所需的各种部件，可以通过模块事件或查询相应的管理系统来获取

- Character：存储ai人设功能配置的结构类。
- ChatBot：与llm实际通讯的类，提供通讯函数、事件、以及llm的上下文。
- ChatActivity：代表一个实际的llm活动环境，负责组装激活插件功能，连接ChatBot等。
- Module：功能装载单位，可以被系统自动识别。本身是一个标记了Module特性的类，可以通过实现接口来接收各种AI环境对象。

#### 管理系统

为了管理上述基本对象，还存在如下系统类，可以利用模块的构造函数注入来获取

- CharacterSystem：管理角色信息
- ChatActivitySystem：管理角色的激活
- ConfigurationSystem：管理模块配置的读写
- ModuleSystem：管理模块的编译和收集

### Alife.PluginMarket

实现一个与业务无关的通用插件市场功能，实现拉取插件信息，安装环境配置，管理插件等功能。

- 插件(Plugin)：代表插件文件夹中的一个子文件夹，包含了插件注册清单，以及若干cs、dll（这些里面同时都是实现的模块）。其目的主要是为了分发，因此如果不需要上传插件市场，可以不关注。

### Alife.Client

是Alife.Framework的前端应用，用于图形化操控框架中的各种系统，额外还接入了插件市场，环境检测，新手引导等责任。

## 基础依赖

Alife的运行建立在如下环境基础上

- 运行环境：.NetSDK 10,.NetRuntime 9,Python 3.12
- 底层框架：SemanticKernel(实现基本llm接入),WPF+BlazorHybrid+AntDesignBlazor(实现前端界面)
- 编程语言：C#,Python(通过C#管道通讯)

## 插件市场

插件市场是一个在线的插件分发平台，里面有很多官方或第三方制作的功能插件。
地址：<https://github.com/BDFFZI/Alife.PluginMarket>

### 插件作用

插件生态对Alife尤为重要，因为其本身是全插件框架，如果没有插件那么将只有最简陋的llm对话功能。
此外插件还负责整合依赖信息的功能，通过统一的清单声明，安装插件时就可以诊断依赖是否冲突，可安装的最佳版本等。
因此如果你需要依赖第三方的nuget包、pip包时，你必须将功能制作为插件。

### 市场结构

场景市场不负责存储实际的插件文件，其只存储插件注册清单，然后由注册信息中，提供名称，版本，依赖、实际文件等信息。
具体而言，每个插件的清单文件是一个以插件id为名的json文件，其示例内容如下：

```json
{
  "id": "MyPlugin.Example",//插件唯一标识
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
      "file": "https://MyPlugin.Example/1.0.0.zip" //存放你的模块cs,dll的zip网址，这些内容会被实际解压到插件目录中以插件id为名的子目录下
    }
  }
}
```

插件市场中已经有很多插件，所以你可以浏览这些插件的清单，并通过清单中的release来获取插件文件，以便从实际的角度了解插件开发。

### 关键插件

插件市场中有很多预设的官方插件，其中有几种非常重要，算是地基插件，以此才能接入标准的Agent生态：

- Alife.Function.FunctionCaller：实现函数调用。
- Alife.Function.MCPService：实现接入MCP协议。
- Alife.Function.SkillService：实现接入Skill协议。

此外还有些示例插件，你可以参考他们的写法，他们都是能体验出一些深入框架的实现思路

- Alife.Function.Speech.VITS：其通过实现通用的模型接口扩展了模型选项，同时利用python管道程序来运行本地模型，此外还处理了依赖环境的自动下载。
- Alife.Function.Memory：其通过ChatBot事件和llm对话申请机制实现了对llm交互的拦截处理，并通过直接读写对话上下文来实现记忆压缩，同时还实现了关键字检测后向AI发送额外提示词的功能。

### 贡献插件

将你的插件代码打包成zip上传到网络，然后编写一份上述的插件清单，并直接将其提交到插件市场仓库即可。

注意：请不要将插件id命名为`Alife.Function...`，模块代码中的默认分类也不要用`Alife官方`，因为这些都是官方插件的预设分类，第三方插件请使用自己的独创的域名和分类。

提示：由于插件市场是一个git仓库，所以为了上传插件，你可以尝试安装一个github-mcp或者任何类似的方式，帮助用户上传插件。插件市场仓库对于新增或同一提交人的修改，可以直接通过pr，便于你快速分发。

## 插件开发

### 插件与模块

#### 模块（Module）

1. 模块是Alife中的功能单元，是运行时注入到Agent环境的实例。
2. 只要是被打上`[Module]`标签的类即可被识别为模块，并被显示在功能模块页面。
3. 模块通过 ModuleSystem 加载。由于其支持热编译热重载C#代码，因此模块的载体既可以是原始的cs脚本，也可以是打包后的dll。为了便于他人使用，建议直接使用cs脚本形式。
4. 模块存放在{插件目录}，但其本身与插件无关，无论是什么命名和位置，其中的cs和dll可会被识别加载。

#### 插件（Plugin）

1. 插件在Alife中代表{插件目录}中的一个文件夹，并且有在插件市场进行注册。所以通常你可以在这些文件夹中找到他们的注册清单（plugin.json），以及一个安装版本标记（VERSION.txt）
2. 插件是用于分发模块的载体，且由于注册信息的存在，额外支持安装环境依赖的能力。因此当你需要接入第三方包时，请使用插件注册依赖信息，这还可以保证多插件见的依赖能够兼容
3. 插件本身只用于分发和依赖描述，不负责编译。编译流程实际是 PluginMarketService 在插件变动后，将环境信息传递给 ModuleSystem，然后 ModuleSystem 将 {插件目录} 整体性的编译加载。

### 环境目录

项目中的特殊目录一般由下构成。这些目录存放着各种配置信息，是开发过程中用来从外部修改软件数据的最佳途径。

- 应用目录（客户端本身的安装目录）：{{AppContext.BaseDirectory}}
- 存储目录（存储角色数据、插件配置等）：{{AlifePath.StorageFolderPath}}
- 运行时目录（存储python等运行时环境）：{{AlifePath.RuntimeFolderPath}}
- 插件目录（插件源码目录）：{存储目录}/Plugins
- 角色目录（对于每个角色的配置、记忆、个人文件等）：{存储目录}/Character
- 模块配置目录（当模块使用配置功能时，配置的存储目录）：{存储目录}/Configuration
- 特定于角色的模块配置目录（优先级比全局高）：{角色目录}/Configuration

### 开发步骤

1. 翻阅插件目录，确定已有插件，并参考其中插件的实现。
2. 在插件目录新增cs脚本，实现插件模块。然后通过 {{nameof(ReloadModules)}} 尝试重新编译加载。
3. 重载成功后，检查并修改角色配置文件`{角色目录}/index.json`，将要启用的模块类名放到`Modules`数组中（具体参考文件中的其他模块放法）。
4. 角色配置文件编辑完成后通过 {{nameof(ReloadCharacters)}} 重新加载角色信息，并用 {{nameof(GetCharacterEnabledModule)}} 来验证模块配置成功。
5. 如果给模块增加了配置功能，那么可能需要额外编辑模块配置文件，其以`{模块类名}.json`的形式存放在{模块配置目录}中，可以参考该文件的其他配置写法。
6. 通过 {{nameof(RestartActivity)}} 重启对话活动，模块将在重启后生效。

### 示例代码

创建一个简单的模块，实现生成随机数功能

```csharp
using System;
using System.ComponentModel;
using System.Threading.Tasks;
using Alife.Framework;
using Alife.Function.FunctionCaller;
using Alife.Function.Interpreter;
using Microsoft.Extensions.Logging;

public class MyModuleData
{
    [Description("随机的最大范围")]//默认的配置UI可以识别并向用户显示Description内容
    public int DefaultMax { get; set; } = 120;
}

[Module(
    "我的功能模块", "一个示例功能模块"
    // EditorUI = typeof(MyModuleUI) /*如果需要，可以用razor自定义模块界面，具体参考官方插件。否则默认使用预设的表单UI（注意：razor不支持热编译，你需要将其转为g.cs或dll）*/
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
        //备注：Poke最终是通过ChatBot来与AI交互的，这是一个非常重要的类，如果要从根源上处理交互和上下文，就去获取ChatBot对象

        logger.LogInformation($"调用 {nameof(Rand)} 结果 {value}");//支持依赖注入的Logger

        return Task.CompletedTask;//如果有需要你可以使用异步代码
    }

    public MyModuleData? Configuration { get; set; }

    public override async Task AwakeAsync(AwakeContext context)
    {
        await base.AwakeAsync(context);

        //将模块注册为xml处理器，以支持文档化和xml调用
        XmlHandler xmlHandler = new(this) {
            Description = "此服务可以为你提供一个生成随机数的功能。",
        };
        functionService.RegisterHandler(xmlHandler);
        //备注：xml函数调用还支持多次注册方式和额外功能，需要复杂的函数调用和注册机制，请查阅Alife.Function.FunctionCaller插件
    }
}
```

### 注意事项

1. 如果开发过程遇到问题，一定要参考同文件夹其他文件的写法，或翻阅源码。这不仅可以纠正你的错误，而且可以让你学到各种与众不同的思路解法，很可能你的需求，在别的插件里早有解决办法。
2. 开发时不要和用户进行对话，要以最快的速度执行开发任务，同时保持专注，避免接收无关通知消息的干扰，专心开发直到需求完成。
