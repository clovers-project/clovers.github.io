# Clovers

**属性：**

`global_adapter` 全局适配器，一个特殊的 Adapter 实例。如果当前工作的适配器没有特定的方法时，会尝试从此适配器中获取方法。

`plugins` 插件列表，在运行 clovers 实例前载入。

`adapter_dict` 适配器字典，key 为适配器名称，value 为适配器实例。

`plugins_dict` 适配器-插件列表字典，key 为适配器名称，value 为插件列表。

    - 不需要在外部修改。在实例启动时会判断当前适配器是否有此插件声明的全部方法，如果没有，则此适配器不会响应该插件。
    - 因为临时任务声明的参数是动态的，所以如果临时任务声明的方法不存在，则会引发异常而不是不响应。
    - 因为响应没有声明，所以如果返回的 Result 发送方法不存在，则会引发异常。

`wait_for` 异步任务列表，在关闭任务时需等待此任务列表

`running` 是否正在运行

## response

对指令进行响应

**返回值：**

`int` ,表示此指令触发了多少个响应

**参数：**

`adapter_key` 对指令进行响应的适配器键

`command` 指令

`**extra` 适配方法需要的额外参数

## load_plugin

加载插件

**返回值：**

`None`

**参数：**

`name` 插件名,可以是当前进程路径下的一个包名或者 Python lib 中的一个模块名。

## startup

启动 clovers

**返回值：**

`None`

**参数：**

`None`

## shutdown

关闭 clovers

**返回值：**

`None`

**参数：**

`None`

# plugin

## plugin.Result

`send_method` 控制适配器方法用什么方式发送你的数据。

`data` 要发送的原始数据。

## plugin.Event

`raw_command` 触发本次响应的原始字符串。

`args` 解析的参数列表。

`kwargs` 一个字典。包含了一些平台特有的参数。

`get_kwargs` event 的一个属性，包含了一些平台特有的参数的获取方法。

## plugin.Handle

`func` 指令响应器的异步函数本体，参数为 plugin.Event 类型，返回值为 plugin.Result 类型。

`extra_args` 指令响应器声明的参数。

`get_extra_args` 指令响应器声明的参数获取方法

`block` 是否阻断插件内后续任务触发。

## plugin.PluginCommands

`str | Iterable[str] | re.Pattern | None` 的类型别名，是指令注册器可接受的参数类型。

## plugin.Plugin

plugin.Plugin 类是插件的核心，也是你编写插件的入口

`name` 插件名

`priority` 插件优先级

`block` 是否阻断后续其他插件触发。

`build_event` event 的构建方法

如果你不想在任务函数内使用原始的 event,你也可以自建 event 类,然后在创建 plugin 实例时注入 build_event 方法。

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

`build_result` result 的构建方法

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

`temp_handles` 临时任务列表，类型为字典，key 是临时任务键（注册时传入） value 是注册时的浮点数时间戳和 plugin.Handle 实例组成的元组

### ready

准备插件。Clovers 实例启动时会对每个插件都调用一次 ready 方法。

**返回值：**

`None`

**参数：**

`None`

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

## plugin.Plugin.Rule

## plugin.PluginLoader

`plugins_list` 插件名列表,例如["plugin1","plugin2"]。从 python lib 路径下的包名加载插件

`plugins_path` 插件文件夹，加载改路径下的文件或文件夹作为插件（排除`_`开头的文件）
