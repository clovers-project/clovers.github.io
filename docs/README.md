# 概述

## clovers

_✨ 高度自定义的聊天平台 Python 异步机器人指令-响应插件框架 ✨_

<img src="https://img.shields.io/badge/python-3.12+-blue.svg" alt="python">
<a href="./LICENSE">
  <img src="https://img.shields.io/github/license/KarisAya/clovers.svg" alt="license">
</a>
<a href="https://pypi.python.org/pypi/clovers">
  <img src="https://img.shields.io/pypi/v/clovers.svg" alt="pypi">
</a>
<a href="https://pypi.python.org/pypi/clovers">
  <img src="https://img.shields.io/pypi/dm/clovers" alt="pypi download">
</a>

## 使用注意事项

1. **请确保你的 Python 版本 >= 3.12**
2. 在响应器中使用到的额外参数都需要在注册响应器时声明
3. 除了指令字符串，其他所有事件信息都是额外参数，需要自定义获取方法。
4. clovers 无法单独使用，需要寄生在其他框架或循环中，比如 [NoneBot2](https://nonebot.dev/)

## 安装

<details open>
<summary>pip</summary>

```bash
pip install clovers
```

</details>

<details>
<summary>poetry</summary>

```bash
poetry add clovers
```

</details>

# 快速开始

以 Nonebot2 框架作为宿主和 nonebot-plugin-clovers 插件为例

创建一个 NoneBot 项目后，安装 [clovers 插件加载器](https://github.com/clovers-project/nonebot-plugin-clovers)

在 nb 项目文件夹下 clovers.toml 中填写配置

```toml
# clovers.toml

[nonebot_plugin_clovers]
plugins_list = [ "clovers_leafgame",]
```

在 plugins_list 配置填写所加载插件的包名

在 clovers_library 文件夹下放入本地 clovers 插件

即可使用。

## 发生了什么

clovers 插件加载器本质是一个 Clovers 实例和一些预定义的 clovers.adapter.Adapter 实例

通过 NoneBot2 的响应器获取指令使 clovers 实例内插件响应

clovers 的理念是完全的自定义，所以当然，如果 nonebot-plugin-clovers 插件无法满足你的需求，你也可以自行编写[适配器方法](#Adapter)

# 如何编写插件？

如果你不做特殊处理，每个插件（即上述 plugins_list 填写的包名或 clovers_library 文件夹下的本地文件）只会向 Clovers 实例添加一个 Plugin 实例

关于 Plugin 类的详细介绍可以参考[文档](/document#plugin.Plugin)

## 开始编写插件

你需要编写一个模块，这个模块需要包含一个`__plugin__`属性，这个属性是一个 clovers.core.plugin.Plugin 类的实例

插件加载器会尝试获取你的模块的`__plugin__`属性，并作为插件放进适配器的插件列表里

如下

```python
from clovers.core.plugin import Plugin

plugin = Plugin() # 创建 Plugin 实例

__plugin__ = plugin # 暴露 __plugin__ 属性
```

你可以通过 Plugin 实例创建多个指令-响应任务，当 clovers 运行时，这些任务就会生效

```python
# 启动时的任务
@plugin.startup
async def _():
    pass

# 关闭时的任务
@plugin.shutdown
async def _():
    pass

# 指令-响应任务
@plugin.handle(["测试"])
async def _(event: Event):
    pass
```

你也可以使用其他插件的`__plugin__`属性添加响应

```python
from some_plugin import __plugin__ as plugin

# do something
@plugin.handle({"其他测试"})
async def _(event: Event):
    pass
```

## 创建 Plugin 实例的参数

| 参数名       | 类型                             | 描述                                   |
| ------------ | -------------------------------- | -------------------------------------- |
| name         | str                              | 插件名称                               |
| priority     | int                              | 插件优先级                             |
| block        | bool                             | 如果本插件有响应，是否阻断后续插件触发 |
| build_event  | Optional[Callable[[Event],Any]]  | 构建 event 的方法                      |
| build_result | Optional[Callable[[Any],Result]] | 构建 result 的方法                     |

## 任务和事件参数

`startup` 在插件初始化时执行，无参数。

`shutdown` 在插件关闭时执行，无参数。

`handle` 指令响应任务，由指令触发，获取到事件参数 `event` ， `event`并不是某个特定的类的实例。而是原始 `Event` 类的实例发送给 `build_event` 构建的返回值。

由此可见原始 `Event` 类你在指令响应任务的函数中唯一获得的参数，你需要的所有东西都在这里。

关于 Event 类的详细介绍可以参考[文档](/document#plugin.Event)

```python
#使用 "你好世界" 触发响应
@plugin.handle(["你好"])
async def _(event: Event):
    print(event.raw_command) # "你好世界"
    print(event.args) # ["世界"]
```

## 指令-响应任务的指令

触发任务的指令可以是字符串，也可以是字符串列表，也可以是一个 re.Pattern 实例

当指令是字符串列表时，handle 装饰器会认为这个指令是一个指令列表，那么它会对字符串进行 startswith 判断。

如果成功触发响应，那么 event.args 会是原指令去掉指令部分后的字符串并按照空格分割为列表。

```python
#触发指令为"你好 世界"时，输出 ["世界"]
#触发指令为"helloworld with extra args"时，输出 ["world","with","extra","args"]
@plugin.handle(["你好","hello"])
async def _(event: Event):
    print(event.args)
```

当指令是字符串时，handle 装饰器会认为这个指令是一个正则字符串，那么它会对指令进行正则匹配。

如果成功触发响应，那么 event.args 会是正则字符串中的 group 列表

```python
#触发指令为"i love you"时，输出 ["i "," you"] 使用时注意去掉参数里的空格
#触发指令为"you love me"时,输出 ["you "," me"]
#触发指令为"make love"时,输出 ["make ", None]
@plugin.handle(r"^(.+)love(.*)")
async def _(event: Event):
    print(event.args)
```

## 指令-响应任务的响应

指令-响应任务函数的返回值可以是任意类型,这个返回值会发送给 build_result 方法构建成 Result 类的实例。

如果你的插件的 build_result is None 那你就必须返回一个 Result 类的实例。

就像你的 build_event is None ,你的参数会是原始的 Event 类的实例那样。

关于 Result 类的详细介绍可以参考[文档](/document#plugin.Result)

接下来的示例是指令为 "测试" 回应 "你好" 的 插件指令-响应任务

```python
@plugin.handle({"测试"})
async def _(event: Event):
    return Result("text", "你好")
```

## 指令-响应任务获取平台参数

如果你在插件中需要获取一些平台参数，那么需要在注册 plugin.handle 时事先声明需要的参数

```python
@plugin.handle(["测试"], extra_args=["user_id","others"])
async def _(event: Event):
    print(event.kwargs["user_id"])
    print(event.kwargs["others"])
    print(event.kwargs["extra"]) # KeyError
```

适配器方法会根据你需要的参数构建 event.kwargs

有时为了优化，你不需要在每次执行任务时都使用某个参数，你也可以声明获取这个参数的方法，在任务中获取。

声明获取参数的方法获取到的值是一个不需要参数的异步函数，存在 get_kwargs 的相应字段里。

异步函数的返回值与参数声明中获取的值完全相同。

```python
@plugin.handle(["测试"],["user_id"], get_extra_args = ["others"])
async def _(event: Event):
    print(event.kwargs["user_id"])
    # print(event.kwargs["others"]) # KeyError
    if condition:
        others = await event.get_kwargs["others"]()
```

## 指令-响应任务的规则

```python
@plugin.handle(
    ["其他功能"],
    extra_args=["to_me"],
    rule=lambda e: e.kwargs["to_me"],
    priority=10,
    block=False,
)
async def _(event: Event):
    pass
```

- rule 是参数为 event，返回值为 bool 的函数或此类函数的列表。
- 如果是函数列表，则所有函数的检查都通过才会触发任务
- 需要注意的是，传给 rule 的参数是 build_event 的返回值，并不是原始的 Event 类实例，除非你的 build_event is None

## 临时任务

```python
@plugin.temp_handle("temp_handle1", 30, ["user_id", "group_id"])
async def _(event: Event, finish):
  if i_should_finish:
    finish()
```

临时任务没有优先级，而且是最优先触发。

临时任务可以在任务中注册,在注册时需要传入一个字符串 `key` 作为这个临时任务的键 。

**注意，一般任务也可以在任务中注册，但是不会生效并且可能导致未定义的行为**

`key` 临时任务 key 如果这个 key 被注册过，并且没有超时也没有结束，那么之前的任务会被下面的任务覆盖

`timeout` 任务超时时间（秒）

temp_handle 会被任意消息触发，请传入规则或在响应函数中内置规则。

temp_handle 任务除了 event，你还会获得一个 Callable 参数 finish，它的功能是结束本任务。如果你不结束，在临时任务超时前每次消息都会触发。

**关于 handle 任务的指令格式和参数列表**

set 格式：合集内的指令都会触发插件

## 配置文件

配置文件存放在一个 toml 文件里，文件由你指定

下面是配置一个例子

clovers.toml

```toml
[clovers_AIchat]
nickname = "小叶子"
timeout = 600
```

意味着 clovers 会加载`./clovers/plugins`文件夹下的文件或文件夹作为插件（排除`_`开头的文件）

插件获取的配置会是一个字典。

为便于插件间的配置互相获取，建议在插件中使用类似下面的代码加载配置

```python
from clovers.core.config import config as clovers_config
config_key = __package__ # 或者你自定义的任何key
default_config = {"some_config_name":"some_config_value"}
# 各种方法获取配置
config_data = clovers_config.get(config_key, {})
default_config.update(config_data)
# 把配置存回总配置
clovers_config[config_key] = config_data
```

## 关于适配器

创建一个适配器

```python
adapter = Adapter()
```

~~创建好了~~

一个适配器可以有多个适配器方法

适配器的所有方法都需要自己写

如果你想使用 clovers 框架，需要使用你接收到的纯文本消息触发适配器响应

像这样

```python
#假设你在一个循环里不断轮询收发消息端是否有新消息
while True:
    command = received_plain_text()
    if command:
        await adapter.response(adapter_key, command, **kwargs)
```

`adapter_key` 适配器方法指定的 key
`kwargs` 适配器方法需要的所有参数

### 适配器方法

获取参数，发送信息的方法。里面所有的方法都需要自己写

发送信息，获取参数

```python
# 假如收发信息框架提供了如下方法

# send_plain_text(text:str)发送纯文本

method = AdapterMethod()
@method.send("text")
async def _(message: str):
    send_plain_text(message)

# send_image(image:bytes) 发送图片，但是需要response的参数

@method.send("image")
async def _(message: bytes,send_image):
    send_image(message)

# sender发送消息的用户信息，通过response的参数传入
# 假设有 sender.user_id 属性为该用户uid
@method.kwarg("user_id")
async def _(sender):
    return sender.user_id

# 注入适配器方法
adapter.methods["my_adapter_method"] = method
```

使用上述适配器

你的 `Result("text", "你好")` 会使用 send_plain_text 发送

你的指令响应任务获取平台参数的 `"user_id"` 就是 sender.user_id

### 使用插件加载器 PluginLoader 向适配器注入插件

```python
loader = PluginLoader(plugins_path, plugins_list)
adapter.plugins = loader.plugins
```

`plugins_list` 插件名列表,例如["plugin1","plugin2"]。从 python lib 路径下的包名加载插件

`plugins_path` 插件文件夹，加载改路径下的文件或文件夹作为插件（排除`_`开头的文件）

或者

```python
plugin = PluginLoader.load("plugin1")
if not plugin is None:
    adapter.plugins.append(plugin)
```

### 开始，结束任务

一些插件会注册一些开始时，结束时运行的任务

所以你需要在开始时，或结束时执行

```python
asyncio.create_task(adapter.startup)
asyncio.create_task(adapter.shutdown)
```

或类似作用的代码

## 📞 联系

如有建议，bug 反馈等可以加群

机器人 bug 研究中心（闲聊群） 744751179

永恒之城（测试群） 724024810

![群号](https://github.com/KarisAya/clovers/blob/master/%E9%99%84%E4%BB%B6/qrcode_1676538742221.jpg)

## 💡 鸣谢

- [nonebot2](https://github.com/nonebot/nonebot2) 跨平台 Python 异步聊天机器人框架 ~~需求都是基于这个写的~~
