<div align="center">

# CLOVERS

_✨ 自定义的聊天平台异步机器人指令-响应插件框架 ✨_

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
</div>

## 💿 安装

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

## 插件获取配置

配置文件存放在一个 toml 文件里，文件由你指定

下面是配置一个例子

clovers.toml

```toml
[nonebot_plugin_clovers]
plugins_path = "./clovers/plugins"
plugins_list = ["clovers_apscheduler"]
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

当然你也可以不这么做

## 关于插件

下面是一个模板

```python
from clovers.core.config import config as clovers_config
from clovers.core.plugin import Plugin, Event
from .config import Config

# 获取你的配置
config_key = __package__
config_data = Config.parse_obj(clovers_config.get(config_key, {}))
clovers_config[config_key] = config_data.dict()

plugin = Plugin()

# 启动时的任务
@plugin.startup
async def _():
    pass

# 关闭时的任务
@plugin.shutdown
async def _():
    pass


# 指令-响应任务
@plugin.handle({"测试"})
async def _(event: Event):
    pass

__plugin__ = plugin
```

插件加载器会尝试获取你的模块的`__plugin__`属性，并作为插件放进适配器的插件列表里

如果你想编写插件的插件，也可以不定义`__plugin__`属性,但是一般你需要使用其他插件的`__plugin__`属性

```python
from some_plugin import __plugin__ as plugin

# do something
@plugin.handle({"其他测试"})
async def _(event: Event):
    pass
```

### 指令-响应任务获取平台参数

如果你在插件中需要获取一些平台参数，那么需要在注册 plugin.handle 时事先声明需要的参数

```python
@plugin.handle({"测试"},{"user_id","others"})
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
@plugin.handle({"测试"},{"user_id"}, get_extra_args = ["others"])
async def _(event: Event):
    print(event.kwargs["user_id"])
    print(event.kwargs["others"]) # function
    others = await event.kwargs["others"]()
```

### 指令-响应任务中的 event

event 是你在指令-响应任务的函数中唯一获得的参数，你需要的所有东西都在 event 里

`raw_command` 触发本次响应的原始字符串
`args` 解析的参数列表

```python
#使用 "你好世界" 触发响应
@plugin.handle({"你好"})
async def _(event: Event):
    print(event.raw_command) # "你好世界"
    print(event.args) # ["世界"]
```

如果你不想使用原始的 event,你也可以自建 event 类,然后在创建 plugin 实例时注入 build_event 方法。

```python
from clovers.core.plugin import Plugin
class Event:
    def __init__(self, event: CloversEvent):
        self.event: CloversEvent = event

    @property
    def raw_command(self):
        return self.event.raw_command

    @property
    def args(self):
        return self.event.args

    @property
    def user_id(self) -> str:
        return self.event.kwargs["user_id"]

plugin = Plugin(build_event=lambda event: Event(event))

@plugin.handle({"测试"},{"user_id"})
async def _(event: Event):
    print(event.user_id) # "123456"
```

### 插件的响应

响应的格式应该是 clovers.core.plugin.Result 类

`send_method` 控制适配器方法用什么方式发送你的数据

`data` 是要发送的原始数据

接下来的示例是指令为 "测试" 回应 "你好" 的 插件指令-响应任务

```python
@plugin.handle({"测试"})
async def _(event: Event):
    return Result("text", "你好")
```

当然如果你认为这样太过繁琐，你也可以使用 build_result 方法

```python
from clovers.core.plugin import Plugin
def build_result(result):
    if isinstance(result, str):
        return Result("text", result)
    if isinstance(result, BytesIO):
        return Result("image", result)
    if isinstance(result, AnyTypeYouNeed):
        return Result("any_method_you_want", result)
    return result

plugin = Plugin(build_result=build_result)

@plugin.handle(["测试"])
async def _(event: Event):
    return "你好"
```

### 关于插件的其他功能

**临时任意触发任务**

```python
@plugin.temp_handle("temp_handle1", 30, ["user_id", "group_id"])
async def _(event: Event, finish):
  if i_should_finish:
    finish()
```

需要的三个参数

`key` 临时任务 key 如果这个 key 被注册过，并且没有超时也没有结束，那么之前的任务会被下面的任务覆盖

`extra_args` 需要的平台参数

`timeout` 任务超时时间（秒）

temp_handle 会被任意消息触发，请在任务内自定义检查规则。

temp_handle 任务除了 event，你还会获得一个 Callable 参数 finish，它的功能是结束本任务。如果你不结束，在临时任务超时前每次消息都会触发。

**关于 handle 任务的指令格式和参数列表**

set 格式：合集内的指令都会触发插件

```python
#触发指令为"你好 世界"时，输出 ["世界"]
#触发指令为"hello1 world with extra args"时，输出 ["1","world","with","extra","args"]
@plugin.handle({"你好","hello"})
async def _(event: Event):
    print(event.args)
```

字符串格式：正则匹配

如果 handle 的指令参数是字符串那么它会进行正则匹配，args 会是正则字符串中的 group 列表

```python
#触发指令为"i love you"时，输出 ["i "," you"] 使用时注意去掉参数里的空格
#触发指令为"you love me"时,输出 ["you "," me"]
#触发指令为"make love"时,输出 ["make ", None]
@plugin.handle(r"^(.+)love(.*)")
async def _(event: Event):
    print(event.args)
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
