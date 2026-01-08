---
weight: 5
bookToc: true
title: "asyncio"
---

asyncio 是一个使用`async`/`await`编写并发代码的库，很多高性能异步Web框架使用了这个库。

--- 

## 概念

首先明确一些概念和理念

### Future & Task

- **Future表示的是未来的一个结果，通过`fut.set_result(42)`设置结果，`add_done_callback`设置完成时的回调** 
    ```python
    _PENDING = base_futures._PENDING
    _CANCELLED = base_futures._CANCELLED
    _FINISHED = base_futures._FINISHED

    class Future:
        """This class is *almost* compatible with concurrent.futures.Future.

        Differences:

        - This class is not thread-safe.

        - result() and exception() do not take a timeout argument and
        raise an exception when the future isn't done yet.

        - Callbacks registered with add_done_callback() are always called
        via the event loop's call_soon().

        - This class is not compatible with the wait() and as_completed()
        methods in the concurrent.futures package.

        (In Python 3.4 or later we may be able to unify the implementations.)
        """

        # Class variables serving as defaults for instance variables.
        _state = _PENDING
        _result = None
        _exception = None
        _loop = None
        _source_traceback = None

        def set_result(self, result):
            """Mark the future done and set its result.

            If the future is already done when this method is called, raises
            InvalidStateError.
            """
            if self._state != _PENDING:
                raise exceptions.InvalidStateError(f'{self._state}: {self!r}')
            self._result = result
            self._state = _FINISHED
            self.__schedule_callbacks()
    ```

- **协程是暂停/恢复的函数，本身不会自动运行，只能放在事件循环里运行，会被封装成Task**


    ```python
    task = asyncio.create_task(foo())

    # cpython/Lib/asyncio/tasks.py
    def create_task(coro, *, name=None):
        """Schedule the execution of a coroutine object in a spawn task.

        Return a Task object.
        """
        loop = events.get_running_loop() # 需要有一个running的事件循环
        task = loop.create_task(coro)
        _set_task_name(task, name)
        return task
    
    class Task(futures._PyFuture):  # Inherit Python Task implementation
                                    # from a Python Future implementation.

        """A coroutine wrapped in a Future."""

        # An important invariant maintained while a Task not done:
        #
        # - Either _fut_waiter is None, and _step() is scheduled;
        # - or _fut_waiter is some Future, and _step() is *not* scheduled.
        #
        # The only transition from the latter to the former is through
        # _wakeup().  When _fut_waiter is not None, one of its callbacks
        # must be _wakeup().

        # If False, don't log a message if the task is destroyed whereas its
        # status is still pending

        def __init__(self, coro, *, loop=None, name=None):
            super().__init__(loop=loop)
            ...
            self._coro = coro
            self._context = contextvars.copy_context()
            ...
            self._loop.call_soon(self.__step, context=self._context)
            _register_task(self)
    ```
    Task的执行是调用了事件循环的`call_soon`方法，将`task.__step`封装成`Handler`放入`loop._ready`队列中，然后循环队列`_run_once`执行（详细参看下方介绍），推动协程/任务向前执行一步

    ```python
    # cpython/Lib/asyncio/base_events.py
    class BaseEventLoop(events.AbstractEventLoop):
        ...
        def create_task(self, coro, *, name=None):
            """Schedule a coroutine object.
            Return a task object.
            """
            self._check_closed()
            if self._task_factory is None:
                task = tasks.Task(coro, loop=self, name=name)
                if task._source_traceback:
                    del task._source_traceback[-1]
            else:
                task = self._task_factory(self, coro)
                tasks._set_task_name(task, name)
            return task
        
        ...
        def call_soon(self, callback, *args, context=None):
        """Arrange for a callback to be called as soon as possible.

        This operates as a FIFO queue: callbacks are called in the
        order in which they are registered.  Each callback will be
        called exactly once.

        Any positional arguments after the callback will be passed to
        the callback when it is called.
        """
        self._check_closed()
        if self._debug:
            self._check_thread()
            self._check_callback(callback, 'call_soon')
        handle = self._call_soon(callback, args, context)
        if handle._source_traceback:
            del handle._source_traceback[-1]
        return handle
        ...

        def _call_soon(self, callback, args, context):
        handle = events.Handle(callback, args, self, context)
        if handle._source_traceback:
            del handle._source_traceback[-1]
        self._ready.append(handle)
        return handle
    ```

- Future是Task的基类，可以感受到Task is a Future，Task天生是awaitable的，事件循环只需要理解Future
    ```python
    await asyncio.gather(task1, task2, fut)
    ```

### async / await

- `async`关键字用于定义协程，可以暂停恢复执行的函数，**`await`关键字用于挂起当前协程，把控制权交给事件循环，这样其他暂停的协程才能被事件循环推进**
- `async for / async with`，这些只有在协程上下文当中才有意义，语法层面因为2者都和`await`有关，而 `await` 只能出现在协程上下文；出现在普通函数中也与语义不相符合

- **我们在 [魔术方法]({{< relref "/docs/languages/python/magic.md" >}}) 有说到，`await xxx`，要求`xxx`实现`__await__`即可，这只是Python语言级的通用机制只要求外部能驱动迭代，而asyncio对xxx有进一步的要求，需要是 Future-like 对象，主要原因是需要通过future的done回调__wakeup task**
    ```python
    def __step(self, exc=None):
            if self.done():
                raise exceptions.InvalidStateError(
                    f'_step(): already done: {self!r}, {exc!r}')
            if self._must_cancel:
                if not isinstance(exc, exceptions.CancelledError):
                    exc = exceptions.CancelledError()
                self._must_cancel = False
            coro = self._coro
            self._fut_waiter = None

            _enter_task(self._loop, self)
            # Call either coro.throw(exc) or coro.send(None).
            try:
                if exc is None:
                    # We use the `send` method directly, because coroutines
                    # don't have `__iter__` and `__next__` methods.
                    result = coro.send(None)
                else:
                    result = coro.throw(exc)
            except StopIteration as exc:
                if self._must_cancel:
                    # Task is cancelled right before coro stops.
                    self._must_cancel = False
                    super().cancel()
                else:
                    super().set_result(exc.value)
            except exceptions.CancelledError:
                super().cancel()  # I.e., Future.cancel(self).
            except (KeyboardInterrupt, SystemExit) as exc:
                super().set_exception(exc)
                raise
            except BaseException as exc:
                super().set_exception(exc)
            else:
                blocking = getattr(result, '_asyncio_future_blocking', None)
                if blocking is not None:
                    # Yielded Future must come from Future.__iter__().
                    if futures._get_loop(result) is not self._loop:
                        new_exc = RuntimeError(
                            f'Task {self!r} got Future '
                            f'{result!r} attached to a different loop')
                        self._loop.call_soon(
                            self.__step, new_exc, context=self._context)
                    elif blocking:
                        if result is self:
                            new_exc = RuntimeError(
                                f'Task cannot await on itself: {self!r}')
                            self._loop.call_soon(
                                self.__step, new_exc, context=self._context)
                        else:
                            result._asyncio_future_blocking = False
                            result.add_done_callback(
                                self.__wakeup, context=self._context)
                            self._fut_waiter = result
                            if self._must_cancel:
                                if self._fut_waiter.cancel():
                                    self._must_cancel = False
                    else:
                        new_exc = RuntimeError(
                            f'yield was used instead of yield from '
                            f'in task {self!r} with {result!r}')
                        self._loop.call_soon(
                            self.__step, new_exc, context=self._context)

                elif result is None:
                    # Bare yield relinquishes control for one event loop iteration.
                    self._loop.call_soon(self.__step, context=self._context)
                elif inspect.isgenerator(result):
                    # Yielding a generator is just wrong.
                    new_exc = RuntimeError(
                        f'yield was used instead of yield from for '
                        f'generator in task {self!r} with {result!r}')
                    self._loop.call_soon(
                        self.__step, new_exc, context=self._context)
                else:
                    # Yielding something else is an error.
                    new_exc = RuntimeError(f'Task got bad yield: {result!r}')
                    self._loop.call_soon(
                        self.__step, new_exc, context=self._context)
            finally:
                _leave_task(self._loop, self)
                self = None  # Needed to break cycles when an exception occurs.

        def __wakeup(self, future):
            try:
                future.result()
            except BaseException as exc:
                # This may also be a cancellation.
                self.__step(exc)
            else:
                # Don't pass the value of `future.result()` explicitly,
                # as `Future.__iter__` and `Future.__await__` don't need it.
                # If we call `_step(value, None)` instead of `_step()`,
                # Python eval loop would use `.send(value)` method call,
                # instead of `__next__()`, which is slower for futures
                # that return non-generator iterators from their `__iter__`.
                self.__step()
            self = None  # Needed to break cycles when an exception occurs.
    ```
    `add_done_callback`在future done时会调用了事件循环的`call_soon`方法，将`task.__wakeup`封装成`Handle`放入`loop._ready`队列中，然后循环队列`_run_once`执行（详细参看下方介绍），这样触发了被`await`挂起协程/任务的继续执行
    ```python
    # cpython/Lib/asyncio/futures.py
    class Future:
        ...
        def add_done_callback(self, fn, *, context=None):
        """Add a callback to be run when the future becomes done.

        The callback is called with a single argument - the future object. If
        the future is already done when this is called, the callback is
        scheduled with call_soon.
        """
        if self._state != _PENDING:
            self._loop.call_soon(fn, self, context=context)
        else:
            if context is None:
                context = contextvars.copy_context()
            self._callbacks.append((fn, context))
    ```

### event loop

事件循环执行的是`loop._ready`FIFO队列中的`handle`，进行了统一的封装，具体可能是推进任务的一小步`task.__step`，延迟的`loop.call_later`，以及`loop.add_reader()/add_writer()`注册的回调（由 `_process_events()` 在 fd 就绪时塞进 `loop._ready`）

```python
# cpython/Lib/asyncio/base_events.py
class BaseEventLoop(events.AbstractEventLoop):
    ...
    def call_later(self, delay, callback, *args, context=None):
        """Arrange for a callback to be called at a given time.

        Return a Handle: an opaque object with a cancel() method that
        can be used to cancel the call.

        The delay can be an int or float, expressed in seconds.  It is
        always relative to the current time.

        Each callback will be called exactly once.  If two callbacks
        are scheduled for exactly the same time, it undefined which
        will be called first.

        Any positional arguments after the callback will be passed to
        the callback when it is called.
        """
        timer = self.call_at(self.time() + delay, callback, *args,
                             context=context)
        if timer._source_traceback:
            del timer._source_traceback[-1]
        return timer

    def call_at(self, when, callback, *args, context=None):
        """Like call_later(), but uses an absolute time.

        Absolute time corresponds to the event loop's time() method.
        """
        self._check_closed()
        if self._debug:
            self._check_thread()
            self._check_callback(callback, 'call_at')
        timer = events.TimerHandle(when, callback, args, self, context)
        if timer._source_traceback:
            del timer._source_traceback[-1]
        heapq.heappush(self._scheduled, timer)
        timer._scheduled = True
        return timer
```

然后`_run_once`对应事件循环中的一次执行

{{< hint info >}}
方便理解run_once代码的一些提示：
```python
# self._ready：队列，存放Handle（以及到期后被搬过来的 TimerHandle），代表能执行
# self._scheduled：小根堆，存放带_when的TimerHandle，是Handle的子类支持按时间排序
# BaseEventLoop run_once执行框架
# -> BaseSelectorEventLoop 基于selector的事件循环
#     -> _UnixSelectorEventLoop 特定unix平台基于selector的实现
# -> BaseProactorEventLoop 基于proactor的事件循环
#     -> ProactorEventLoop 特定windows平台基于proactor的实现

# Selector和Proactor都有select方法，同时BaseProactorEventLoop中设置了_selector为_proactor的别称
# `self._selector.select(timeout)`

class BaseProactorEventLoop(base_events.BaseEventLoop):

    def __init__(self, proactor):
        super().__init__()
        logger.debug('Using proactor: %s', proactor.__class__.__name__)
        self._proactor = proactor
        self._selector = proactor   # convenient alias
```
{{< /hint >}}

```python
class BaseEventLoop(events.AbstractEventLoop):
    ...
    def _run_once(self):
            """Run one full iteration of the event loop.

            This calls all currently ready callbacks, polls for I/O,
            schedules the resulting callbacks, and finally schedules
            'call_later' callbacks.
            """

            # lazy cancellation 移除cancel的
            sched_count = len(self._scheduled)
            if (sched_count > _MIN_SCHEDULED_TIMER_HANDLES and
                self._timer_cancelled_count / sched_count >
                    _MIN_CANCELLED_TIMER_HANDLES_FRACTION):
                # Remove delayed calls that were cancelled if their number
                # is too high
                new_scheduled = []
                for handle in self._scheduled:
                    if handle._cancelled:
                        # 表示不在 _scheduled 小根堆里了
                        handle._scheduled = False
                    else:
                        new_scheduled.append(handle)

                heapq.heapify(new_scheduled)
                self._scheduled = new_scheduled
                self._timer_cancelled_count = 0
            else:
                # Remove delayed calls that were cancelled from head of queue.
                while self._scheduled and self._scheduled[0]._cancelled:
                    self._timer_cancelled_count -= 1
                    handle = heapq.heappop(self._scheduled)
                    handle._scheduled = False   

            timeout = None

            # ready队列有能执行的或者loop退出了，selector timeout为0，不要阻塞在selector上了
            # 否则通过select的超时返回机制实现等待一段时间
            if self._ready or self._stopping:
                timeout = 0
            elif self._scheduled:
                # Compute the desired timeout.
                when = self._scheduled[0]._when
                timeout = min(max(0, when - self.time()), MAXIMUM_SELECT_TIMEOUT)

            event_list = self._selector.select(timeout)
            self._process_events(event_list)

            # Handle 'later' callbacks that are ready.
            end_time = self.time() + self._clock_resolution
            while self._scheduled:
                handle = self._scheduled[0]
                if handle._when >= end_time:
                    break
                handle = heapq.heappop(self._scheduled)
                handle._scheduled = False
                self._ready.append(handle)

            # This is the only place where callbacks are actually *called*.
            # All other places just add them to ready.
            # Note: We run all currently scheduled callbacks, but not any
            # callbacks scheduled by callbacks run this time around --
            # they will be run the next time (after another I/O poll).
            # Use an idiom that is thread-safe without using locks.
            ntodo = len(self._ready)
            for i in range(ntodo):
                handle = self._ready.popleft()
                if handle._cancelled:
                    continue
                if self._debug:
                    try:
                        self._current_handle = handle
                        t0 = self.time()
                        handle._run()
                        dt = self.time() - t0
                        if dt >= self.slow_callback_duration:
                            logger.warning('Executing %s took %.3f seconds',
                                        _format_handle(handle), dt)
                    finally:
                        self._current_handle = None
                else:
                    handle._run()
            handle = None  # Needed to break cycles when an exception occurs.

class BaseSelectorEventLoop(base_events.BaseEventLoop):
    """Selector event loop.

    See events.EventLoop for API specification.
    """
    ...

    # 登记一个可读事件发生后的回调，只要有一个非阻塞 fd，就可以挂到 asyncio 的调度体系里
    # low-level的api，https://docs.python.org/3/library/asyncio-eventloop.html#watch-a-file-descriptor-for-read-events
    def add_reader(self, fd, callback, *args):
        """Add a reader callback."""
        self._ensure_fd_no_transport(fd)
        return self._add_reader(fd, callback, *args)
    
    def _add_reader(self, fd, callback, *args):
        self._check_closed()
        handle = events.Handle(callback, args, self, None)
        try:
            key = self._selector.get_key(fd)
        except KeyError:
            self._selector.register(fd, selectors.EVENT_READ,
                                    (handle, None))
        else:
            mask, (reader, writer) = key.events, key.data
            self._selector.modify(fd, mask | selectors.EVENT_READ,
                                  (handle, writer))
            if reader is not None:
                reader.cancel()
    

    def add_writer(self, fd, callback, *args):
        ...
    def _add_writer(self, fd, callback, *args):
        ...

    # 触发执行（入loop._ready队列）多个可读/可写事件发生后的回调
    # event_list 来自 self._selector.select(timeout) 的返回值，形如：[(key1, mask1), (key2, mask2), ...]
    # key 是 selectors.SelectorKey，key.fileobj：注册的对象（通常是 socket / fd），key.data：注册时附带的数据（这里被用来挂 (reader, writer)）
    # mask 是位掩码，表示就绪类型，selectors.EVENT_READ 可读，selectors.EVENT_WRITE 可写
    # 把 reader 这个回调加入 loop._ready 队列（_add_callback）
    def _process_events(self, event_list):
        for key, mask in event_list:
            fileobj, (reader, writer) = key.fileobj, key.data
            if mask & selectors.EVENT_READ and reader is not None:
                if reader._cancelled:
                    self._remove_reader(fileobj)
                else:
                    self._add_callback(reader)
            if mask & selectors.EVENT_WRITE and writer is not None:
                if writer._cancelled:
                    self._remove_writer(fileobj)
                else:
                    self._add_callback(writer)
    ...
    def _add_callback(self, handle):
        """Add a Handle to _scheduled (TimerHandle) or _ready."""
        assert isinstance(handle, events.Handle), 'A Handle is required here'
        if handle._cancelled:
            return
        assert not isinstance(handle, events.TimerHandle)
        self._ready.append(handle)
```

---

## asyncio.sleep

理解一下`asyncio.sleep`是如何实现的，顺带将上面的知识点串起来

```python
# cpython/Lib/asyncio/tasks.py
async def sleep(delay, result=None, *, loop=None):
    """Coroutine that completes after a given time (in seconds)."""
    if delay <= 0:
        await __sleep0()
        return result

    if loop is None:
        loop = events.get_running_loop()
    else:
        warnings.warn("The loop argument is deprecated since Python 3.8, "
                      "and scheduled for removal in Python 3.10.",
                      DeprecationWarning, stacklevel=2)

    future = loop.create_future()
    h = loop.call_later(delay,
                        futures._set_result_unless_cancelled,
                        future, result)
    try:
        return await future
    finally:
        h.cancel()
```
然后在`_run_once()`中循环处理，处理最近的`timer`

`await asyncio.sleep(2)` 效果会把当前协程暂停然后等2秒继续执行，大致实现如下：
- 事件循环执行创建Task时封装的`Task.__step` handle，`coro.send(None)` 推进外层协程（`包含了 await asyncio.sleep(2)` 的协程）执行，外层协程会进入 sleep() 协程并一路跑到 `return await future`，然后在这里挂起，把等待对象future交回给`Task.__step`
- `asyncio.sleep`中创建了TimerHandle，处理为调用future的`set_result`，这个TimerHandle入`loop._scheduled`小根堆
- `Task.__step` handle 为获得的future添加了完成时的`add_done_callback`回调，用于`__wakeup`协程
- 每次事件循环执行`_run_once`，当`_ready`没有可以执行的任务时，通过`select(timeout)`实现阻塞的等待
- 然后满足时间要求，会将TimeHandle入`_ready`队列，最终执行这个future的`set_result`
- `Task.__step` handle 获得的future完成触发done的回调，`__wakeup`了协程（`add_done_callback`也是将回调经`loop.call_soon`入 `_ready`后执行）

---

## 异步爬虫

```python
import asyncio
import aiohttp

async def fetch(session: aiohttp.ClientSession, url: str) -> str:
    async with session.get(url) as resp:
        resp.raise_for_status()
        return await resp.text()

async def main(urls: list[str]):
    timeout = aiohttp.ClientTimeout(total=20)
    conn = aiohttp.TCPConnector(limit=100)  # 连接池上限
    async with aiohttp.ClientSession(timeout=timeout, connector=conn) as session:
        tasks = [asyncio.create_task(fetch(session, u)) for u in urls]
        pages = await asyncio.gather(*tasks, return_exceptions=True)
        for u, p in zip(urls, pages):
            if isinstance(p, Exception):
                print("FAIL", u, repr(p))
            else:
                print("OK  ", u, len(p))

if __name__ == "__main__":
    urls = ["https://example.com"] * 50
    asyncio.run(main(urls))
```