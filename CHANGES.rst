===========
 Changelog
===========

.. currentmodule:: gevent

1.5a5 (unreleased)
==================

- Make `gevent.lock.RLock.acquire` accept the *timeout* parameter.
- Fix an ``AttributeError`` when wrapping gevent's ``FileObject``
  around an opened text stream. Reported in :issue:`1542` by
  dmrlawson.


1.5a4 (2020-03-23)
==================

Platform and Packaging Updates
------------------------------

- Python version updates: gevent is now tested with CPython 3.6.10,
  3.7.7 and 3.8.2. It is also tested with PyPy2 7.3 and PyPy 3.6
  7.3.

- The include directories used to compile the C extensions have been
  tweaked with the intent of making it easier to use older debug
  versions of Python. See :issue:`1461`.

- The binary wheels and installed package no longer include generated
  C source files and headers. Also excluded are Cython .pxd files,
  which were never documented. Please file an issue if you miss these.

Library and Dependency Updates
------------------------------

- Upgrade libev from 4.25 to 4.31 and update its embedded
  ``config.guess`` to the latest. See :issue:`1504`.

  .. important::

     libev, when built with ``EV_VERIFY >= 2``, now performs
     verification of file descriptors when IO watchers are started
     *and* when they are stopped. If you first close a file descriptor
     and only then stop an associated watcher, libev will abort the
     process.

     Using the standard gevent socket and file objects handles this
     automatically, but if you're using the IO watchers directly,
     you'll need to watch out for this.

     The binary wheels gevent distributes *do not* set ``EV_VERIFY``
     and don't have this issue.

- Make libuv and libev use the Python memory allocators. This assists
  with debugging. The event libraries allocate small amounts of memory
  at startup. The allocation functions have to take the GIL, but
  because of the limited amount of actual allocation that gets done
  this is not expected to be a bottleneck.

- Update the bundled ``tblib`` library to the unreleased 1.7.0
  version. The only change is to add more attributes to ``Frame`` and ``Code``
  objects for pytest compatibility. See :pr:`1541`.

Potentially Breaking Changes
----------------------------

- Remove the magic proxy object ``gevent.signal``. This served as both
  a deprecated alias of `gevent.signal_handler` *and* the module
  `gevent.signal`. This made it confusing to humans and static
  analysis tools alike. The alias was deprecated since gevent 1.1b4.
  See :issue:`1596`.

Other
-----

- Make `gevent.subprocess.Popen.communicate` raise exceptions raised
  by reading from the process, like the standard library. In
  particular, under Python 3, if the process output is being decoded
  as text, this can now raise ``UnicodeDecodeError``. Reported in
  :issue:`1510` by Ofer Koren.

- Make `gevent.subprocess.Popen.communicate` be more careful about
  closing files. Previously if a timeout error happened, a second call
  to ``communicate`` might not close the pipe.

- Add `gevent.contextvars`, a cooperative version of `contextvars`.
  This is available to all Python versions. On Python 3.7 and above,
  where `contextvars` is a standard library module, it is
  monkey-patched by default. See :issue:`1407`.

- Use `selectors.PollSelector` as the `selectors.DefaultSelector`
  after monkey-patching if `select.poll` was defined. Previously,
  gevent replaced it with `selectors.SelectSelector`, which has a
  different set of limitations (e.g., on certain platforms such as
  glibc Linux, it has a hardcoded limitation of only working with file
  descriptors < 1024). See :issue:`1466` reported by Sam Wong.

- Make the dnspython resolver work if dns python had been imported
  before the gevent resolver was initialized. Reported in
  :issue:`1526` by Chris Utz and Josh Zuech.


1.5a3 (2020-01-01)
==================

- Python version updates: gevent is now tested with CPython 2.7.17,
  3.5.9, 3.6.9, 3.7.5 and 3.8.0 (final). It is also tested with PyPy2
  7.2 and PyPy 3.6 7.2

- Fix using monkey-patched ``threading.Lock`` and ``threading.RLock``
  objects as spin locks by making them call ``sleep(0)`` if they
  failed to acquire the lock in a non-blocking call. This lets other
  callbacks run to release the lock, simulating preemptive threading.
  Using spin locks is not recommended, but may have been done in code
  written for threads, especially on Python 3. See :issue:`1464`.

- Fix Semaphore (and monkey-patched threading locks) to be fair. This
  eliminates the rare potential for starvation of greenlets. As part
  of this change, the low-level method ``rawlink`` of Semaphore,
  Event, and AsyncResult now always remove the link object when
  calling it, so ``unlink`` can sometimes be optimized out. See
  :issue:`1487`.

- Make ``gevent.pywsgi`` support ``Connection: keep-alive`` in
  HTTP/1.0. Based on :pr:`1331` by tanchuhan.

- Fix a potential crash using ``gevent.idle()`` when using libuv. See
  :issue:`1489`.

- Fix some potential crashes using libuv async watchers.

- Make ``ThreadPool`` consistently raise ``InvalidThreadUseError``
  when ``spawn`` is called from a thread different than the thread
  that created the threadpool. This has never been allowed, but was
  inconsistently enforced. On gevent 1.3 and before, this would always
  raise "greenlet error: invalid thread switch," or ``LoopExit``. On
  gevent 1.4, it *could* raise ``LoopExit``, depending on the number
  of tasks, but still, calling it from a different thread was likely
  to corrupt libev or libuv internals.

- Remove some undocumented, deprecated functions from the threadpool module.

- libuv: Fix a perceived slowness spawning many greenlets at the same time
  without yielding to the event loop while having no active IO
  watchers or timers. If the time spent launching greenlets exceeded
  the switch interval and there were no other active watchers, then
  the default IO poll time of about .3s would elapse between spawning
  batches. This could theoretically apply for any non-switching
  callbacks. This can be produced in synthetic benchmarks and other
  special circumstances, but real applications are unlikely to be
  affected. See :issue:`1493`.

- Fix using the threadpool inside a script or module run with ``python
  -m gevent.monkey``. Previously it would use greenlets instead of
  native threads. See :issue:`1484`.

- Fix potential crashes in the FFI backends if a watcher was closed
  and stopped in the middle of a callback from the event loop and then
  raised an exception. This could happen if the hub's ``handle_error``
  function was poorly customized, for example. See :issue:`1482`

- Make ``gevent.killall`` stop greenlets from running that hadn't been
  run yet. This make it consistent with ``Greenlet.kill()``. See
  :issue:`1473` reported by kochelmonster.

- Make ``gevent.spawn_raw`` set the ``loop`` attribute on returned
  greenlets. This lets them work with more gevent APIs, notably
  ``gevent.killall()``. They already had dictionaries, but this may
  make them slightly larger, depending on platform (on CPython 2.7
  through 3.6 there is no apparent difference for one attribute but on
  CPython 3.7 and 3.8 dictionaries are initially empty and only
  allocate space once an attribute is added; they're still smaller
  than on earlier versions though).


File Object Changes
-------------------

.. caution:: There may be breaking changes here for applications that
             relied on the old behaviour. The old behaviour was under
             specified and inconsistent and really only worked
             consistently with 'wb' and 'rb' modes, so most
             applications shouldn't be affected.

- The file objects (``FileObjectPosix``, ``FileObjectThread``) now
  consistently support text and binary modes. If neither 'b' nor 't' is given
  in the mode, they will read and write native strings. If 't' is
  given, they will always work with unicode strings, and 'b' will
  always work with byte strings. (``FileObjectPosix`` already worked this
  way.) See :issue:`1441`.

- The file objects accept *encoding*, *errors* and *newline*
  arguments. On Python 2, these are only used if 't' is in the mode.

- The default mode for ``FileObjectPosix`` changed from ``rb`` to simply
  ``r``, for consistency with the other file objects and the standard
  `open` and :func:`io.open` functions.

- Fix ``FileObjectPosix`` improperly being used from multiple
  greenlets. Previously this was hidden by forcing buffering, which
  raised ``RuntimeError``.


1.5a2 (2019-10-21)
==================

- Add support for CPython 3.8.0. (Windows wheels are not yet available.)

- Update to Cython 0.29.13 and cffi 1.12.3.

- Add an ``--module`` option to ``gevent.monkey`` allowing to run a Python
  module rather than a script. See :pr:`1440`.

- Improve the way joining the main thread works on Python 3.

- Implement ``SSLSocket.verify_client_post_handshake()`` when available.

- Fix tests when TLS1.3 is supported.

- Disable Nagle's algorithm in the backdoor server. This can improve
  interactive response time.

- Test on Python 3.7.4. There are important SSL test fixes.

1.5a1 (2019-05-02)
==================

Platform and Packaging Updates
------------------------------

- Python version updates: gevent is now tested with CPython 2.7.16,
  3.5.6, 3.6.8, and 3.7.2. It is also tested with PyPy2 7.1 and PyPy 3.6
  7.1 (PyPy 7.0 and 7.1 were not capable of running SSL tests on Travis
  CI).

- Support for Python 3.4 has been removed, as that version is no
  longer supported uptstream.

- gevent binary wheels are now manylinux2010 and include libuv
  support. pip 19 is needed to install them. See :issue:`1346`.

- gevent is now compiled with Cython 0.29.6 and cffi 1.12.2.

- gevent sources include a pyproject.toml file, specifying the build
  requirements and enabling build isolation. pip 18 or above is needed
  to take advantage of this. See :issue:`1180`.

- libev-cffi: Let the compiler fill in the definition of ``nlink_t`` for
  ``st_nlink`` in ``struct stat``, instead of trying to guess it
  ourself. Reported in :issue:`1372` by Andreas Schwab.

- Remove the ``Makefile``. Its most useful commands, ``make clean``
  and ``make distclean``, can now be accomplished in a cross-platform
  way using ``python setup.py clean`` and ``python setup.py clean
  -a``, respectively. The remainder of the ``Makefile`` contained
  Travis CI commands that have been moved to ``.travis.yml``.

- Deprecate the ``EMBED`` and ``LIBEV_EMBED``, etc, build-time environment
  variables. Instead, use ``GEVENTSETUP_EMBED`` and
  ``GEVENTSETUP_EMBED_LIBEV``. See :issue:`1402`.

- The CFFI backends now respect the embed build-time setting. This allows
  building the libuv backend without embedding libuv (except on Windows).

- Support test resources. This allows disabling tests that use the
  network. See :ref:`limiting-test-resource-usage` for more.


Library and Dependency Updates
------------------------------

- Upgrade libuv from 1.24.0 to 1.27.0.

- Upgrade libev from 4.23 to 4.25 and update its embedded
  ``config.guess`` to the latest.

- Upgrade c-ares from 1.14 to 1.15.

- dnspython >= 1.16.0 is now required for the dnspython resolver.

Bug fixes
---------

- Python 3.7 subprocess: Copy a ``STARTUPINFO`` passed as a parameter.
  Contributed by AndCycle in :pr:`1352`.

- subprocess: ``WIFSTOPPED`` and ``SIGCHLD`` are now handled for
  determining ``Popen.returncode``. See
  https://bugs.python.org/issue29335

- subprocess: No longer close redirected FDs if they are in
  ``pass_fds``. This is `a bugfix from Python 3.7
  <https://bugs.python.org/issue32270>`_ applied to all versions
  gevent runs on.

- Fix certain operations on a Greenlet in an invalid state (with an
  invalid parent) to raise a `TypeError` sooner rather than an
  `AttributeError` later. This is also slightly faster on CPython with
  Cython. Inspired by :issue:`1363` as reported by Carson Ip. This
  means that some extreme corner cases that might have passed by
  replacing a Greenlet's parent with something that's not a gevent hub
  now no longer will.

- Fix: The ``spawning_stack`` for Greenlets on CPython should now have
  correct line numbers in more cases. See :pr:`1379`.

Enhancements
------------

- The result of ``gevent.ssl.SSLSocket.makefile()`` can be used as a
  context manager on Python 2.

- Python 2: If the backport of the ``_thread_`` module from
  ``futures`` has already been imported at monkey-patch time, also
  patch this module to be consistent. The ``pkg_resources`` package
  imports this, and ``pkg_resources`` is often imported early on
  Python 2 for namespace packages, so if ``futures`` is installed this
  will likely be the case.

- Python 2: Avoid a memory leak when an `io.BufferedWriter` is wrapped
  around a socket. Reported by Damien Tournoud in :issue:`1318`.

- Avoid unbounded memory usage when creating very deep spawn trees.
  Reported in :issue:`1371` by dmrlawson.

- Win: Make ``examples/process.py`` do something useful. See
  :pr:`1378` by Robert Iannucci.

- Spawning greenlets can be up to 10% faster. See :pr:`1379`.


1.4.0 (2019-01-04)
==================

- Build with Cython 0.29 in '3str' mode.

- Test with PyPy 6.0 on Windows.

- Add support for application-wide callbacks when ``Greenlet`` objects
  are started. See :pr:`1289`, provided by Yury Selivanov.

- Fix consuming a single ready object using
  ``next(gevent.iwait(objs))``. Previously such a construction would
  hang because `iter` was not called. See :pr:`1288`, provided by Josh
  Snyder. This is not recommended, though, as unwaited objects will
  have dangling links (but see next item).

- Make `gevent.iwait` return an iterator that can now also be used as
  a context manager. If you'll only be consuming part of the iterator,
  use it in a ``with`` block to avoid leaking resources. See
  :pr:`1290`, provided by Josh Snyder.

- Fix semaphores to immediately notify links if they are ready and
  ``rawlink()`` is called. This behaves like ``Event`` and
  ``AsyncEvent``. Note that the order in which semaphore links are
  called is not specified. See :issue:`1287`, reported by Dan Milon.

- Improve safety of handling exceptions during interpreter shutdown.
  See :issue:`1295` reported by BobDenar1212.

- Remove the deprecated ability to specify ``GEVENT_RESOLVER`` and
  other importable settings as a ``path/to/a/package.module.item``.
  This had race conditions and didn't work with complicated resolver
  implementations. Place the required package or module on `sys.path`
  first.

- Reduce the chances that using the blocking monitor functionality
  could result in apparently random ``SystemError:
  Objects/tupleobject.c: bad argument to internal function``. Reported
  in :issue:`1302` by Ulrich Petri.

- Refactored the gevent test runner and test suite to make them more
  reusable. In particular, the tests are now run with ``python -m
  gevent.tests``. See :issue:`1293`.

- Make a monkey-patched ``socket.getaddrinfo`` return socket module
  enums instead of plain integers for the socket type and address
  family on Python 3. See :issue:`1310` reported by TheYOSH.

  .. note:: This requires Python 3.4.3 or above due to an undocumented
            change in that version.

- Make gevent's pywsgi server set the non-standard environment value
  ``wsgi.input_terminated`` to True. See :issue:`1308`.

- Make `gevent.util.assert_switches` produce more informative messages
  when the assertion fails.

- Python 2: If a `gevent.socket` was closed asynchronously (in a
  different greenlet or a hub callback), `AttributeError` could result
  if the socket was already in use. Now the correct socket.error
  should be raised.

- Fix :meth:`gevent.threadpool.ThreadPool.join` raising a
  `UserWarning` when using the libuv backend. Reported in
  :issue:`1321` by ZeroNet.

- Fix ``FileObjectPosix.seek`` raising `OSError` when it should have
  been `IOError` on Python 2. Reported by, and PR by, Ricardo Kirkner.
  See :issue:`1323`.

- Upgrade libuv from 1.23.2 to 1.24.0.

1.3.7 (2018-10-12)
==================

- Formatting run info no longer includes ``gevent.local.local``
  objects that have no value in the greenlet. See :issue:`1275`.

- Fixed negative length in pywsgi's Input read functions for non chunked body.
  Reported in :issue:`1274` by tzickel.

- Upgrade libuv from 1.22.0 to 1.23.2.

- Fix opening files in text mode in CPython 2 on Windows by patching
  libuv. See :issue:`1282` reported by wiggin15.

1.3.6 (2018-08-17)
==================

- gevent now depends on greenlet 0.4.14 or above. gevent binary wheels
  for 1.3.5 and below must have greenlet 0.4.13 installed on Python
  3.7 or they will crash. Reported by Alexey Stepanov in :issue:`1260`
  and pkittenis in :issue:`1261`.

- :class:`gevent.local.local` subclasses correctly supports
  ``@staticmethod`` functions. Reported by Brendan Powers in
  :issue:`1266`.


1.3.5 (2018-07-16)
==================

- Update the bundled libuv from 1.20.1 to 1.22.0.

- Test Python 3.7 on Appveyor. Fix the handling of Popen's
  ``close_fds`` argument on 3.7.

- Update Python versions tested on Travis, including PyPy to 6.0. See :issue:`1195`.

- :mod:`gevent.queue` imports ``_PySimpleQueue`` instead of
  ``SimpleQueue`` so that it doesn't block the event loop.
  :func:`gevent.monkey.patch_all` makes this same substitution in
  :mod:`queue`. This fixes issues with
  :class:`concurrent.futures.ThreadPoolExecutor` as well. Reported in
  :issue:`1248` by wwqgtxx and :issue:`1251` by pyld.

- :meth:`gevent.socket.socket.connect` doesn't pass the port (service)
  to :func:`socket.getaddrinfo` when it resolves an ``AF_INET`` or
  ``AF_INET6`` address. (The standard library doesn't either.) This
  fixes an issue on Solaris. Reported in :issue:`1252` by wiggin15.

- :meth:`gevent.socket.socket.connect` works with more address
  families, notably AF_TIPC, AF_NETLINK, AF_BLUETOOTH, AF_ALG and AF_VSOCK.


1.3.4 (2018-06-20)
==================

- Be more careful about issuing ``MonkeyPatchWarning`` for ssl
  imports. Now, we only issue it if we detect the one specific
  condition that is known to lead to RecursionError. This may produce
  false negatives, but should reduce or eliminate false positives.

- Based on measurements and discussion in :issue:`1233`, adjust the
  way :mod:`gevent.pywsgi` generates HTTP chunks. This is intended to
  reduce network overhead, especially for smaller chunk sizes.

- Additional slight performance improvements in :mod:`gevent.pywsgi`.
  See :pr:`1241`.

1.3.3 (2018-06-08)
==================

- :func:`gevent.sleep` updates the loop's notion of the current time
  before sleeping so that sleep duration corresponds more closely to
  elapsed (wall clock) time. :class:`gevent.Timeout` does the same.
  Reported by champax and FoP in :issue:`1227`.

- Fix an ``UnboundLocalError`` in SSL servers when wrapping a socket
  throws an error. Reported in :issue:`1236` by kochelmonster.

1.3.2.post0 (2018-05-30)
========================

- Fix a packaging error in manylinux binary wheels that prevented some
  imports from working. See :issue:`1219`.


1.3.2 (2018-05-29)
==================

- Allow weak refeneces to :class:`gevent.queue.Queue`. Reported in
  :issue:`1217` by githrdw.


1.3.1 (2018-05-18)
==================

- Allow weak references to :class:`gevent.event.Event`. Reported in
  :issue:`1211` by Matias Guijarro.

- Fix embedded uses of :func:`gevent.Greenlet.spawn`, especially under
  uwsgi. Reported in :issue:`1212` by Kunal Gangakhedkar.

- Fix :func:`gevent.os.nb_write` and :func:`gevent.os.nb_read` not
  always closing the IO event they opened in the event of an
  exception. This would be a problem especially for libuv.

1.3.0 (2018-05-11)
==================

- Python 3.7 passes the automated memory leak checks. See :issue:`1197`.

- Update autoconf's config.guess and config.sub to the latest versions
  for c-ares and libev.

- :class:`gevent.local.local` subclasses that mix-in ABCs can be instantiated.
  Reported in :issue:`1201` by Bob Jordan.

1.3b2 (2018-05-03)
==================

- On Windows, CFFI is now a dependency so that the libuv backend
  really can be used by default.

- Fix a bug detecting whether we can use the memory monitoring
  features when psutil is not installed.

- `gevent.subprocess.Popen` uses ``/proc/self/fd`` (on Linux) or
  ``/dev/fd`` (on BSD, including macOS) to find the file descriptors
  to close when ``close_fds`` is true. This matches an optimization
  added to Python 3 (and backports it to Python 2.7), making process
  spawning up to 9 times faster. Also, on Python 3, since Python 3.3
  is no longer supported, we can also optimize the case where
  ``close_fds`` is false (not the default), making process spawning up
  to 38 times faster. Initially reported in :issue:`1172` by Ofer Koren.

- The bundled libuv is now 1.20.1, up from 1.19.2. See :issue:`1177`.

- The long-deprecated and undocumented module ``gevent.wsgi`` was removed.

- Add `gevent.util.assert_switches` to build on the monitoring
  functions. Fixes :issue:`1182`.

- A started monitor thread for the active hub now survives a fork. See
  :issue:`1185`.

- The greenlet tracer functions used for the various monitoring
  capabilities are now compiled with Cython for substantially lower
  overhead. See :pr:`1190`.

- libuv now collects all pending watchers and runs their callbacks at
  the end of the loop iteration using UV_RUN_ONCE. This eliminates the
  need to patch libuv to be greenlet-safe. It also means that
  zero-duration timer watchers are actual timer watchers again
  (instead of being turned into check watchers); newly added
  zero-duration timers cannot block the event loop because they won't
  be run until a safe time.

- Python 3.7.0b4 is now the tested and supported version of Python
  3.7. PyPy 6.0 has been tested, although CI continues to use 5.10.

1.3b1 (2018-04-13)
==================

Dependencies
------------

- Cython 0.28.2 is now used to build gevent from a source checkout.

- The bundled libuv is now 1.19.2, up from 1.18.0.

Platform Support
----------------

- Travis CI tests on Python 3.7.0b3.

- Windows now defaults to the libuv backend if CFFI is installed. See
  :issue:`1163`.

Bug Fixes
---------

- On Python 2, when monkey-patching `threading.Event`, also
  monkey-patch the underlying class, ``threading._Event``. Some code
  may be type-checking for that. See :issue:`1136`.

- Fix libuv io watchers polling for events that only stopped watchers
  are interested in, reducing CPU usage. Reported in :issue:`1144` by
  wwqgtxx.

Enhancements
------------

- Add additional optimizations for spawning greenlets, making it
  faster than 1.3a2.

- Use strongly typed watcher callbacks in the libuv CFFI extensions.
  This prevents dozens of compiler warnings.

- When gevent prints a timestamp as part of an error message, it is
  now in UTC format as specified by RFC3339.

- Threadpool threads that exit now always destroy their hub (if one
  was created). This prevents some forms of resource leaks (notably
  visible as blocking functions reported by the new monitoring abilities).

- Hub objects now include the value of their ``name`` attribute in
  their repr.

- Pools for greenlets and threads have lower overhead, especially for
  ``map``. See :pr:`1153`.

- The undocumented, internal implementation classes ``IMap`` and
  ``IMapUnordered`` classes are now compiled with Cython, further
  reducing the overhead of ``[Thread]Pool.imap``.

- The classes `gevent.event.Event` and `gevent.event.AsyncResult`
  are compiled with Cython for improved performance, as is the
  ``gevent.queue`` module and ``gevent.hub.Waiter`` and certain
  time-sensitive parts of the hub itself. Please report any
  compatibility issues.

- ``python -m gevent.monkey <script>`` accepts more values for
  ``<script>``, including paths to packages or compiled bytecode.
  Reported in :issue:`1157` by Eddie Linder.

- Add a simple event framework for decoupled communication. It uses
  :mod:`zope.event` if that is installed.

- :mod:`gevent.monkey` has support for plugins in the form of event
  subscribers and setuptools entry points. See :pr:`1158` and
  :issue:`1162`. setuptools must be installed at runtime for its entry
  points to function.

Monitoring and Debugging
------------------------

- Introduce the configuration variable
  `gevent.config.track_greenlet_tree` (aka
  ``GEVENT_TRACK_GREENLET_TREE``) to allow disabling the greenlet tree
  features for applications where greenlet spawning is performance
  critical. This restores spawning performance to 1.2 levels.

- Add an optional monitoring thread for each hub. When enabled, this
  thread (by default) looks for greenlets that block the event loop
  for more than 0.1s. You can add your own periodic monitoring
  functions to this thread. Set ``GEVENT_MONITOR_THREAD_ENABLE`` to
  use it, and ``GEVENT_MAX_BLOCKING_TIME`` to configure the blocking
  interval.

- The monitoring thread emits events when it detects certain
  conditions, like loop blocked or memory limits exceeded.

- Add settings for monitoring memory usage and emitting events when a
  threshold is exceeded and then corrected. gevent currently supplies
  no policy for what to do when memory exceeds the configured limit.
  ``psutil`` must be installed to use this. See :pr:`1150`.


1.3a2 (2018-03-06)
==================

Dependencies
------------

- Cython 0.28b1 or later is now required to build gevent from a source
  checkout (Cython is *not* required to build a source distribution
  from PyPI).

- Update c-ares to 1.14.0. See :issue:`1105`.

- gevent now **requires** the patched version of libuv it is
  distributed with. Building gevent with a non-embedded libuv, while
  not previously supported, is not possible now. See
  :issue:`1126`.


Platform Support
----------------

- Travis CI tests on Python 3.7.0b2 and PyPy 2.7 5.10.0 and PyPy 3.5
  5.10.1.

Build Changes
-------------

- Fix building from a source distribution (PyPI) without Cython
  installed.

Enhancements
------------

- Add the ``dnspython`` resolver as a lightweight alternative to
  c-ares. It is generally faster than c-ares and is supported on PyPy.
  c-ares may be deprecated in the future. See :pr:`1088` and
  :issue:`1103`.

- Add the module :mod:`gevent.time` that can be imported instead of
  :mod:`time`, much like :mod:`gevent.socket` can be imported instead
  of :mod:`socket`. It contains ``gevent.sleep``. This aids
  monkey-patching.

- Simple subclasses of `gevent.local.local` now have the same
  (substantially improved) performance characteristics of plain
  `gevent.local.local` itself, making them 2 to 3 times faster than
  before. See :pr:`1117`. If there are any compatibility
  problems, please open issues.

Monitoring and Debugging
~~~~~~~~~~~~~~~~~~~~~~~~

- Greenlet objects now keep track of their spawning parent greenlet
  and the code location that spawned them, in addition to maintaining
  a "spawn tree local" mapping. This adds some runtime overhead in
  relative terms, but absolute numbers are still relatively small.
  Based on a proposal from PayPal and comments by Mahmoud Hashemi and
  Kurt Rose. See :issue:`755` and :pr:`1115`. As always, feedback is
  appreciated.

- Greenlet objects now have a `minimal_ident
  <gevent.Greenlet.minimal_ident>` property. It functions
  similarly to ``Thread.ident`` or ``id`` by uniquely identifying the
  greenlet object while it remains alive, and it can be reused after
  the greenlet object is dead. It is different in that it is small and
  sequential. Based on a proposal from PayPal and comments by Mahmoud
  Hashemi and Kurt Rose. See :issue:`755`. As always, feedback is
  appreciated.

- `gevent.Greenlet` objects now have a `gevent.Greenlet.name`
  attribute that is included in the default repr.

- Include the values of `gevent.local.local` objects associated with
  each greenlet in `gevent.util.format_run_info`.

- Add `gevent.util.GreenletTree` to visualize the greenlet tree. This
  is used by `gevent.util.format_run_info`.


Subprocess
~~~~~~~~~~

- Make :class:`gevnt.subprocess.Popen` accept the ``restore_signals``
  keyword argument on all versions of Python, and on Python 2 have it
  default to false. It previously defaulted to true on all versions;
  now it only defaults to true on Python 3. The standard library in
  Python 2 does not have this argument and its behaviour with regards
  to signals is undocumented, but there is code known to rely on
  signals not being restored under Python 2. Initial report and patch
  in :pr:`1063` by Florian Margaine.

- Allow :class:`gevent.subprocess.Popen` to accept the keyword
  arguments ``pass_fds`` and ``start_new_session`` under Python 2.
  They have always had the same default as Python 3, namely an empty
  tuple and false, but now are accessible to Python 2.

- Support the ``capture_output`` argument added to Python 3.7 in
  :func:`gevent.subprocess.run`.

Configuration
~~~~~~~~~~~~~

- Centralize all gevent configuration in an object at
  ``gevent.config``, allowing for gevent to be configured through code
  and not *necessarily* environment variables, and also provide a
  centralized place for documentation. See :issue:`1090`.

  - The new ``GEVENT_CORE_CFFI_ONLY`` environment variable has been
    replaced with the pre-existing ``GEVENT_LOOP`` environment
    variable. That variable may take the values ``libev-cext``,
    ``libev-cffi``, or ``libuv-cffi``, (or be a list in preference
    order, or be a dotted name; it may also be assigned to an
    object in Python code at ``gevent.config.loop``).

  - The ``GEVENTARES_SERVERS`` environment variable is deprecated in
    favor of ``GEVENT_RESOLVER_SERVERS``. See :issue:`1103`.


Bug Fixes
---------

- Fix calling ``shutdown`` on a closed socket. It was raising
  ``AttributeError``, now it once again raises the correct
  ``socket.error``. Reported in :issue:`1089` by André Cimander.

- Fix an interpreter crash that could happen if two or more ``loop``
  objects referenced the default event loop and one of them was
  destroyed and then the other one destroyed or (in the libev C
  extension implementation only) deallocated (garbage collected). See
  :issue:`1098`.

- Fix a race condition in libuv child callbacks. See :issue:`1104`.

Other Changes
-------------

- The internal, undocumented module ``gevent._threading`` has been
  simplified.

- The internal, undocumented class ``gevent._socket3._fileobject`` has
  been removed. See :issue:`1084`.

- Simplify handling of the libev default loop and the ``destroy()``
  method. The default loop, when destroyed, can again be requested and
  it will regenerate itself. The default loop is the only one that can
  receive child events.

- Make :meth:`gevent.socket.socket.sendall` up to ten times faster on
  PyPy3, through the same change that was applied in gevent 1.1b3 for PyPy2.

- Be more careful about issuing a warning about patching SSL on
  Python 2. See :issue:`1108`.

- Signal handling under PyPy with libuv is more reliable. See
  :issue:`1112`.

- The :mod:`gevent.greenlet` module is now compiled with Cython to
  offset any performance decrease due to :issue:`755`. Please open
  issues for any compatibility concerns. See :pr:`1115` and :pr:`1120`.

- On CPython, allow the pure-Python implementations of
  `gevent.Greenlet`, `gevent.local` and `gevent.lock` to be
  used when the environment variable ``PURE_PYTHON`` is set. This is
  not recommended except for debugging and testing. See :issue:`1118`.

- :meth:`gevent.select.poll.poll` now interprets a *timeout* of -1 the
  same as a *timeout* of *None* as the standard requires. Previously,
  on libuv this was interpreted the same as a *timeout* of 0. In
  addition, all *timeout* values less than zero are interpreted like
  *None* (as they always were under libev). See :issue:`1127`.

- Monkey-patching now defaults to patching ``threading.Event``.

1.3a1 (2018-01-27)
==================

Dependencies
------------

- gevent is now built and tested with Cython 0.27. This is required
  for Python 3.7 support.

- Update c-ares to 1.13.0. See :issue:`990`.


Platform Support
----------------

- Add initial support for Python 3.7a3. It has the same level of
  support as Python 3.6.

  - Using unreleased Cython 0.28 and greenlet 0.4.13; requires Python 3.7a3.

  - The ``async`` functions and classes have been renamed to
    ``async_`` due to ``async`` becoming a keyword in Python 3.7.
    Aliases are still in place for older versions. See :issue:`1047`.

- gevent is now tested on Python 3.6.4. This includes the following
  fixes and changes:

  - Errors raised from :mod:`gevent.subprocess` will have a
    ``filename`` attribute set.
  - The :class:`threading.Timer` class is now monkey-patched and can
    be joined. Previously on Python 3.4 and above, joining a ``Timer``
    would hang the process.
  - :meth:`gevent.ssl.SSLSocket.unwrap` behaves more like the standard
    library, including returning a SSLSocket and allowing certain
    timeout-related SSL errors to propagate. The added standard
    library tests ``test_ftplib.py`` now passes.
  - :class:`gevent.subprocess.Popen` accepts a "path-like object" for
    the *cwd* parameter on all platforms. Previously this only worked
    on POSIX platforms under Python 3.6. Now it also works on Windows under
    Python 3.6 (as expected) and is backported to all previous versions.

- Linux CI now tests on PyPy3 3.5-5.9.0, updated from PyPy3 3.5-5.7.1.
  See :issue:`1001`. PyPy2 has been updated to 5.9.0 from 5.7.1,
  Python 2.7 has been updated to 2.7.14 from 2.7.13, Python 3.4 is
  updated to 3.4.7 from 3.4.5, Python 3.5 is now 3.5.4 from 3.5.3, and
  Python 3.6 is now 3.6.4 from 3.6.0.

- Drop support for Python 3.3. The documentation has only claimed
  support for 3.4+ since gevent 1.2 was released, and only 3.4+ has
  been tested. This merely removes the supporting Trove classifier and
  remaining test code. See :issue:`997`.

- PyPy is now known to run on Windows using the libuv backend, with
  caveats. See the section on libuv for more information.

- Due to security concerns, official support for Python 2.7.8 and
  earlier (without a modern SSL implementation) has been dropped.
  These versions are no longer tested with gevent, but gevent can
  still be installed on them. Supporting code will be removed in the
  next major version of gevent. See :issue:`1073`.

Build Changes
-------------

- When building gevent from a source checkout (*not* a distributed
  source distribution), ``make`` is no longer required and the
  ``Makefile`` is not used. Neither is an external ``cython`` command.
  Instead, the ``cythonize`` function is used, as recommended by
  Cython. (The external commands were never required by source
  distributions.) See :issue:`1076`.

- :class:`gevent.local.local` is compiled with Cython on CPython.

- The Cython ares 'channel' class is no longer declared to be publicly
  accessible from a named C structure. Doing so caused a conflict with
  the c-ares header files.

Bug Fixes
---------

- If a single greenlet created and destroyed many
  :class:`gevent.local.local` objects without ever exiting, there
  would be a leak of the function objects intended to clean up the
  locals after the greenlet exited. Introduce a weak reference to
  avoid that. Reported in :issue:`981` by Heungsub Lee.

- pywsgi also catches and ignores by default
  :const:`errno.WSAECONNABORTED` on Windows. Initial patch in
  :pr:`999` by Jan van Valburg.

- :meth:`gevent.subprocess.Popen.communicate` returns the correct type
  of str (not bytes) in universal newline mode under Python 3, or when
  an encoding has been specified. Initial patch in :pr:`939` by
  William Grzybowski.

- :meth:`gevent.subprocess.Popen.communicate` (and in general,
  accessing ``Popen.stdout`` and ``Popen.stderr``) returns the correct
  type of str (bytes) in universal newline mode under Python 2.
  Previously it always returned unicode strings. Reported in
  :issue:`1039` by Michal Petrucha.

- :class:`gevent.fileobject.FileObjectPosix` returns native strings in
  universal newline mode on Python 2. This is consistent with what
  :class:`.FileObjectThread` does. See :issue:`1039`.

- ``socket.send()`` now catches ``EPROTOTYPE`` on macOS to handle a race
  condition during shutdown. Fixed in :pr:`1035` by Jay Oster.

- :func:`gevent.socket.create_connection` now properly cleans up open
  sockets if connecting or binding raises a :exc:`BaseException` like
  :exc:`KeyboardInterrupt`, :exc:`greenlet.GreenletExit` or
  :exc:`gevent.timeout.Timeout`. Reported in :issue:`1044` by
  kochelmonster.


Other Changes
-------------

- ``Pool.add`` now accepts ``blocking`` and ``timeout`` parameters,
  which function similarly to their counterparts in ``Semaphore``.
  See :pr:`1032` by Ron Rothman.

- Defer adjusting the stdlib's list of active threads until
  ``threading`` is monkey patched. Previously this was done when
  :mod:`gevent.threading` was imported. That module is documented to
  be used as a helper for monkey patching, so this should generally
  function the same, but some applications ignore the documentation
  and directly import that module anyway.

  A positive consequence is that ``import gevent.threading, threading;
  threading.current_thread()`` will no longer return a DummyThread
  before monkey-patching. Another positive consequence is that PyPy
  will no longer print a ``KeyError`` on exit if
  :mod:`gevent.threading` was imported *without* monkey-patching.

  See :issue:`984`.

- Specify the Requires-Python metadata for improved installation
  support in certain tools (setuptools v24.2.1 or newer is required).
  See :issue:`995`.

- Monkey-patching after the :mod:`ssl` module has been imported now
  prints a warning because this can produce ``RecursionError``.

- :class:`gevent.local.local` objects are now approximately 3.5 times faster
  getting, setting and deleting attributes on PyPy. This involved
  implementing more of the attribute protocols directly. Please open
  an issue if you have any compatibility problems. See :issue:`1020`.

- :class:`gevent.local.local` is compiled with Cython on CPython. It
  was already 5 to 6 times faster due to the work on :issue:`1020`,
  and compiling it with Cython makes it another 5 to 6 times faster,
  for a total speed up of about 35 times. It is now in the same
  ballpark as the native :class:`threading.local` class. It also uses
  one pointer less memory per object, and one pointer less memory per
  greenlet. See :pr:`1024`.

- More safely terminate subprocesses on Windows with
  :meth:`gevent.subprocess.Popen.terminate`. Reported in :issue:`1023`
  by Giacomo Debidda.

- gevent now uses cffi's "extern 'Python'" callbacks. These should be
  faster and more stable. This requires at least cffi 1.4.0. See :issue:`1049`.

- gevent now approximately tries to stick to a scheduling interval
  when running callbacks, instead of simply running a count of
  callbacks. The interval is determined by
  :func:`gevent.getswitchinterval`. On Python 3, this is the same as
  the thread switch interval. On Python 2, this defaults to 0.005s and
  can be changed with :func:`gevent.setswitchinterval`. This should
  result in more fair "scheduling" of greenlets, especially when
  ``gevent.sleep(0)`` or other busy callbacks are in use. The interval
  is checked every 50 callbacks to keep overhead low. See
  :issue:`1072`. With thanks to Arcadiy Ivanov and Antonio Cuni.

libuv
-----

- Add initial *experimental* support for using libuv as a backend
  instead of libev, controlled by setting the environment variable
  ``GEVENT_CORE_CFFI_ONLY=libuv`` before importing gevent. This
  suffers a number of limitations compared to libev, notably:

  - libuv support is not available in the manylinux wheels uploaded to
    PyPI. The manylinux specification requires glibc 2.5, while libuv
    requires glibc 2.12. Install from source to access libuv on Linux
    (e.g., pip's ``--no-binary`` option).

  - Timers (such as ``gevent.sleep`` and ``gevent.Timeout``) only
    support a resolution of 1ms (in practice, it's closer to 1.5ms).
    Attempting to use something smaller will automatically increase it
    to 1ms and issue a warning. Because libuv only supports
    millisecond resolution by rounding a higher-precision clock to an
    integer number of milliseconds, timers apparently suffer from more
    jitter.

  - Using negative timeouts may behave differently from libev.

  - libuv blocks delivery of all signals, so signals are handled using
    an (arbitrary) 0.3 second timer. This means that signal handling
    will be delayed by up to that amount, and that the longest the
    event loop can sleep in the operating system's ``poll`` call is
    that amount. Note that this is what gevent does for libev on
    Windows too.

  - libuv only supports one io watcher per file descriptor, whereas
    libev and gevent have always supported many watchers using
    different settings. The libev behaviour is emulated at the Python
    level, but that adds overhead.

  - Looping multiple times and expecting events for the same file
    descriptor to be raised each time without any data being read or
    written (as works with libev) does not appear to work correctly on
    Linux when using ``gevent.select.poll`` or a monkey-patched
    ``selectors.PollSelector``.

  - The build system does not support using a system libuv; the
    embedded copy must be used. Using setuptools to compile libuv was
    the most portable method found.

  - If anything unexpected happens, libuv likes to ``abort()`` the
    entire process instead of reporting an error. For example, closing
    a file descriptor it is using in a watcher may cause the entire
    process to be exited.

  - There may be occasional otherwise unexplained and hard to
    duplicate crashes. If you can duplicate a crash, **please** submit
    an issue.

  - This is the only backend that PyPy can use on Windows. As of this
    alpha, there are many known issues with non-blocking sockets
    (e.g., as used by :mod:`asyncore`; see ``test_ftplib.py``) and
    sometimes sockets not getting closed in a timely fashion
    (apparently; see ``test_httpservers.py``) and communicating with
    subprocesses (it always hangs). Help tracking those down would be
    appreciated. Only PyPy2 is tested.

  Feedback and pull requests are welcome, especially to address the
  issues mentioned above.

  Other differences include:

  - The order in which timers and other callbacks are invoked may be
    different than in libev. In particular, timers and IO callbacks
    happen in a different order, and timers may easily be off by up to
    half of the supposed 1ms resolution. See :issue:`1057`.

  - Starting a ``timer`` watcher does not update the loop's time by
    default. This is because, unlike libev, a timer callback can cause
    other timer callbacks to be run if they expire because the loop's
    time updated, without cycling the event loop. See :issue:`1057`.

    libev has also been changed to follow this behaviour.

    Also see :issue:`1072`.

  - Timers of zero duration do not necessarily cause the event loop to
    cycle, as they do in libev. Instead, they may be called
    immediately. If zero duration timers are added from other zero
    duration timer callbacks, this can lead the loop to appear to
    hang, as no IO will actually be done.

    To mitigate this issue, ``loop.timer()`` detects attempts to use
    zero duration timers and turns them into a check watcher. check
    watchers do not support the ``again`` method.

  - All watchers (e.g., ``loop.io``) and the ``Timeout`` class have a
    ``close`` method that should be called when code is done using the
    object (they also function as context managers and a ``with``
    statement will automatically close them). gevent does this
    internally for sockets, file objects and internal timeouts.
    Neglecting to close an object may result in leaking native
    resources. To debug this, set the environment variables
    ``GEVENT_DEBUG=debug`` and ``PYTHONTRACEMALLOC=n`` before starting
    the process.

    The traditional cython-based libev backend will not leak if
    ``close`` is not called and will not produce warnings. The
    CFFI-based libev backend will not currently leak but will produce
    warnings. The CFFI-based libuv backend may leak and will produce
    warnings.

  Again, this is extremely experimental and all of it is subject to
  change.

  See :issue:`790` for history and more in-depth discussion.

libev
-----

- The C extension has been updated to use more modern Cython idioms
  and generate less code for simplicity, faster compilation and better
  cache usage. See :pr:`1077`.

  - Watcher objects may be slightly larger. On a 64-bit platform, a
    typical watcher may be 16 bytes (2 pointers) larger. This is
    offset by slight performance gains.

  - Cython is no longer preprocessed. Certain attributes that were
    previously only defined in certain compilation modes (notably
    LIBEV_EMBED) are now always defined, but will raise
    ``AttributeError`` or have a negative value when not available. In
    general these attributes are not portable or documented and are
    not implemented by libuv or the CFFI backend. See :issue:`1076`.

  - Certain private helper functions (``gevent_handle_error``, and part of
    ``gevent_call``) are now implemented in Cython instead of C. This
    reduces our reliance on internal undocumented implementation
    details of Cython and Python that could change. See :pr:`1080`.
