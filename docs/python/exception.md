# 错误和异常

## 处理异常

```PYTHON
>>> def divide(x, y):
...     try:
...         result = x / y
...     except ZeroDivisionError:
...         print("division by zero!")
...     else:
...         print("result is", result)
...     finally:
...         print("executing finally clause")
```
try 语句的工作原理如下：

- 首先，执行 try 子句 （try 和 except 关键字之间的（多行）语句）。
- 如果没有异常发生，则跳过 except 子句 并完成 try 语句的执行。
- 如果在执行 try 子句时发生了异常，则跳过该子句中剩下的部分。 然后，如果异常的类型和 except 关键字后面的异常匹配，则执行 except 子句，然后继续执行 try 语句之后的代码。
- 如果发生的异常和 except 子句中指定的异常不匹配，则将其传递到外部的 try 语句中；如果没有找到处理程序，则它是一个 未处理异常，执行将停止并显示如上所示的消息。
- 一个 try 语句可能有多个 except 子句，以指定不同异常的处理程序。 最多会执行一个处理程序。 处理程序只处理相应的 try 子句中发生的异常，而不处理同一 try 语句内其他处理程序中的异常
- 一个 except 子句可以将多个异常命名为带括号的元组
    ```PYTHON
    ... except (RuntimeError, TypeError, NameError):
    ...     pass
    ```
- try ... except 语句有一个可选的 else 子句 在使用时必须放在所有的 except 子句后面

## 抛出异常raise

```PYTHON
raise NameError('HiThere')
```

## 用户自定义异常

程序可以通过创建新的异常类来命名它们自己的异常（有关Python 类的更多信息，请参阅 类）。异常通常应该直接或间接地从 `Exception` 类派生。

```PYTHON
class Error(Exception):
    """Base class for exceptions in this module."""
    pass

class InputError(Error):
    """Exception raised for errors in the input.

    Attributes:
        expression -- input expression in which the error occurred
        message -- explanation of the error
    """

    def __init__(self, expression, message):
        self.expression = expression
        self.message = message

class TransitionError(Error):
    """Raised when an operation attempts a state transition that's not
    allowed.

    Attributes:
        previous -- state at beginning of transition
        next -- attempted new state
        message -- explanation of why the specific transition is not allowed
    """

    def __init__(self, previous, next, message):
        self.previous = previous
        self.next = next
        self.message = message
```

## finally

如果存在 finally 子句，则 finally 子句将作为 try 语句结束前的最后一项任务被执行。 finally 子句不论 try 语句是否产生了异常都会被执行。 以下几点讨论了当异常发生时一些更复杂的情况：
- 如果在执行 try 子句期间发生了异常，该异常可由一个 except 子句进行处理。 如果异常没有被某个 except 子句所处理，则该异常会在 finally 子句执行之后被重新引发。
- 异常也可能在 except 或 else 子句执行期间发生。 同样地，该异常会在 finally 子句执行之后被重新引发。
- 如果在执行 try 语句时遇到一个 break, continue 或 return 语句，则 finally 子句将在执行 break, continue 或 return 语句之前被执行。
- 如果 finally 子句中包含一个 return 语句，则返回值将来自 finally 子句的某个 return 语句的返回值，而非来自 try 子句的 return 语句的返回值。

## 内置异常

在 Python 中，所有异常必须为一个派生自 `BaseException` 的类的实例。 在带有提及一个特定类的 except 子句的 try 语句中

内置异常类可以被子类化以定义新的异常；鼓励程序员从 `Exception` 类或它的某个子类而不是从 `BaseException` 来派生新的异常

当引发一个新的异常（而不是简单地使用 raise 来重新引发 当前在处理的异常）时，隐式的异常上下文可以通过使用带有 raise 的 `from` 子句来补充一个显式的原因:
```PYTHON
raise new_exc from original_exc
```
跟在 from 之后一表达式必须为一个异常或 None

## 基类

### exception BaseException
所有内置异常的基类。 它不应该被用户自定义类直接继承 (这种情况请使用 `Exception`)。 如果在此类的实例上调用 `str()`，则会返回实例的参数表示，或者当没有参数时返回空字符串。

**args**

传给异常构造器的`参数元组`。 某些内置异常 (例如 OSError) 接受特定数量的参数并赋予此元组中的元素特殊的含义，而其他异常**通常只接受一个给出错误信息的单独字符串**。

**with_traceback(tb)**

此方法将 tb 设为异常的新回溯信息并返回该异常对象。 它通常以如下的形式在异常处理程序中使用:
```PYTHON
try:
    ...
except SomeException:
    tb = sys.exc_info()[2]
    raise OtherException(...).with_traceback(tb)
```

## sys.exc_info()

本函数返回的`元组`包含三个值，它们给出当前正在处理的异常的信息。返回的信息仅限于当前线程和当前堆栈帧。如果当前堆栈帧没有正在处理的异常，则信息将从调用的下级堆栈帧或上级调用者等位置获取，依此类推，直到找到正在处理异常的堆栈帧为止。此处的“处理异常”被定义为“执行 except 子句”。任何堆栈帧都只能访问当前正在处理的异常的信息。

如果整个堆栈都没有正在处理的异常，则返回包含三个 `None` 值的元组。否则返回值为 (`type, value, traceback`)。它们的含义是：`type` 是正在处理的`异常类型`（它是 BaseException 的子类）；`value` 是`异常实例`（异常类型的实例）；`traceback` 是一个 `回溯对象`，该对象封装了最初发生异常时的调用堆栈。

## traceback

该模块提供了一个标准接口来提取、格式化和打印 Python 程序的堆栈跟踪结果

这个模块使用 traceback 对象 —— 这是存储在 sys.last_traceback 中的对象类型变量，并作为 `sys.exc_info()` 的第三项被返回

### traceback.print_tb(tb, limit=None, file=None)
如果*limit*是正整数，那么从 traceback 对象 "tb" 输出最高 limit 个（从调用函数开始的）栈的堆栈回溯条目；如果 limit 是负数就输出 abs(limit) 个回溯条目；又如果 limit 被省略或者为 None，那么就会输出所有回溯条目。如果 file 被省略或为 None 那么就会输出至标准输出``sys.stderr``否则它应该是一个打开的文件或者文件类对象来接收输出

### traceback.print_exception(etype, value, tb, limit=None, file=None, chain=True)
打印回溯对象 tb 到 file 的异常信息和整个堆栈回溯。这和 print_tb() 比有以下方面不同：

如果 tb 不为 None，它将打印一个开始信息``Traceback (most recent call last):``

它将在输出完堆栈回溯后，输出异常中的 etype 和 value 的信息

if type(value) is SyntaxError and value has the appropriate format, it prints the line where the syntax error occurred with a caret indicating the approximate position of the error.

### traceback.print_exc(limit=None, file=None, chain=True)
This is a shorthand for `print_exception(*sys.exc_info(), limit, file, chain)`

### traceback.format_exc(limit=None, chain=True)
This is like print_exc(limit) but returns a string instead of printing to a file.

```PYTHON
import sys, traceback

def run_user_code(envdir):
    source = input(">>> ")
    try:
        exec(source, envdir)
    except Exception:
        print("Exception in user code:")
        print("-"*60)
        traceback.print_exc(file=sys.stdout)
        print("-"*60)

envdir = {}
while True:
    run_user_code(envdir)

def lumberjack():
    bright_side_of_death()

def bright_side_of_death():
    return tuple()[0]

try:
    lumberjack()
except IndexError:
    exc_type, exc_value, exc_traceback = sys.exc_info()
    print("*** print_tb:")
    traceback.print_tb(exc_traceback, limit=1, file=sys.stdout)
    print("*** print_exception:")
    # exc_type below is ignored on 3.5 and later
    traceback.print_exception(exc_type, exc_value, exc_traceback,
                              limit=2, file=sys.stdout)
    print("*** print_exc:")
    traceback.print_exc(limit=2, file=sys.stdout)
    print("*** format_exc, first and last line:")
    formatted_lines = traceback.format_exc().splitlines()
    print(formatted_lines[0])
    print(formatted_lines[-1])
    print("*** format_exception:")
    # exc_type below is ignored on 3.5 and later
    print(repr(traceback.format_exception(exc_type, exc_value,
                                          exc_traceback)))
    print("*** extract_tb:")
    print(repr(traceback.extract_tb(exc_traceback)))
    print("*** format_tb:")
    print(repr(traceback.format_tb(exc_traceback)))
    print("*** tb_lineno:", exc_traceback.tb_lineno)
```
