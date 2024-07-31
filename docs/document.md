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

**参数：**

`adapter_key` 对指令进行响应的适配器键

`command` 指令

`**extra` 适配方法需要的额外参数

**返回值：**

`int` 表示此指令触发了多少个响应

## load_plugin

加载插件

**参数：**

`name` 插件名,可以是当前进程路径下的一个包名或者 Python lib 中的一个模块名。

**返回值：**

`None`

## startup

启动 clovers

**参数：**

无

**返回值：**

`None`

## shutdown

关闭 clovers

**参数：**

无

**返回值：**

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

**属性：**

`func` 指令响应器的异步函数本体，参数为 plugin.Event 类型，返回值为 plugin.Result 类型。

`extra_args` 指令响应器声明的参数。

`get_extra_args` 指令响应器声明的参数获取方法

`block` 是否阻断插件内后续任务触发。

## plugin.PluginCommands

`str | Iterable[str] | re.Pattern | None` 的类型别名，是指令注册器可接受的参数类型。

## plugin.Plugin

plugin.Plugin 类是插件的核心，也是你编写插件的入口

**属性：**

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

### 直接调用

**参数：**

`message` 类型 `str` 插件需要响应的消息

**返回值：**

`list[tuple[Handle, Event]] | None` 返回此插件响应的任务和需要适配器注入参数的事件列表

元祖内的 Event 实例需要经过适配器处理后才会传入 Handle 实例的 func 方法

### ready

准备插件。Clovers 实例启动时会对每个插件都调用一次 ready 方法。

**参数：**

无

**返回值：**

`bool` 如果当前插件没有响应任务，则返回 False，否则返回 True。

### plugin.Plugin.Rule

**属性：**

`checker` 函数列表

    - 此列表内的函数的返回值必须是布尔值
    - 如果列表内的任意函数返回值为 False，则此任务不会被执行
    - 列表内函数的参数是 build_event 的返回值，如果 build_event 为 None，则此参数为原始 Event 实例

**方法：**

`check` 函数装饰器，为 handle.func 添加检查.

### handle

**参数：**

`command` 触发任务的指令。

`extra_args` 声明需要的参数。

`get_extra_args` 声明需要的参数，包装成异步函数。

`rule` 响应的触发的规则。

`priority` 插件内部指令-响应任务的优先级。

`block` 如果本任务有响应，是否阻断插件内后续任务触发。

**返回值：**

装饰器

### temp_handle

`key` 临时任务 key 如果这个 key 被注册过，并且没有超时也没有结束，那么之前的任务会被下面的任务覆盖

`extra_args` 声明需要的参数。

`get_extra_args` 声明需要的参数，包装成异步函数。

`timeout` 任务超时时间（秒）

`rule` 响应的触发的规则。

`block` 如果本任务有响应，是否阻断插件内后续任务触发。

### plugin.Plugin.Finish

结束临时任务的类，作为参数传入临时任务函数中，表示本任务结束。

直接调用是结束任务，调用 `delay` 方法并传入秒数可以相应延长任务超时时间

下面是示例：

```python

@plugin.temp_handle("test", timeout=10)
async def _(event: Event, finish: plugin.Plugin.Finish):
    # do something
    if condition:
        finish() # 结束任务
    else:
        finish.delay(10) # 不结束任务并且延长任务超时时间

```

### startup

装饰器

注册一个启动任务

### shutdown

装饰器

注册一个结束任务

## plugin.PluginLoader

**属性：**

`plugins_list` 插件名列表,例如["plugin1","plugin2"]。从 python lib 路径下的包名加载插件

`plugins_path` 插件文件夹，加载改路径下的文件或文件夹作为插件（排除`_`开头的文件）

`plugins` 从 self.plugins_list 和 self.plugins_path 加载的插件列表

### load

静态方法

**参数：**

`name` 插件名,可以是当前进程路径下的一个包名或者 Python lib 中的一个模块名。

**返回值：**

`Plugin | None` 如果加载成功则返回插件实例，否则返回 `None`

### plugins_from_path

从 self.plugins_path 加载插件,返回一个插件列表

### plugins_from_list

从 self.plugins_list 加载插件,返回一个插件列表

# adapter

## adapter.Adapter

plugin.Adapter 类是响应器的核心

**属性：**

`kwarg_dict` 关键字参数方法字典

`send_dict` 发送方法字典

### kwarg

注册一个关键字参数方法

**参数：**

`method_name` 参数方法名

**返回值：**

装饰器

### send

注册一个发送方法

**参数：**

`method_name` 发送方法名

**返回值：**

装饰器

### remix

混合适配器方法，此适配器会获得参数适配器存在但自己不存在的方法

**参数：**

`adapter` 适配器实例

**返回值：**

`None`

### kwarg_method

获取一个关键字参数方法
**参数：**

`key` 参数方法名

**返回值：**

kwarg 注册的名为 key 的异步函数

### send_method

**参数：**

`key` 发送方法名

**返回值：**

send 注册的名为 key 的异步函数

### response

获得适配器的响应

**参数：**

`handle` 将要用此适配器响应的指令响应任务

`event` 将要用此适配器处理的事件参数

`**kwargs` 此适配器需要的其它参数，即注册 kwarg 和 send 方法时函数本体传入的参数。

# config

在我们写插件，适配器，或其他任何场景我们需要用到配置时。

除了 json，数据库，乃至系统环境变量等方法外，我们还可以使用此模块内的配置类来进行配置的存取。

配置类是一个字典的子类，所以你可以把它当做字典随意使用。

当然我们也可以让每个插件都遵循一定的规则。

下面是推荐的规则：

```python
from pydantic import BaseModel

class Config(BaseModel):
    # 定义一些属性

from clovers.core.config import config as clovers_config

config_key = __package__ # 或者你自定义的任何配置键名
config_data = Config.model_validate(clovers_config.get(config_key, {})) # 从 clovers_config 获取配置字典并规范成 Config 类型
clovers_config[config_key] = config_data.model_dump() # 将规范配置存回 clovers_config
```

## config.Config

配置类，为字典的子类

**属性：**

`path` 定义调用 save() 方法时配置文件保存在哪个位置。

### load

类方法

**参数：**

`path` 从这个地址加载配置文件，并将其保存到 self.path 属性中

**返回值：**

`Config` 实例

### save

保存配置文件
**参数：**

无

**返回值：**

`None`

## config.config

运行时的配置文件实例

这个实例的文件位置从环境变量中的 `clovers_config_file` 读取，如果没有找到，则使用 `clovers.toml` 作为默认配置文件。

## logger

### logger.logger

默认的日志记录器

```

```
