.. _guide-app:

=============
 实例
=============

.. contents::
    :local:
    :depth: 1

Celery 在使用前必须被实例化，而这个实例被称之为 Celery 应用实例 (或简称为 *app*)。

Celery 实例是线程安全的，因此具有不同配置、组件及任务的多个 Celery 应用
能够在同一进程空间中共存。

让我们先动手创建一个实例：

.. code-block:: pycon

    >>> from celery import Celery
    >>> app = Celery()
    >>> app
    <Celery __main__:0x100469fd0>

最后一行展示了 Celery 实例的文本表示的内容：
其中包含了该 Celery 实例的名称 (``Celery``), 
主模块的名称 (``__main__``), 
以及该对象的内存地址 (``0x100469fd0``).

主模块名称
==========

上一小节打印出来的内容中，只有主模块名称是最重要的，原因如下：

当你向 Celery 发送一条执行某个任务的消息时，这条消息中不会包含任务的源码，消息中只会带有任务的
名称。就像网络中的主机名一样：每个 worker 都会维护一个任务名到其定义函数的映射，这个映射被称为
*任务注册表（task registry）*。

每当你定义一个任务，Celery 就会将该任务添加到本地的注册表（local registry）中：

.. code-block:: pycon

    >>> @app.task
    ... def add(x, y):
    ...     return x + y

    >>> add
    <@task: __main__.add>

    >>> add.name
    __main__.add

    >>> app.tasks['__main__.add']
    <@task: __main__.add>

在上面的例子中，我们再次看到了 ``__main__``，当 Celery 无法检测该任务属于哪个模块时，其
会使用主模块的 name 来生成任务的名称。

这种问题只会在下面两种情况中出现：

    #. 定义任务的模块当前正在被当做主模块运行
    #. 当前 Celery 实例是在 Python shell（REPL） 环境下创建的

在下面的例子中，定义 task 的模块同样用于调用 :meth:`@worker_main`: 来启动 worker：

:file:`tasks.py`:

.. code-block:: python

    from celery import Celery
    app = Celery()

    @app.task
    def add(x, y): return x + y

    if __name__ == '__main__':
        app.worker_main()

当该模块以主模块来运行时，其中定义的 task 名称都会以 "``__main__``" 开头，但是，当该模块
是被在别的模块中被导入时，其中定义的 task 的名称就会以 "``tasks``"（task 所在模块的真实名称）
开头命名：

.. code-block:: pycon

    >>> from tasks import add
    >>> add.name
    tasks.add

当然你也可以为更改主模块的名称，示例如下：

.. code-block:: pycon

    >>> app = Celery('tasks')
    >>> app.main
    'tasks'

    >>> @app.task
    ... def add(x, y):
    ...     return x + y

    >>> add.name
    tasks.add

.. seealso:: :ref:`task-names`

配置
=============

Celery 提供了一些选项来让用户定义其行为，你可以通过直接设置实例的属性来修改这些选项，
也可以使用特定的配置模块来改变这些选项。

通过访问实例的 :attr:`@conf` 对象获取 Celery 当前的配置：

.. code-block:: pycon

    >>> app.conf.timezone
    'Europe/London'

你也可以直接修改该对象的属性值来修改配置：

.. code-block:: pycon

    >>> app.conf.enable_utc = True

或者通过 ``update`` 方法来一次性更新多个配置项：

.. code-block:: python

    >>> app.conf.update(
    ...     enable_utc=True,
    ...     timezone='Europe/London',
    ...)

配置对象中包含多个字典对象，这些字典存储了来源不同的配置选项，Celery 会按照以下的顺序来读取某个配置的值：

    #. 运行时修改
    #. 配置模块（如果存在的话）
    #. 默认配置（:mod:`celery.app.defaults`）

而且，Celery 允许你通过 :meth:`@add_defaults` 方法来添加新的默认配置源。

.. seealso::

    到 :ref:`Configuration reference <configuration>` 中获取所有
    可用配置选项及其默认值和详细信息。

``config_from_object``
----------------------

:meth:`@config_from_object` 能够从某个配置对象中加载配置。

这个对象可以是一个模块或是包含有配置属性的任意对象。

需要注意的是，任何在 :meth:`~@config_from_object` 方法调用前设置的配置都会
被重置，如果你需要设置一些额外的配置选项，请在调用该方法后进行。

样例 1: 通过模块名进行配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:meth:`@config_from_object` 方法接收 Python 模块的全限定名，或是 Python 属性
名，例如 ``"celeryconfig"``, ``"myproj.config.celery"``, or
``"myproj.config:CeleryConfig"``：

.. code-block:: python

    from celery import Celery

    app = Celery()
    app.config_from_object('celeryconfig')

``celeryconfig`` 模块定义如下：

:file:`celeryconfig.py`:

.. code-block:: python

    enable_utc = True
    timezone = 'Europe/London'

只要 ``import celeryconfig`` 是合法的，Celery 实例就能够使用该模块下的配置。

样例 2: 通过模块对象进行配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

你也可以通过传递一个已经导入的模块对象来进行配置，但这种方式是不推荐的。

.. tip::

    推荐使用模块名来进行配置是因为在使用 prefork pool 时，使用模块名称配置实例的方式
    不会去序列化该模块。如果你遇到了配置问题或是 pickle 错误，请尝试使用模块名进行配置。

.. code-block:: python

    import celeryconfig

    from celery import Celery

    app = Celery()
    app.config_from_object(celeryconfig)


样例 3: 通过类/对象来进行配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    from celery import Celery

    app = Celery()

    class Config:
        enable_utc = True
        timezone = 'Europe/London'

    app.config_from_object(Config)
    # or using the fully qualified name of the object:
    #   app.config_from_object('module:Config')

``config_from_envvar``
----------------------

:meth:`@config_from_envvar` 方法会从环境变量中获取配置模块。

下面的样例展示了如何从名为 :envvar:`CELERY_CONFIG_MODULE`: 的环境变量中加载配置：

.. code-block:: python

    import os
    from celery import Celery

    #: Set default configuration module name
    os.environ.setdefault('CELERY_CONFIG_MODULE', 'celeryconfig')

    app = Celery()
    app.config_from_envvar('CELERY_CONFIG_MODULE')

之后你便能够通过环境变量来指定当前 Celery 实例要加载的配置模块：

.. code-block:: console

    $ CELERY_CONFIG_MODULE="celeryconfig.prod" celery worker -l info

.. _app-censored-config:

配置中的敏感信息
----------------------

有时候处于调试的目的你可能会将配置选项的值打印出来，但与此同时，你也想从中过滤掉一下敏感
信息：如密码、API 秘钥、私钥等。

Celery 提供了若干个用于展示配置选项的工具函数，其中一个是 :meth:`~celery.app.utils.Settings.humanize`：

.. code-block:: pycon

    >>> app.conf.humanize(with_defaults=False, censored=True)

该方法会将配置选项以字符串列表的形式返回，默认情况下 :meth:`~celery.app.utils.Settings.humanize`：
只会返回对默认值进行了修改的配置，若要将所有配置都返回，需要将 ``with_defaults`` 参数设置为 ``True``。

如果你需要将配置信息以字典的形式进行展示，可以选择 :meth:`~celery.app.utils.Settings.table` 方法：

.. code-block:: pycon

    >>> app.conf.table(with_defaults=False, censored=True)

请注意，Celery 无法剔除掉配置中的所有敏感信息，因为其是通过正则表达式来搜索敏感信息的一些常用命名。如果
你希望 Celery 绑定剔除掉配置中的敏感信息，你应该根据 Celery 匹配敏感信息的规则来为这些配置项命名。

如果配置名称中包含以下字符串，则 Celery 会认为这些配置中包含敏感信息：

``API``, ``TOKEN``, ``KEY``, ``SECRET``, ``PASS``, ``SIGNATURE``, ``DATABASE``

懒加载
========

Celery 实例是懒加载的，这意味中不到真正要使用到它的时候，岂不会被加载：

创建一个 :class:`@Celery` 实例时会完成以下工作：

    #. 创建一个逻辑时钟对象，用于监听事件
    #. 创建一个任务注册表（task registry）
    #. 将该实例设置成当前实例（current app），如果 ``set_as_current`` 参数
       为 ``False``，这一步不会执行
    #. 调用 :meth:`@on_init` 进行回调（该方法默认实现为空）

:meth:`@task` 装饰器并不会在任务定义时就创建该任务，而是将其推迟到任务被使用时
或是实例最终创建完成后（*finalized*）。

下面的例子很好的说明了在你使用任务或是访问其属性（例子中访问的是 :meth:`repr`）前，该任务都不会被创建：

.. code-block:: pycon

    >>> @app.task
    >>> def add(x, y):
    ...    return x + y

    >>> type(add)
    <class 'celery.local.PromiseProxy'>

    >>> add.__evaluated__()
    False

    >>> add        # <-- causes repr(add) to happen
    <@task: __main__.add>

    >>> add.__evaluated__()
    True

通过调用 :meth:`@finalize` 方法或是显式的访问 :attr:`@tasks` 属性能够完成实例的最终创建（*Finalization*）。

这个过程会完成以下工作：

    #. 复制必须在 app 之间进行共享的任务

        所有的任务默认都是可共享的，但如果 task 装饰器
        中的 ``shared`` 参数被设置为 ``False``，该任务
        就会成为其绑定的 Celery 实例（app）的私有任务
    
    #. 执行所有未执行的 task 装饰器

    #. 确保所有任务都绑定到当前实例上（current app)

        将任务绑定到实例上能够让其读取配置中的默认值。

.. _default-app:

.. topic:: The "default app"

    Celery 并不总是需要创建一个实例，因为旧的版本只有模块层的 API，为了向
    后兼容，这些旧的 API 一直都会存在，这些旧的 API 会保留到 Celery 5.0。

    Celery 每次都会创建一个默认的实例，如果用户没有初始化自定义的应用实例，Celery
    就会使用这个默认的实例。

    :mod:`celery.task` 用于兼容旧的 API，如果你要使用自定义应用程序，则不应该
    使用该模块中提供的 API。正确的做法是使用实例中定义的方法。

    旧的 Task 基类提供了许多用于兼容老版本的功能，其中某些功能可能会与新版本的功能
    不兼容，两种 Task 的导入方式如下所示：

    .. code-block:: python

        from celery.task import Task   # << OLD Task base class.

        from celery import Task        # << NEW base class.

    即使你再使用旧的模块层 API，我们还是推荐你使用新的基类。


实例链
==================

虽然能够通过当前应用（current_app）这个对象获取当前实例，但最好的方法是通过
参数将当前实例传递给其他对象。

我们将这种行为称之为实例链，因为其根据传递路径形成了一条应用实例链。

下面代码中所示的行为是不推荐的：

.. code-block:: python

    from celery import current_app

    class Scheduler(object):

        def run(self):
            app = current_app

正确的做法是使用 ``app`` 参数将当前实例传入：

.. code-block:: python

    class Scheduler(object):

        def __init__(self, app):
            self.app = app

Celery 内部使用 :func:`celery.app.app_or_default` 函数来获取当前实例来保证
对基于模块的旧 API 的兼容性。

.. code-block:: python

    from celery.app import app_or_default

    class Scheduler(object):
        def __init__(self, app=None):
            self.app = app_or_default(app)

你可以在开发环境中设置 :envvar:`CELERY_TRACE_APP` 环境变量来让 Celery 在实例
链断开时抛出异常：

.. code-block:: console

    $ CELERY_TRACE_APP=1 celery worker -l info


.. topic:: Evolving the API

    Celery 在其诞生至今的七年中发生了很多变化。

    例如，在 Celery 最初的版本中，任何 callable 对象都能够作为任务使用：

    .. code-block:: pycon

        def hello(to):
            return 'hello {0}'.format(to)

        >>> from celery.execute import apply_async

        >>> apply_async(hello, ('world!',))

    或者你也可以根据你的需求来创建一个 Task 类，并在其中重载 Task 的某些
    行为

    .. code-block:: python

        from celery.task import Task
        from celery.registry import tasks

        class Hello(Task):
            queue = 'hipri'

            def run(self, to):
                return 'hello {0}'.format(to)
        tasks.register(Hello)

        >>> Hello.delay('world!')

    后来，我们决定将使用任何 callable 对象来创建任务这一特性列为一种反模
    式（anti-pattern），因为这个特性导致 Celery 难以使用除 pickle 之外
    的序列化方式。这个功能在 2.0 之后就被剔除了，取而代之的是装饰器：

    .. code-block:: python

        from celery.task import task

        @task(queue='hipri')
        def hello(to):
            return 'hello {0}'.format(to)

抽象类任务
==============

所有使用 :meth:`~@task` 装饰器创建的任务都会继承自应用实例中的 :attr:`~@Task`
类。

当然你也可以通过 ``base`` 参数来指定某个任务的父类：

.. code-block:: python

    @app.task(base=OtherTask):
    def add(x, y):
        return x + y

需要注意的是，任何自定义的任务类都需要继承自 :class:`celery.Task`。

.. code-block:: python

    from celery import Task

    class DebugTask(Task):

        def __call__(self, *args, **kwargs):
            print('TASK STARTING: {0.name}[{0.request.id}]'.format(self))
            return super(DebugTask, self).__call__(*args, **kwargs)


.. tip::

    确保你在重载了 Task 类的 ``__call__`` 方法的同时调用其父类的 ``__call__`` 方法，
    因为 Task 类的 ``__call__`` 方法会设置任务被直接调用时使用的默认请求。


Task 基类的特殊之处在与其没有雨任何一个应用实例进行板顶，因为任务一旦绑定到特定的实例后，其
就会该实例中的配置。

通过 :meth:`@task` 装饰器能够对某个任务基类进行实例化：

.. code-block:: python

    @app.task(base=DebugTask)
    def add(x, y):
        return x + y

你也能够通过更改实例的 :meth:`@Task` 属性来改变实例默认的任务基类：

.. code-block:: pycon

    >>> from celery import Celery, Task

    >>> app = Celery()

    >>> class MyBaseTask(Task):
    ...    queue = 'hipri'

    >>> app.Task = MyBaseTask
    >>> app.Task
    <unbound MyBaseTask>

    >>> @app.task
    ... def add(x, y):
    ...     return x + y

    >>> add
    <@task: __main__.add>

    >>> add.__class__.mro()
    [<class add of <Celery __main__:0x1012b4410>>,
     <unbound MyBaseTask>,
     <unbound Task>,
     <type 'object'>]
