PEP:               0000
Title:             Unifying Future implementations
Version:           $Revision$
Last-Modified:     $Date$
Author:            Sergii Mikhtoniuk <mikhtonyuk@gmail.com>
Status:            None
Type:              Standards Track
Content-Type:      text/x-rst
Created:           21-Dec-2013
Python-Version:    3.X
Post-History:

========
Abstract
========

This is a proposal for unification of Future classes of ``concurrent.futures`` and ``asyncio`` packages which currently have similar interfaces but are completely unrelated types.

This proposal includes:

- implementing Future class hierarchy with two leaf classes for ``cooperative`` and ``multithreaded`` concurrency scenarios

- making ``asyncio.Future`` a concrete implementation of cooperative concurrency future

- providing common set of futures composition methods

- unifying callback scheduling mechanism

- unifying error handling strategy

==========
Motivation
==========

``Future`` is an abstraction for representing result of asynchronous operation. It is independent of the specific way asynchronous operation is performed, whether it is by means of multithreading or using coroutine-based cooperative concurrency.

Currently standard Python library has two ``Future`` types for cooperative and multithreaded concurrency: ``concurrent.futures.Future`` and ``asyncio.Future`` respectively. These types have similar interfaces, but they are completely unrelated.

As a consequence:

- It is hard to reason about interoperability of code with different concurrency models.

- Implementing computations on futures in concurrency-agnostic way is highly error-prone as implementation needs to rely soley on callbacks and duck-typing.

- Implementations have different error handling strategies.

- Composition methods such as ``concurrent.futures.wait`` and ``asyncio.wait`` are duplicated for different types of futures.

- Cooperative concurrency ``Future`` is tightly coupled with ``asyncio`` package.

- ``concurrent.future.Future`` is tightly coupled to provided executors.


Proposed Solution
-----------------

Proposed solution includes:

- Bringing cooperative and multithreaded futures under single type hierarchy.

- Defining ``concurrent.futures.cooperative.Future`` class for cooperative concurrency scenario, preserving interface of current ``asyncio.Future`` (purely callback-based, without any blocking methods).

- Defining ``concurrent.futures.multithreaded.Future`` class which ensures thread-safety of core ``Future`` state and adds blocking operations.

- Implementing common unhandled error policy and callback scheduling mechanism.

- Subclassing ``asyncio.Future`` from ``cooperative.Future`` while adding only methods specific to ``yield from``.

- Implementing ``concurrent.futures.Future`` in terms of ``concurrent.futures.multithreaded.Future`` for backward-compatibility, while deprecating its use (especially discouraging use of ``running()`` and ``set_running_or_notify_cancel()`` methods as posing to much restrictions on scheduler implementations).

- Implementing common set of ``Future`` composition methods (such as ``all()``, ``first()``, ``first_sucessful()`` etc.).


=============
Specification
=============

Proposed package defines following abstractions:

- ``Future`` - represents result of async operation, which is expected to be completed by some async system.

- ``Executor`` - a callable object used for scheduling callbacks. For example in cooperative concurrency scenario this can be ``event_loop.call_soon()`` method, and for multithreaded case it can be either ``thread pool`` for scheduling callbacks on different thread), or ``Synchronous`` scheduler for running callback in same context as processing (if acceptable).


Future type hierarchy
---------------------

``Future`` type hierarchy in package ``concurrent.futures`` consists of:

- ``FutureBase`` class, implementing core ``Future`` state handling, as well as callback maintenance and invocation.

- ``FutureBaseExt`` class, inherits ``FutureBase`` and adds composition and convenience methods on top of ``FutureBase`` interface.

- ``cooperative.Future`` class, inherits ``FutureBaseExt`` and playing role as base class for all cooperative concurrency ``Future`` implementations (such as ``asyncio.Future``).

- ``multithreaded.Future`` class, inherits ``FutureBaseExt`` and serves basic implementation for ``Futures`` used in multithreaded scenarios. It adds ability to for blocking on ``result()`` and ``exception()`` methods with configurable timeout.


FutureBase interface
--------------------

``FutureBase`` is a common interface between futures of all concurrency models. It only exposes callback subscription as a primary way of handling ``Future`` results.

``FutureBase`` interface is the same as of current implementation of ``asyncio.Future``, with the exception of new ``try_set_*`` convenience methods.

``__init__(*, clb_executor=None)``

	Initializes future instance. 

	``clb_executor`` specifies default executor object for scheduling callbacks (by default set from ``config.Default.CALLBACK_EXECUTOR``)

``add_done_callback(fun_res, *, executor=None)``

	Add a callback to be run when the future becomes done.

	The callback is called with a single argument - the future object. If the future is already done when this is called, the callback is scheduled with call_soon.

``remove_done_callback(fn)``

	Remove all instances of a callback from the "call when done" list.

	Returns the number of callbacks removed.

``cancelled()``

	Return True if the future was cancelled.

``done()``

	Return True if the future is done.

	Done means either that a result / exception are available, or that the future was cancelled.

``result()``

	Return the result this future represents.

	If the future has been cancelled, raises CancelledError. If the future's result isn't yet available, raises InvalidStateError. If the future is done and has an exception set, this exception is raised.

``exception()``

	Return the exception that was set on this future.

	The exception (or None if no exception was set) is returned only if the future is done.  If the future has been cancelled, raises CancelledError.  If the future isn't done yet, raises
	InvalidStateError.

``cancel()``

	Cancel the future and schedule callbacks.

	If the future is already done or cancelled, return False.  Otherwise, change the future's state to cancelled, schedule the callbacks and return True.

``set_result(result)``

	Mark the future done and set its result.

	If the future is already done when this method is called, raises InvalidStateError.

``try_set_result(result)``

	Attempts to mark the future done and set its result.

	Returns False if the future is already done when this method is called.

``set_exception(exception)``

	Mark the future done and set an exception.

	If the future is already done when this method is called, raises InvalidStateError.

``try_set_exception(exception)``

	Attempts to mark the future done and set an exception.

	Returns False if the future is already done when this method is called.

``set_from(other)``

	Copies result of another future into this one.

	Copies either result, exception, or cancelled state depending on how other future was completed.

	If this future is already done when this method is called, raises InvalidStateError. Other future should be done() before making this call.

``try_set_from(other)``
	
	Copies result of another future into this one.

	Copies either result, exception, or cancelled state depending on how other future was completed.

	Returns False if this future is already done when this method is called. Other future should be done() before making this call.


FutureBaseExt interface
-----------------------

``FutureBaseExt`` class adds various composition and convenience methods on top of ``FutureBase`` interface.

Composition methods allow making simple computations on ``Future`` result without resorting to blocking (with either ``yield from`` or timeouts). They also provide
means for combining multiple different futures together.

``recover(fun_ex_or_value, executor=None) -> Future``

	Returns future that will contain result of original if it completes successfully, or set from result of provided function in case of failure. New future inherits default callback executor from original future. Propagates exceptions from function as well as cancellation.

	fun_ex_or_value: function that accepts Exception parameter or just value to use in error case.
	parameter, executor - Executor to use when performing call to function.

``map(fun_res, executor=None) -> Future``

	Returns future which will be set from result of applying provided function to original future value. New future inherits default callback executor from original future. Propagates exceptions from function as well as cancellation.

	fun_res - function that accepts original result and returns new value,
	executor - Executor to use when performing call to function.

``then(future_fun, executor=None) -> Future``

	Returns future which represents two futures chained one after another. Failures are propagated from first future, from second future and from callback function. Cancellation is propagated both ways.

	future_fun - either function that returns future to be chained after successful completion of first one, or Future instance directly.
	executor - Executor to use when performing call to function.

``fallback(future_fun, executor=None) -> Future``

	Returns future that will contain result of original if it completes successfully, or will be set from future returned from provided function in case of failure. Provided function is called only if original future fails. Cancellation is propagated both ways.

	future_fun - either function that returns future to be used for fallback, or Future instance directly.
	executor - Executor to use when performing call to function.

``@classmethod all(futures, clb_executor=None) -> Future``

	Transforms list of futures into one future that will contain list of results. In case of any failure future will be failed with first exception to occur. Cancellation is propagated both ways - if aggregate future is cancelled it will cancel all child futures.

	futures - list of futures to combine
	clb_executor - default executor to use when running new future's callbacks.

``@classmethod first(futures, clb_executor=None) -> Future``

	Returns future which will be set from result of first future to complete, both successfully or with failure. Cancellation is propagated both ways - if aggregate future is cancelled it will cancel all child futures.

	futures - list of futures to combine.
	clb_executor - default executor to use when running new future's callbacks.

``@classmethod first_successful(futures, clb_executor=None) -> Future``

	Returns future which will be set from result of first future to complete successfully, last detected error will be set in case when all of the provided future fail. In case of cancelling aggregate future all child futures will be cancelled. Only cancellation of all child future triggers cancellation of aggregate future.

	futures - list of futures to combine.
	clb_executor - default executor to use when running new future's callbacks.

``@classmethod reduce(futures, fun, *vargs, executor=None, clb_executor=None) -> Future``

	Returns future which will be set with reduced result of all provided futures. In case of any failure future will be failed with first exception to occur. Cancellation is propagated both ways - if aggregate future is cancelled it will cancel all child futures.

	futures - list of futures to combine, fun - reduce-compatible function.
	executor - Executor to use when performing call to function.
	clb_executor - default executor to use when running new future's callbacks.

``@classmethod convert(future) -> Future``
	
	Performs future type conversion.
        
	It either makes sure that passed future is safe to use with current 
	future type, or raises TypeError indicating incompatibility.

	Override this method in leaf future classes to enable
	compatibility between different Future implementations.


cooperative.Future interface
------------------------------

``cooperative.Future`` inherits ``FutureBaseExt`` and does not have any additional operations. Its only purpose is to separate concurrent and multithreaded branches of hierarchy.


multithreaded.Future interface
------------------------------

``multithreaded.Future`` adds thread-safety and blocking operations to ``FutureBase`` methods.

``result(*, timeout=None)``

	Return the result this future represents. If the future has not yet been completed this method blocks for up to timeout seconds. If timeout is not specified it will block for unlimited time.

	If the future has been cancelled, raises CancelledError. If the future does not complete in specified time frame, raises TimeoutError. If the future is done and has an exception set, this exception is raised.

``exception(*, timeout=None)``

	Return the exception that was set on this future. If the future has not yet been completed this method blocks for up to timeout seconds. If timeout is not specified it will block for unlimited time.

	If the future has been cancelled, raises CancelledError. If the future does not complete in specified time frame, raises TimeoutError.


Executors
---------

``Future`` API allows two ways of specifying execution context for callbacks. First is by setting default callback executor on ``Future`` creation. Second is by specifying it on per-callback basis when registering the callback.

Controlling default callback context may be necessary for some services. For example if a service implements HTTP request multiplexing in dedicated thread, it's dangerous to run user's callbacks synchronously, because they may block or run for considerably long time. Such behavior would block the whole HTTP request processing thread. Such services can override default callback executor and set it to, for example, to ``ThreadPoolExecutor`` when returning future to client. This way every callback of this future will run in ``ThreadPool`` thread unless user specifies different executor explicitly.

``Executor`` is a callable object that receives function to execute:

``__call__(fn, *args, **kwargs)``

	Schedules function for execution.

By default all futures use ``SynchronousExecutor`` for all callbacks. This can be changed globally by setting ``config.Default.CALLBACK_EXECUTOR`` field on application startup, but generally default executor should be controlled on per-service basis.


Unhandled errors policy
-----------------------

Errors in async operations should not go unnoticed, that's why proposed package expects that all ``Future`` failures will be handled explicitly by callbacks, by getting ``result`` or ``exception``, or by use of composition methods. Unhandled error is detected when ``Future`` object is about to be deleted by GC, and none of the above methods were used to detect the exception. Unhandled error may also be result of uncaught exception in the callback itself.

To avoid exceptions being lost package defines default callback for unhandled failures that logs errors to ``logging.error`` by default. You can override this behavior by setting ``config.Default.UNHANDLED_FAILURE_CALLBACK`` field on application startup.

It is possible to ignore errors explicitly using ``f.recover(None)``.


Compatibility between different types of futures
------------------------------------------------

For applications which may use both cooperative and multithreaded concurrency models at the same time there's a high risk of mixing different future types in composition methods. If this goes unnoticed it may result in race conditions.

To prevent this, all composition methods of ``FutureBaseExt`` perform type checking, raising ``TypeError`` when class which was used for calling composition method does not match with class of ``Future`` instances. This check is performed by ``FutureBaseExt.convert(future)`` method.

However this restriction is too strict. For example it is safe to use callbacks of ``cooperative.Future`` in composition methods of ``multithreaded.Future``. Also ``multithreaded.Future`` can be easily wrapped into ``asyncio.Future`` by using ``loop.call_soon_threadsafe()`` method. To allow such compatibility and conversions ``multithreaded.Future`` and ``asyncio.Future`` override ``convert(future)`` method.

Resulting conversions are:

- ``multithreaded.Future`` compatible with all subclasses of ``concurrent.Future``

- ``asyncio.Future`` compatible with all subclasses of itself (``asyncio.Task``)

- ``asyncio.Future`` converts subclasses of ``multithreaded.Future`` via wrapping

As a result, this compatibility mechanism allows mixing futures of different types and nature in composition methods safely.


==============
Usage examples
==============

Computations
------------

::

	from concurrent.futures.multithreaded import Future
	from concurrent.executors import ThreadPoolExecutor

	with ThreadPoolExecutor(10) as tp:
		def sqr(x):
			return x * x

		fsquares = [tp.submit(sqr, v) for v in range(10)]
		fsum = Future.all(fsquares).map(sum)

		print(fsum.result())

Chaining requests
-----------------

::

	from concurrent.futures.multithreaded import Future

	def authenticate_async(login, pwd):
		return Future.successful(True)

	def request_async(request):
		return Future.successful(request)

	fauth = authenticate_async('john', 'swordfish')
	frequest = fauth.then(lambda: request_async('echo'))

	resul = yieldfrequest.on_success(lambda resp: print("auth and request successful: " + resp))
	frequest.on_failure(lambda ex: print("auth or request failed"))


Fallbacks
---------

::

	from concurrent.futures.multithreaded import Future

	def connect_ssl():
		return Future.failed(SocketError())

	def connect_plain():
		return Future.successful('socket')

	fconnection = connect_ssl().fallback(connect_plain)

Hedged requests
---------------

::

	import asyncio
	import random
	
	def request(ip):
		yield from asyncio.sleep(random.random())
		if random.choice([True, False]):
			return "ok"
		raise Exception("fail")

	IPs = ['ip1', 'ip2', 'ip3', 'ip4', 'ip5']
	futures = [asyncio.Task(request(ip)) for ip in IPs]
	response = yield from asyncio.Future.first_successful(futures)
	print(response)

======================
Backward compatibility
======================

Interface differences
---------------------

``concurrent.futures.Future``

	Class is deprecated in favor of ``concurrent.futures.multithreaded.Future``.

``running()``

	Left for compatibility but always returns ``False``.

``set_running_or_notify_cancel()``
	
	Left for compatibility but simply returns ``cancelled()`` result.

Module functions differences:

``concurrent.futures.wait()``

``concurrent.futures.as_completed()``

``asyncio.wait()``

``asyncio.gather()``

	Function are deprecated in favor of combination methods and use them internally.

``asyncio.wrap_future()``

	Deprecated in favor of ``Future.convert(future)``, all composition methods perform this conversion automatically.

Concrete implementations of schedulers are considered to be beyond the scope of ``concurrent.futures`` module, so they are moved into ``concurrent.schedulers`` sub-package. For backward compatibility ``ThreadPoolExecutor`` and ``ProcessPoolExecutor`` are still accessible from ``concurrent.futures`` package.


===================
Further development
===================

- ``Observable`` abstraction for asynchronous streams of data.
- ThreadPool-based scheduler allowing non-blocking future cancellation by timeout. 
- Debugging facilities for tracing and graphing future chains.


==========
References
==========

- PEP 3148 describes current implementation of ``concurrent.futures``
- PEP 3156 describes ``asyncio`` package implementing own futures


