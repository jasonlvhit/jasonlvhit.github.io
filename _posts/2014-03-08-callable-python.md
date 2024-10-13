---
layout: post
title: 短记Python中的callable
description: "Python中的callable"
category: articles
tags: [Python]
---


参考Stackoverflow中的一个[问答](http://stackoverflow.com/questions/111234/what-is-a-callable-in-python)
  <br>  
我们可以使用quit()和exit()退出python shell，想一想在我们敲击这两个函数的背后到底发生了什么。你可能会认为quit()和exit()都是用函数实现的，其实不是这样的，他们都是使用了同一个Quitter类作为实现：


[site.py](http://svn.python.org/projects/python/trunk/Lib/site.py)
{% highlight py %}
	class Quitter(object):
	    def __init__(self, name):
	        self.name = name
	    def __repr__(self):
	        return 'Use %s() or %s to exit' % (self.name, eof)
	    def __call__(self, code=None):
	        # Shells like IDLE catch the SystemExit, but listen when their
	        # stdin wrapper is closed.
	        try:
	            sys.stdin.close()
	        except:
	            pass
	        raise SystemExit(code)
	__builtin__.quit = Quitter('quit')
	__builtin__.exit = Quitter('exit')
{% endhighlight %}


喔，类也能被这样使用，也可以被作为一个函数来调用，没错，这就是callable了。callable，从字面的意义上来看，就是可调用的，我们都知道函数是可调用的，没错，函数是一个callable对象。  
<br>
创造一个callable对象易如反掌，只需要在我们的类中定义并实现__call__方法，即使你在__call__里面什么也不干，这个类所创建的对象也就变得callable了。在Python 3中，所有的函数的type都是一个‘function’的class，在这个function的class中，同样定义了__call__方法，在我们实际调用函数的时候，调用的就是这个__call__方法。  
<br>
判断一个对象实例是否callable，可以使用callable函数：
{% highlight py %}
Python 2.7:
>>> "hello"()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'str' object is not callable
>>> callable("hello")
False
>>> def foo(x):
...  return x
...
>>> callable(foo)
True
>>> foo(8)
8
>>> foo.__call__(8)
8
>>> type(foo).__call__(foo, 8)
8
>>> type(foo)
<type 'function'>
>>> foo
<function foo at 0x0000000000000>
{% endhighlight %}
这里注意在Python 3中的区别：
{% highlight py %}
Python 3.3
>>> def foo(x):
...  return x
...
>>> type(foo)
<class 'function'>
{% endhighlight %}


看出其中的区别了吧。  
<br>
最后，让我们再来看一下C Python中check一个对象是否是callable的函数：

[object.c](http://svn.python.org/view/python/trunk/Objects/object.c?view=markup&pathrev=64962)
{% highlight cpp %}
	/* Test whether an object can be called */

	int
	PyCallable_Check(PyObject *x)
	{
	    if (x == NULL)
	        return 0;
	    if (PyInstance_Check(x)) {
	        PyObject *call = PyObject_GetAttrString(x, "__call__");
	        if (call == NULL) {
	            PyErr_Clear();
	            return 0;
	        }
	        /* Could test recursively but don't, for fear of endless
	           recursion if some joker sets self.__call__ = self */
	        Py_DECREF(call);
	        return 1;
	    }
	    else {
	        return x->ob_type->tp_call != NULL;
	    }
	}
{% endhighlight %}


注释挺有意思的。我们可以看到，函数检查一个对象instance中是否存在__callable__属性，对不起，未必是属性，property又是另一个故事  
<br>
如果不是一个instance，返回 x->ob_type->tp_call != NULL 这条语句的值, tp_call是一个指针，这个指针为NULL的话，x就不是callable的。
