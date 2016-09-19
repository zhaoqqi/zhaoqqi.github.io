---
layout: post
title: Eventlet(3) - greenlet
categories: [OpenStack, Python]
description: greenlet.
keywords: OpenStack, Python, greenlet
---

### greenlet

以下内容翻译自 greenlet [官方文档](https://greenlet.readthedocs.io/en/latest/)，边翻译边学习。

#### 动机

> The “greenlet” package is a spin-off of Stackless, a version of CPython that supports micro-threads called “tasklets”. Tasklets run pseudo-concurrently (typically in a single or a few OS-level threads) and are synchronized with data exchanges on “channels”.

`greenlet` 是从 stackless 拆分出来的，由 CPython 编译开发，支持名为 `tasklets` 的微线程。
`tasklets` 以伪并发的形式运行(比如，在一个或多个操作系统级别的线程中)，使用 `channels` 同步和交换数据。

> A “greenlet”, on the other hand, is a still more primitive notion of micro-thread with no implicit scheduling; coroutines, in other words. This is useful when you want to control exactly when your code runs. You can build custom scheduled micro-threads on top of greenlet; however, it seems that greenlets are useful on their own as a way to make advanced control flow structures. For example, we can recreate generators; the difference with Python’s own generators is that our generators can call nested functions and the nested functions can yield values too. (Additionally, you don’t need a “yield” keyword. See the example in test/test_generator.py).

换句话说，一个 `greenlet` 是没有隐式调度的微线程(micro-thread)的一种更原始的概念，也可以叫做协同程序(corountine)。这对你精确控制代码何时执行极为有益。你可以通过 `greenlet` 创建自定义调度策略的微线程(micro-thread)。不管怎样，`greenlet` 是一种先进的流控制结构。举例来说，我们可以重复创建 `greenlet` 的生成器，与 Python 原生的生成器的区别是，`greenlet` 的生成器可以调用嵌套的方法，同时嵌套方法可以 `yield` (另外，你不必显示使用关键字 yield。具体参考 `test/test_generator.py` 中的例子)。

> Greenlets are provided as a C extension module for the regular unmodified interpreter.

`greenlets` 是作为C扩展模块来使用。

下面是greenlet的一个简单的栈空间模型(from greenlet.c)

```c
A PyGreenlet is a range of C stack addresses that must be
saved and restored in such a way that the full range of the
stack contains valid data when we switch to it.
 
Stack layout for a greenlet:
 
               |     ^^^       |
               |  older data   |
               |               |
  stack_stop . |_______________|
        .      |               |
        .      | greenlet data |
        .      |   in stack    |
        .    * |_______________| . .  _____________  stack_copy + stack_saved
        .      |               |     |             |
        .      |     data      |     |greenlet data|
        .      |   unrelated   |     |    saved    |
        .      |      to       |     |   in heap   |
 stack_start . |     this      | . . |_____________| stack_copy
               |   greenlet    |
               |               |
               |  newer data   |
               |     vvv       |
```

##### 举例

> Let’s consider a system controlled by a terminal-like console, where the user types commands. Assume that the input comes character by character. In such a system, there will typically be a loop like the following one:

我们想象一个可以由用户输入命令的，由终端控制台操作的系统。假设每次输入单个字符。在这样的系统中，一定有像下面这个循环处理：

```python
def process_commands(*args):
    while True:
        line = ''
        while not line.endswith('\n'):
            line += read_next_char()
        if line == 'quit\n':
            print "are you sure?"
            if read_next_char() != 'y':
                continue    # ignore the command
        process_command(line)
```

> Now assume that you want to plug this program into a GUI. Most GUI toolkits are event-based. They will invoke a call-back for each character the user presses. [Replace “GUI” with “XML expat parser” if that rings more bells to you :-)] In this setting, it is difficult to implement the read_next_char() function needed by the code above. We have two incompatible functions:

现在假设你想将这段程序嵌入一个GUI。大多GUI工具包都是基于事件的。用户每次输入命令系统都会基于输入事件调用回调函数。在这个背景下，上面这段代码很难实现 `read_next_char()` 这个方法。现在存在两个互不相容的方法：

```python
def event_keydown(key):
    ??

def read_next_char():
    ?? should wait for the next event_keydown() call
```

> You might consider doing that with threads. Greenlets are an alternate solution that don’t have the related locking and shutdown problems. You start the process_commands() function in its own, separate greenlet, and then you exchange the keypresses with it as follows:

也许你会想到用线程处理这种情况。`greenlets` 是一个替代方案，并且不存在一般线程的资源争抢问题。在一个单独 `greenlet` 中执行方法 `process_commands()`，像下面这段代码一样交换按键的响应：

```python
def event_keydown(key):
        # jump into g_processor, sending it the key
    g_processor.switch(key)

def read_next_char():
        # g_self is g_processor in this simple example
    g_self = greenlet.getcurrent()
        # jump to the parent (main) greenlet, waiting for the next key
    next_char = g_self.parent.switch()
    return next_char

g_processor = greenlet(process_commands)
g_processor.switch(*args)   # input arguments to process_commands()

gui.mainloop()
```

> In this example, the execution flow is: when read_next_char() is called, it is part of the g_processor greenlet, so when it switches to its parent greenlet, it resumes execution in the top-level main loop (the GUI). When the GUI calls event_keydown(), it switches to g_processor, which means that the execution jumps back wherever it was suspended in that greenlet – in this case, to the switch() instruction in read_next_char() – and the key argument in event_keydown() is passed as the return value of the switch() in read_next_char().

在这个例子中，执行流程是这样的：当 `read_next_char` 被调用时，它是 `g_processor` 这个 greenlet 的一部分。所以当它切换到父亲线程 执行时，程序流恢复到更高一层的 main loop(the GUI) 来执行。当 GUI 调用 `event_keydown()` 时，程序切换到 `g_processor` ，也就是说执行流程跳转回在 greenlet 中任何它被挂起的地方 - in this case, to the switch() instruction in read_next_char() – and the key argument in event_keydown() is passed as the return value of the switch() in read_next_char(). ( 实在翻译不出来了 :) ) 

写一下自己对上一段的理解，`g_processor` 为运行 `process_commands` 方法的 greenlet 对象，`g_processor.switch(*args)` 切换执行权限到自己开始处理键盘输入。`gui.mainloop()` 开始运行后，等待键盘输入事件。当用户键入命令时，`event_keydown(key)` 被回调响应 keydown 事件，`g_processor.switch(key)` 切换到 `process_commands()` 方法处理输入。`read_next_char()` 方法切换到父协程执行，并返回用户输入的字符。父协程就是 `gui.mainloop()`，gui 则继续监听用户输入事件。

有一点不明白，`g_self.parent.switch()` 的返回值为什么是用户输入的字符呢？

> Note that read_next_char() will be suspended and resumed with its call stack preserved, so that it will itself return to different positions in process_commands() depending on where it was originally called from. This allows the logic of the program to be kept in a nice control-flow way; we don’t have to completely rewrite process_commands() to turn it into a state machine.

注意，`read_next_char()` 会根据它的保留的函数调用栈被 suspend/resume。所以，根据它最开始在哪里被调用来确定返回 `process_commands()` 方法中后，再次开始执行时的位置。这会保证程序执行逻辑流程可控，我们不用必须重写 `process_commands()` 使它成为一个状态机。( 翻译的赶脚像闻脚 :) )


#### 使用方法

##### 介绍

> A “greenlet” is a small independent pseudo-thread. Think about it as a small stack of frames; the outermost (bottom) frame is the initial function you called, and the innermost frame is the one in which the greenlet is currently paused. You work with greenlets by creating a number of such stacks and jumping execution between them. Jumps are never implicit: a greenlet must choose to jump to another greenlet, which will cause the former to suspend and the latter to resume where it was suspended. Jumping between greenlets is called “switching”.

"greenlet"是一个独立的轻量级伪线程。把它当作一小块堆栈中的多个帧，最外面的(栈底)的帧是最初函数调用的位置，最里面的(栈顶)的帧是 greenlet 当前暂停(paused)的位置。使用 greenlets 工作，就是创建多个这样的栈，在这些栈之间切换执行。需要显示切换栈的执行：正在执行的 greenlet 执行完毕后必须选择显示指出下一个要切换到哪个 greenlet，切换使前一个 greenlet 挂起(suspend)，后一个 greenlet 恢复上次挂起的位置继续执行(resume)。greenlets 之间的执行跳转成为"“switching”"。

> When you create a greenlet, it gets an initially empty stack; when you first switch to it, it starts to run a specified function, which may call other functions, switch out of the greenlet, etc. When eventually the outermost function finishes its execution, the greenlet’s stack becomes empty again and the greenlet is “dead”. Greenlets can also die of an uncaught exception.

在 greenlet 刚刚创建的时候，它获取到一个初始化的空栈。第一次切换到它执行时，它开始执行指定的方法。最终最外层的方法执行完毕后，greenlet 的栈又变为空栈，eventlet 的状态变为"dead"。greenlets 也会因为遇到未捕捉的异常而"dead"。

For example:

```python
from greenlet import greenlet

def test1():
    print 12
    gr2.switch()
    print 34

def test2():
    print 56
    gr1.switch()
    print 78

gr1 = greenlet(test1)
gr2 = greenlet(test2)
gr1.switch()
```

> The last line jumps to test1, which prints 12, jumps to test2, prints 56, jumps back into test1, prints 34; and then test1 finishes and gr1 dies. At this point, the execution comes back to the original `gr1.switch()` call. Note that 78 is never printed.

最后一行使程序跳转到 test1，打印12，然后跳转到 test2，打印56，跳转回 test1，打印34。然后 test1 结束运行，gr1 dies。这时候，程序流程回到最初的 `gr1.switch()` 调用。注意，78不会被打印了。

##### 父亲绿色线程

> Let’s see where execution goes when a greenlet dies. Every greenlet has a “parent” greenlet. The parent greenlet is initially the one in which the greenlet was created (this can be changed at any time). The parent is where execution continues when a greenlet dies. This way, greenlets are organized in a tree. Top-level code that doesn’t run in a user-created greenlet runs in the implicit “main” greenlet, which is the root of the tree.

我们看下一个 greenlet 结束运行时程序流怎么走。每个 greenlet 都有父亲线程。父亲线程是在 greenlet创建的时候(创建后也可以在任何时候修改)初始化的。当一个 greenlet 结束运行时会跳转到父亲线程。greenlets 是按照树状结构组织的。更高一层的代码没有运行在用户创建的 greenlet中，而是隐式的运行在 "main" greenlet中，即整个树状结构的root上。

> In the above example, both gr1 and gr2 have the main greenlet as a parent. Whenever one of them dies, the execution comes back to “main”.

在上面的例子中，gr1 和 gr2 的父亲线程是 main greenlet。每当其中一个执行结束，程序流程执行都会回到"main"。

> Uncaught exceptions are propagated into the parent, too. For example, if the above test2() contained a typo, it would generate a NameError that would kill gr2, and the exception would go back directly into “main”. The traceback would show test2, but not test1. Remember, switches are not calls, but transfer of execution between parallel “stack containers”, and the “parent” defines which stack logically comes “below” the current one.

未捕获的异常也会导致程序回到父亲线程。例如，如果上面的 test2() 有一个排版错误，会产生一个 NameError从而 kill掉 gr2，这个异常会是程序直接回到"main"。traceback 会显示 test2的异常信息，而不是 test1 的。要记住，"switches"不是函数调用，而是在平行的"栈容器"之间转移，当前 greenlet 执行结束后下一个"switch"到哪一个 greenlet 执行由父亲线程定义。

##### 实例化

> `greenlet.greenlet` is the greenlet type, which supports the following operations:

`greenlet.greenlet` 是 greenlet 类型，提供以下操作：

> `greenlet(run=None, parent=None)`

> Create a new greenlet object (without running it). `run` is the callable to invoke, and `parent` is the  parent greenlet, which defaults to the current greenlet.

创建一个 greenlet 对象(只创建不运行)。`run` 是可以调用的 callable 类型(方法或者有__call__方法的类)，`parent` 是父亲线程，如果不指定默认父亲是当前 greenlet。

> `greenlet.getcurrent()`

> Returns the current greenlet (i.e. the one which called this function).

`greenlet.getcurrent()` 返回当前 greenlet(例如，调用 function 的 greenlet)。

> `greenlet.GreenletExit`

> This special exception does not propagate to the parent greenlet; it can be used to kill a single greenlet.

抛出也不会跳转到父亲 greenlet 的特殊类型异常，用来结束一个单一 greenlet。

> The greenlet type can be subclassed, too. A greenlet runs by calling its `run` attribute, which is normally set when the greenlet is created; but for subclasses it also makes sense to define a `run` method instead of giving a `run` argument to the constructor.

greenlet 类型可以被子类继承。greenlet 的运行开始于它的 `run` 属性被调用，`run` 属性是在 greenlet 创建时设置的。对于继承了 greenlet 的子类来说也有实现自己的 `run` 方法的必要，而不是在构造函数中传入参数 `run`。

##### 绿色线程的切换(Switching)

> Switches between greenlets occur when the method switch() of a greenlet is called, in which case execution jumps to the greenlet whose switch() is called, or when a greenlet dies, in which case execution jumps to the parent greenlet. During a switch, an object or an exception is “sent” to the target greenlet; this can be used as a convenient way to pass information between greenlets. For example:

绿色线程的切换发生在以下情况：某个绿色线程的switch()方法被调用，这种情况下，程序跳转到switch()方法被调用的绿色线程执行；某个绿色线程执行结束(dead)，这种情况下，程序跳转到父亲绿色线程。跳转过程中，一个对象或者异常被"send"给下一跳绿色线程，这是一种方便的绿色线程间传递信息的方法。比如：

```python
def test1(x, y):
    z = gr2.switch(x+y)
    print z

def test2(u):
    print u
    gr1.switch(42)

gr1 = greenlet(test1)
gr2 = greenlet(test2)
gr1.switch("hello", " world")
```

> This prints “hello world” and 42, with the same order of execution as the previous example. Note that the arguments of test1() and test2() are not provided when the greenlet is created, but only the first time someone switches to it.

结果是打印 "hello world" 和 42，像之前几个例子一样的执行顺序。注意，方法 test1() 和 test2() 的参数在创建绿色线程的时候并没有指定，但是在第一次切换到绿色线程执行的时候必须指定(gr2.swtich("hello", " world"))。

> Here are the precise rules for sending objects around:

> `g.switch(*args, **kwargs)`

> Switches execution to the greenlet g, sending it the given arguments. As a special case, if g did not start yet, then it will start to run now.

程序执行切换到绿色线程 g，并传递参数。作为特例，如果 g 还没有开始运行，那么现在它将开始运行。

> Dying greenlet

> If a greenlet’s run() finishes, its return value is the object sent to its parent. If run() terminates with an exception, the exception is propagated to its parent (unless it is a greenlet.GreenletExit exception, in which case the exception object is caught and returned to the parent).

如果某个绿色线程的 run() 方法结束运行，返回值是切换时指定传递给父亲线程的对象。如果 run() 方法抛出异常导致运行结束，那么这个异常将会传递给父亲线程(除非异常是 greenlet.GreenletExit，这个异常将被捕获并传递给父亲线程)。

> Apart from the cases described above, the target greenlet normally receives the object as the return value of the call to `switch()` in which it was previously suspended. Indeed, although a call to `switch()` does not return immediately, it will still return at some point in the future, when some other greenlet switches back. When this occurs, then execution resumes just after the `switch()` where it was suspended, and the `switch()` itself appears to return the object that was just sent. This means that `x = g.switch(y)` will send the object `y` to `g`, and will later put the (unrelated) object that some (unrelated) greenlet passes back to us into `x`.

除了上面的描述，通常情况下，目标绿色线程在上次被挂起的地方，接收调用 `switch()` 方法的返回值。尽管调用 `swich()` 方法不是立即返回，但在未来，当去其他绿色线程切换回来时也会返回。这时，程序执行流程会在上次挂起的地方继续往下执行，而且 `switch()` 方法会返回上一个执行线程 send 的对象。这就意味着，`x = g.switch(y)` 将 send object `y` 给 `g`，随后，某个(不相关)的绿色线程也会将传递(不相关)的对象回来给 `x`。

> Note that any attempt to switch to a dead greenlet actually goes to the dead greenlet’s parent, or its parent’s parent, and so on. (The final parent is the “main” greenlet, which is never dead.)

注意：任何尝试切换到一个 dead 绿色线程的行为，最终都会切换到该 dead 绿色线程的父亲线程，或者父亲线程的父亲线程，等等。(最终/或者叫root父亲线程就是 "main"绿色线程，它永远不会 dead)

##### 绿色线程的方法和属性

> `g.switch(*args, **kwargs)` 

> Switches execution to the greenlet g. See above.

> `g.run`

> The callable that g will run when it starts. After g started, this attribute no longer exists.

> `g.parent`

> The parent greenlet. This is writeable, but it is not allowed to create cycles of parents.

> `g.gr_frame`

> The current top frame, or None.

> `g.dead`

> True if g is dead (i.e. it finished its execution).

> `bool(g)`

> True if g is active, False if it is dead or not yet started.

> `g.throw([typ, [val, [tb]]])`

> Switches execution to the greenlet g, but immediately raises the given exception in g. If no argument is provided, the exception defaults to greenlet.GreenletExit. The normal exception propagation rules apply, as described above. Note that calling this method is almost equivalent to the following:

```python
def raiser():
    raise typ, val, tb
g_raiser = greenlet(raiser, parent=g)
g_raiser.switch()
```

> except that this trick does not work for the greenlet.GreenletExit exception, which would not propagate from g_raiser to g.

##### Python线程和绿色线程 

> Greenlets can be combined with Python threads; in this case, each thread contains an independent “main” greenlet with a tree of sub-greenlets. It is not possible to mix or switch between greenlets belonging to different threads.

绿色线程可以和 Python的线程绑定使用。在这种情况下，每个 Python 线程有一个独立的 "main" 绿色线程，"main" 为根，下面管理树状的一系列子绿色线程。Python 线程之间不可以混用，Python 线程之间的绿色线程也不可以切换。

```c
_______________________________________
| python process                        |
|   _________________________________   |
|  | python thread                   |  |
|  |   _____   ___________________   |  |
|  |  | hub | | pool              |  |  |
|  |  |_____| |   _____________   |  |  |
|  |          |  | greenthread |  |  |  |
|  |          |  |_____________|  |  |  |
|  |          |   _____________   |  |  |
|  |          |  | greenthread |  |  |  |
|  |          |  |_____________|  |  |  |
|  |          |   _____________   |  |  |
|  |          |  | greenthread |  |  |  |
|  |          |  |_____________|  |  |  |
|  |          |                   |  |  |
|  |          |        ...        |  |  |
|  |          |___________________|  |  |
|  |                                 |  |
|  |_________________________________|  |
|                                       |
|   _________________________________   |
|  | python thread                   |  |
|  |_________________________________|  |
|   _________________________________   |
|  | python thread                   |  |
|  |_________________________________|  |
|                                       |
|                 ...                   |
|_______________________________________|
```

##### 绿色线程的回收

> If all the references to a greenlet object go away (including the references from the parent attribute of other greenlets), then there is no way to ever switch back to this greenlet. In this case, a GreenletExit exception is generated into the greenlet. This is the only case where a greenlet receives the execution asynchronously. This gives try:finally: blocks a chance to clean up resources held by the greenlet. This feature also enables a programming style in which greenlets are infinite loops waiting for data and processing it. Such loops are automatically interrupted when the last reference to the greenlet goes away.

如果某个绿色线程对象再没有任何一个引用(包含父亲线程中其他绿色线程中)，程序执行流程就再也不会切换到这个绿色线程了。在这种情况下，一个 GreenletExit异常会生成到该绿色线程中。这是绿色线程唯一接收一个异步异常的场景。可以使用 try:finally: 代码块清理绿色线程的资源。这个特性也给了一种处理并终结绿色线程无线循环等待/处理数据的编程方式。在绿色线程最后一个引用消失后，循环自动终止。

> The greenlet is expected to either die or be resurrected by having a new reference to it stored somewhere; just catching and ignoring the GreenletExit is likely to lead to an infinite loop.

绿色线程通过一个保存在某处的新的引用而 dead或复活，单纯捕获或者忽略 GreenletExit异常可以导致无限循环。(翻译的好囧)

> Greenlets do not participate in garbage collection; cycles involving data that is present in a greenlet’s frames will not be detected. Storing references to other greenlets cyclically may lead to leaks.


##### 代码追踪支持

> Standard Python tracing and profiling doesn’t work as expected when used with greenlet since stack and frame switching happens on the same Python thread. It is difficult to detect greenlet switching reliably with conventional methods, so to improve support for debugging, tracing and profiling greenlet based code there are new functions in the greenlet module:

使用绿色线程后，标准的 Python追踪和分析，因为栈和帧在同一个 Python线程中切换而表现的并不像最初的预期。使用传统的方法很难可靠的检测绿色线程的切换。所以，为了改进调试，greenlet 提供了追踪和分析绿色线程代码一系列新的方法：

> `greenlet.gettrace()`

> Returns a previously set tracing function, or None.

> `greenlet.settrace(callback)`

> Sets a new tracing function and returns a previous tracing function, or None. The callback is called on various events and is expected to have the following signature:

```python
def callback(event, args):
    if event == 'switch':
        origin, target = args
        # Handle a switch from origin to target.
        # Note that callback is running in the context of target
        # greenlet and any exceptions will be passed as if
        # target.throw() was used instead of a switch.
        return
    if event == 'throw':
        origin, target = args
        # Handle a throw from origin to target.
        # Note that callback is running in the context of target
        # greenlet and any exceptions will replace the original, as
        # if target.throw() was used with the replacing exception.
        return
```

> For compatibility it is very important to unpack args tuple only when event is either `'switch'` or `'throw'` and not when `event` is potentially something else. This way API can be extended to new events similar to sys.`settrace()`.
