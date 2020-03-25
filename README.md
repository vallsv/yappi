<p align="center">
    <img src="https://raw.githubusercontent.com/sumerc/yappi/master/Misc/logo.png" alt="yappi">
</p>

<h1 align="center">Yappi</h1>
<p align="center">
    Yet Another Python Profiler, but this time <b>thread&coroutine</b> aware.
</p>

<p align="center">
    <img src="https://www.travis-ci.org/sumerc/yappi.svg?branch=master">
    <img src="https://ci.appveyor.com/api/projects/status/github/sumerc/yappi?branch=master&svg=true">
    <img src="https://img.shields.io/pypi/v/yappi.svg">
    <img src="https://img.shields.io/pypi/dw/yappi.svg">
    <img src="https://img.shields.io/pypi/pyversions/yappi.svg">
    <img src="https://img.shields.io/github/last-commit/sumerc/yappi.svg">
    <img src="https://img.shields.io/github/license/sumerc/yappi.svg">
</p>

## Highlights

- **Fast**: Yappi is fast. It is completely written in C and lots of love&care went into making it fast.
- **Unique**: Yappi supports multithreaded and [asynchronous code](https://github.com/sumerc/yappi/blob/master/doc/coroutine-profiling.md) profiling. Tagging/filtering multiple profiler results has interesting [use cases](https://github.com/sumerc/yappi/blob/master/doc/api.md#set_tag_callback).
- **Intuitive**: Profiler can be started/stopped and results can be obtained from any time and any thread.
- **Standards Complaint**: Profiler results can be saved in [callgrind](http://valgrind.org/docs/manual/cl-format.html) or [pstat](http://docs.python.org/3.4/library/profile.html#pstats.Stats) formats.
- **Rich in Feature set**: Profiler results can show either [Wall Time](https://en.wikipedia.org/wiki/Elapsed_real_time) or actual [CPU Time](http://en.wikipedia.org/wiki/CPU_time) and can be aggregated from different sessions. Various flags are defined for filtering and sorting profiler results.
- **Robust**: Yappi had seen more than *8 years* of production usage.

## Motivation

CPython standard distribution comes with three deterministic profilers. `cProfile`, `Profile` and `hotshot`. `cProfile` is implemented as a C module based on `lsprof`, `Profile` is in pure Python and `hotshot` can be seen as a small subset of a cProfile. The major issue is that all of these profilers lack support for multi-threaded programs and CPU time.

If you want to profile a  multi-threaded application, you must give an entry point to these profilers and then maybe merge the outputs. None of these profilers are designed to work on long-running multi-threaded applications. It is also not possible to profile an application that start/stop/retrieve traces on the fly with these profilers. 

Now fast forwarding to 2019: With the latest improvements on `asyncio` library and asynchronous frameworks, most of the current profilers lacks the ability to show correct wall/cpu time or even call count information per-coroutine. Thus we need a different kind of approach to profile asynchronous code. Yappi, with v1.2 introduces the concept of `coroutine profiling`. With `coroutine-profiling`, you should be able to profile correct wall/cpu time and call count of your coroutine. (including the time spent in context switches, too). You can see details [here](https://github.com/sumerc/yappi/blob/master/doc/coroutine-profiling.md).


## Installation

Can be installed via PyPI

```
$ pip install yappi
```

OR from the source directly.

```
$ pip install git+https://github.com/sumerc/yappi#egg=yappi
```

## Examples

A simple example:
```python
import yappi

def a():
    for _ in range(10000000):  # do something CPU heavy
        pass

yappi.set_clock_type("cpu") # Use set_clock_type("wall") for wall time
yappi.start()
a()

yappi.get_func_stats().print_all()
yappi.get_thread_stats().print_all()
'''

Clock type: CPU
Ordered by: totaltime, desc

name                                  ncall  tsub      ttot      tavg      
doc.py:5 a                            1      0.117907  0.117907  0.117907

name           id     tid              ttot      scnt        
_MainThread    0      139867147315008  0.118297  1
'''
```

Profile a multithreaded application and retrieve stats by thread id:

```python
import yappi
import time
import threading

_NTHREAD = 3


def _work(n):
    time.sleep(n * 0.1)


yappi.start()

threads = []
# generate _NTHREAD threads
for i in range(_NTHREAD):
    t = threading.Thread(target=_work, args=(i + 1, ))
    t.start()
    threads.append(t)
# wait all threads to finish
for t in threads:
    t.join()

yappi.stop()

# retrieve thread stats by their thread id (given by yappi)
threads = yappi.get_thread_stats()
for thread in threads:
    print(
        "Function stats for (%s) (%d)" % (thread.name, thread.id)
    )  # it is the Thread.__class__.__name__
    yappi.get_func_stats(filter={"ctx_id": thread.id}).print_all()
'''
Function stats for (Thread) (3)

name                                  ncall  tsub      ttot      tavg
..hon3.7/threading.py:859 Thread.run  1      0.000017  0.000062  0.000062
doc3.py:8 _work                       1      0.000012  0.000045  0.000045

Function stats for (Thread) (2)

name                                  ncall  tsub      ttot      tavg
..hon3.7/threading.py:859 Thread.run  1      0.000017  0.000065  0.000065
doc3.py:8 _work                       1      0.000010  0.000048  0.000048


Function stats for (Thread) (1)

name                                  ncall  tsub      ttot      tavg
..hon3.7/threading.py:859 Thread.run  1      0.000010  0.000043  0.000043
doc3.py:8 _work                       1      0.000006  0.000033  0.000033
'''
```

Different ways to filter/sort stats:
```python
import package_a
import yappi
import sys

def a():
    pass

def b():
    pass

yappi.start()
a()
b()
package_a.a()
yappi.stop()

# filter by module object
current_module = sys.modules[__name__]
stats = yappi.get_func_stats(
    filter_callback=lambda x: yappi.module_matches(x, [current_module])
)  # x is a yappi.YFuncStat object
stats.sort("name", "desc").print_all()
'''
Clock type: CPU
Ordered by: name, desc

name                                  ncall  tsub      ttot      tavg
doc2.py:10 b                          1      0.000001  0.000001  0.000001
doc2.py:6 a                           1      0.000001  0.000001  0.000001
'''

# filter by function object
stats = yappi.get_func_stats(
    filter_callback=lambda x: yappi.func_matches(x, [a, b])
).print_all()
'''
name                                  ncall  tsub      ttot      tavg
doc2.py:6 a                           1      0.000001  0.000001  0.000001
doc2.py:10 b                          1      0.000001  0.000001  0.000001
'''

# filter by module name
stats = yappi.get_func_stats(filter_callback=lambda x: 'package_a' in x.module
                             ).print_all()
'''
name                                  ncall  tsub      ttot      tavg
package_a/__init__.py:1 a             1      0.000001  0.000001  0.000001
'''

# filter by function name
stats = yappi.get_func_stats(filter_callback=lambda x: 'a' in x.name
                             ).print_all()
'''
name                                  ncall  tsub      ttot      tavg
doc2.py:6 a                           1      0.000001  0.000001  0.000001
package_a/__init__.py:1 a             1      0.000001  0.000001  0.000001
'''
```

Profile an async application:
```python
xxx
```

Use tags for context-aware profiling:
```python
xxx
```


## Documentation

- [Introduction](https://github.com/sumerc/yappi/blob/master/doc/introduction.md)
- [Clock Types](https://github.com/sumerc/yappi/blob/master/doc/clock_types.md)
- [API](https://github.com/sumerc/yappi/blob/master/doc/api.md)
- [Coroutine Profiling](https://github.com/sumerc/yappi/blob/master/doc/coroutine-profiling.md) _(new in 1.2)_

  Note: Yes. I know I should be moving docs to readthedocs.io. Stay tuned!


## Related Talks

  Special thanks to A.Jesse Jiryu Davis:
- [Python Performance Profiling: The Guts And The Glory (PyCon 2015)](https://www.youtube.com/watch?v=4uJWWXYHxaM)

## PyCharm Integration

Yappi is the default profiler in `PyCharm`. If you have Yappi installed, `PyCharm` will use it. See [the official](https://www.jetbrains.com/help/pycharm/profiler.html) documentation for more details.

