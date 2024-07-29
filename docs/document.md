## plugin.Plugin

Plugin 类是插件的核心，也是你编写插件的入口

你可以通过 Plugin 实例创建多个指令-响应任务，当 clovers 运行时，这些任务就会生效

```python
from clovers.core.plugin import Plugin

plugin = Plugin()

__plugin__ = plugin
```

### build_event

如果你不想使用原始的 event,你也可以自建 event 类,然后在创建 plugin 实例时注入 build_event 方法。

```python
from clovers.core.plugin import Plugin, Event as CloversEvent
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

@plugin.handle(["测试"],["user_id"])
async def _(event: Event):
    print(event.user_id) # "123456"
```

### build_event

当然如果你认为返回 Result 实例太过繁琐，你也可以使用 build_result 方法

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

### handle

`command` 触发任务的指令。

`extra_args` 声明需要的参数。

`get_extra_args` 声明需要的参数，包装成异步函数。

`rule` 响应的触发的规则。

`priority` 插件内部指令-响应任务的优先级。

`block` 如果本任务有响应，是否阻断插件内后续任务触发。

### temp_handle

`key` 临时任务 key 如果这个 key 被注册过，并且没有超时也没有结束，那么之前的任务会被下面的任务覆盖

`extra_args` 声明需要的参数。

`get_extra_args` 声明需要的参数，包装成异步函数。

`timeout` 任务超时时间（秒）

`rule` 响应的触发的规则。

`block` 如果本任务有响应，是否阻断插件内后续任务触发。

## plugin.PluginLoader

`plugins_list` 插件名列表,例如["plugin1","plugin2"]。从 python lib 路径下的包名加载插件

`plugins_path` 插件文件夹，加载改路径下的文件或文件夹作为插件（排除`_`开头的文件）

## plugin.Event

`raw_command` 触发本次响应的原始字符串。

`args` 解析的参数列表。

`kwargs` 一个字典。包含了一些平台特有的参数。

`get_kwargs` event 的一个属性，包含了一些平台特有的参数的获取方法。

## plugin.Result

`send_method` 控制适配器方法用什么方式发送你的数据。

`data` 要发送的原始数据。
