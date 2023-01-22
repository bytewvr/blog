---
layout: post
title: Singleton design pattern
category: programming 
date: 2023-01-20
tags:
  - design pattern
  - pyqtSignal
  - python
  - Singleton
---
I recently needed to limit the number of instances of particular classes to 1 - the singleton design pattern seemed an obvious choice, but in Python the implementation of the Singleton proved to be harder to implement correctly when involving pyqtSignals. 

<!--more-->

## Object creation
The intention of a singleton pattern is to ensure that only a single instance of a class is instantiated, further calls to the class will return the same instance. Object creation in python is a two-step process. First, the object is constructed and memory is allocated by calling the `super()` class. Second, the object is initialized. These two steps involve calls to the dunder (double underscore) methods `__new__` and `__init__` respectively. 

```python
import logging
FORMAT = "[%(funcName)10s():%(lineno)3s] %(message)s"
logging.basicConfig(format=FORMAT, level=logging.DEBUG)
logger = logging.getLogger(__name__)

class myClass():
    def __new__(cls):
        logger.debug(f"Init - {cls.__class__}")
        return super().__new__(cls)
    
    def __init__(self):
        logger.debug(f"Init - {self.__class__}")
        super().__init__()

a = myClass()
```
Creating an instance of `myClass` results in the following debug text showing the expected order of method calls
```
[   __new__():  3] Init - <class 'type'>
[  __init__():  7] Init - <class '__main__.myClass'>
```
A common pattern to implement the Singleton in Python is as follows
```python
class myClass():
    def __init__(self):
        logger.debug(f"Init - {self.__class__}")
        super().__init__()

    def __new__(cls):
        if not hasattr(cls, '_instance'):
            cls._instance = super().__new__(cls)
            logger.debug(f"New instance: {cls}")
        logger.debug(f"{cls}")
        return cls._instance

a = myClass()
b = myClass()
```
While both `a` and `b` are a reference to the same instance of `myClass` (see their address on the stack below),  `__init__` is called twice.
```
[   __new__():  9] New instance: <class '__main__.myClass'>
[   __new__(): 10] <__main__.myClass object at 0x7fd9417cb6d0>
[  __init__():  3] Init - <class '__main__.myClass'>
[   __new__(): 10] <__main__.myClass object at 0x7fd9417cb6d0>
[  __init__():  3] Init - <class '__main__.myClass'>
```
While just an annoyance in the case above, something peculiar happened once I started adding signals ( `pyqtSignals`) to the mix. In the following snippet I added signals to the class and connected two receivers after instantiation. 
```python
from PyQt5.QtCore import QObject, pyqtSignal

class myClass(QObject):
    signal = pyqtSignal(object)
    logger.debug(f'signal init -  {signal}')
    
    def __init__(self):
        logger.debug(f"Init - {self.__class__}")
        super().__init__()

    def __new__(cls):
        if not hasattr(cls, '_instance'):
            cls._instance = super().__new__(cls)
            logger.debug(f"New instance: {cls}")
        logger.debug(f"{cls}")
        return cls._instance

    def emit_signal(self):
        self.signal.emit('signal')

def responderA(msg):
    logger.debug(f"{msg}")

def responderB(msg):
    logger.debug(f"{msg}")

a = myClass()
connA = a.signal.connect(responderA)

b = myClass()
connB = b.signal.connect(responderB)

a.emit_signal()
b.emit_signal()
```
I expected two messages from both `responderA()` and `responderB()`, but I got this instead:
```
[   myClass():  8] signal init -  <unbound PYQT_SIGNAL PyQt_PyObject)>
[   __new__(): 17] New instance: <class '__main__.myClass'>
[  __init__(): 11] Init - <class '__main__.myClass'>
[  __init__(): 11] Init - <class '__main__.myClass'>
[responderB(): 27] signal
[responderB(): 27] signal
```
While `b` is a referenced to the same object due to the singleton nature, the signal seems to be disconnected from `responderA()`. We can check the number of receivers a signal is connected to with 
```python
def number_of_signal_receivers(instance, signal_name):
    return QObject.receivers(instance, instance.__getattr__(signal_name))

...

a = myClass()
a.signal.connect(responderA)
print(number_of_signal_receivers(a, b'signal'))

b = myClass()
b.signal.connect(responderB)
print(number_of_signal_receivers(b, b'signal'))
```
Omitting the debug output we observe that the receiver count is not increasing after connecting `responderB()` to our signal. 
```
1
1
```
How do we fix this? 
# Solution 1
After creating the object instance and connecting a signal any subsequent call to ```__init__()``` will wipe out our previous connections. We could try to restrict the init-call to the first call with
```python
class myClass(QObject):
    signal = pyqtSignal(object)
    _initialized = None
    
	def __init__(self):
		if self._initialized is None:
			logger.debug(f"Init - {self.__class__}")
			super().__init__()
			self._initialized = True
   ...
```
This leads to properly connected signals as the increase in the receiver counts after the second print of the `number_of_signal_receivers()` demonstrates. 
# Solution 2
However, we can also prevent the second call to`__init__()` altogether using metaclasses. While all objects in Python ultimately inherit from Object, the factory that generates the object is the `type` class. `type(object)` , `type(myClass)` results in `type` (or a Qt wrapper such as `sip.wrappertype`). As it is generating classes it is called a metaclass. 
When we write `myClass()` the metaclass' `__call__` gets called (in this case belonging to `type`). Then if `__new__` and `__init__` are defined in the child class they will be called or the methods from the object class respectively. We can inject our own Singleton metaclass into the game to prevent a call to `__new__` and `__init__` altogether.
```python
class Singleton(type(QObject), type):
    def __call__(cls, *args, **kwargs):
        logger.debug(f"Singleton {cls.__class__}")
        if not hasattr(cls, '_instance'):
            cls._instance = super().__call__(*args, **kwargs)
            logger.debug(f"Singleton - new instance {cls._instance}")
        logger.debug(f"Singleton {cls.__class__}")
        return cls._instance

class myClass(QObject, metaclass=Singleton):
    signal = pyqtSignal(object)
    logger.debug(f'signal init -  {signal}')

    def __init__(self):
        logger.debug(f"A - {self.__class__}")
        super().__init__()

    def __new__(self):
        logger.debug(f"A - {self.__class__}")
        return super().__new__(self)

    def emit_signal(self):
        self.signal.emit('signal')

def responderA(msg):
    logger.debug(f"{msg}")
def responderB(msg):
    logger.debug(f"{msg}")

a = myClass()
connA = a.signal.connect(responderA)
print(number_of_signal_receivers(a, b'signal'))

b = myClass()
connB = b.signal.connect(responderB)
print(number_of_signal_receivers(b, b'signal'))

a.emit_signal()
b.emit_signal()
```
The debug printout shows that we only get one call to the new and init dunder methods of our `myClass`, and the signal retains its connections - pretty sweet.
```
[   myClass(): 22] signal init -  <unbound PYQT_SIGNAL PyQt_PyObject)>
[  __call__(): 13] Singleton <class '__main__.Singleton'>
[   __new__(): 29] A - <class '__main__.Singleton'>
[  __init__(): 25] A - <class '__main__.myClass'>
[  __call__(): 16] Singleton - new instance <__main__.myClass object at 0x7fd941475ab0>
[  __call__(): 17] Singleton <class '__main__.Singleton'>
[  __call__(): 13] Singleton <class '__main__.Singleton'>
[  __call__(): 17] Singleton <class '__main__.Singleton'>
[responderA(): 36] signal
[responderB(): 38] signal
[responderA(): 36] signal
[responderB(): 38] signal
```
# Conclusion
I presented two methods that keep our pyqtSignal descriptor connected to its receivers. It was definitely quite a treat to learn about metaclasses. 

# Additional Resources
[stackoverflow - Receiving pyqtsignal from singleton](https://stackoverflow.com/questions/59459770/receiving-pyqtsignal-from-singleton) <br />
[Understanding Object Instantiation and Metaclasses in Python](https://www.honeybadger.io/blog/python-instantiation-metaclass/) by Rupesh Mishra <br />
[Python’s super() considered super!](https://rhettinger.wordpress.com/2011/05/26/super-considered-super/) by Raymond Hettinger

