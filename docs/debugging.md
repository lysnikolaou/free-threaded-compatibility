# Debugging thread safety issues

Until now, the GIL has allowed developers to ignore C
safety issues when writing parallel programs, since the GIL ensured that
all thread execution was serialized, allowing for simultaneous access
to Python objects and state defined in the interpreter.

The new free-threaded model ensures that Python code access originating from
other Python code frames is safe and is guaranteed to not produce any major
interpreter crash, as opposed to unrestricted C code access, which can
present any of the common C thread-safety issues.

Usually, concurrency issues arise when two or more threads try to modify the
same value in memory. In Python, this commonly occurs when a class or function
defines native shared state, either via an attribute or a variable that can be
modified from native code in each thread execution scope.

The most common issues related to concurrency in the context of free-threaded
CPython extensions are either dirty reads/writes to global/shared C state,
unexpected behavior due to simultaneous access to C calls that are not
thread-safe, and finally, major runtime crashes due to memory allocation
issues and forbidden pointer lookups. While the first case depends on the
actual implementation of the algorithm/routine and may produce unintended
results, it would not cause a fatal crash of the interpreter, as opposed
to the last two cases.

In order to discover, handle and debug concurrency issues at large, there are
several strategies, which we will summarize next.

## pytest plugins to discover concurrency issues

As parallel testing has become a critical component to ensure compatibility
with free-threaded CPython, several community-led pytest plugins have been
implemented that attempt to smoke out issues by running all tests in a test
suite in a concurrent manner:

- [pytest-run-parallel](https://github.com/Quansight-Labs/pytest-run-parallel)
- [pytest-freethreaded](https://github.com/tonybaloney/pytest-freethreaded)

The advantage of using a pytest plugin as opposed to manually using the
`threading` and/or `concurrent.futures` modules mainly resides in their
ability to integrate with the ecosystem constructs like markers, fixtures,
skip and failure flags. For more information regarding the usage of these
libraries please refer to the documentation of each project.

### Repeated test execution

Given the non-deterministic nature of parallel execution, tests for code that
has a concurrency issue may still pass most of the time.
In order to more reliably reproduce a test failure under concurrency, we
recommend using [pytest-repeat](https://github.com/pytest-dev/pytest-repeat),
which enables the `--count` flag in the `pytest` command:

```bash
# Setting PYTHON_GIL=0 ensures that the GIL is effectively disabled.
PYTHON_GIL=0 pytest -x -v --count=100 test_concurrent.py
```

We advise to set `count` to `100` (or even larger if needed), in order to
ensure at least one concurrent clash event.

## Writing explicitly concurrent test cases

It may be desirable to have tests using, e.g., `threading` or
`concurrent.futures` in your test suite in order to prevent adding
additional test dependencies or to test a particular subset of tests for
concurrency issues by default. The stdlib `threading` module defines several
low-level parallel primitives that can be used to test for concurrency,
while the `concurrent.futures` module provides higher-level constructs.

For example, consider a method `MyClass.call_unsafe` that has been flagged as
having concurrency issues since it mutates attributes of a shared object that
is accessed by multiple threads. We can write a test for it using either
`threading` or `concurrent.futures` primitives:

<details>
<summary>Example using threading:</summary>

```python
import threading

# Library to test
from mylib import MyClass


def test_call_unsafe_concurrent_threading():
    # Defines a thread barrier that will be spawned before parallel execution
    # this increases the probability of concurrent access clashes.
    n_threads = 10
    barrier = threading.Barrier(n_threads)

    # This object will be shared by all the threads.
    cls_instance = MyClass(...)

    results = []

    def closure():
        # Ensure that all threads reach this point before concurrent execution.
        barrier.wait()
        r = cls_instance.call_unsafe()
        results.append(r)

    # Spawn n threads that call call_unsafe concurrently.
    workers = []
    for _ in range(0, n_threads):
        workers.append(threading.Thread(target=closure))

    for worker in workers:
        worker.start()

    for worker in workers:
        worker.join()

    # Do something about the results
    assert check_results(results)
```

</details>

<details>
<summary>Example using concurrent.futures:</summary>

```python
import threading
from concurrent.futures import ThreadPoolExecutor

# Library to test
from mylib import MyClass


def test_call_unsafe_concurrent_pool():
    # Defines a thread barrier that will be spawned before parallel execution
    # this increases the probability of concurrent access clashes.
    n_threads = 10
    barrier = threading.Barrier(n_threads)

    # This object will be shared by all the threads.
    cls_instance = MyClass(...)

    def closure():
        # Ensure that all threads reach this point before concurrent execution.
        barrier.wait()
        r = cls_instance.call_unsafe()
        return r

    with ThreadPoolExecutor(max_workers=n_threads) as executor:
        futures = [executor.submit(closure) for _ in range(n_threads)]

    results = [f.result() for f in futures]

    # Do something about the results
    assert check_results(results)
```

</details>

## Debugging tests that depend on native calls

If your code has native dependencies, either via C/C++ or Cython, `gdb`
(or `lldb`) can be used as follows:

```bash
# Setting PYTHON_GIL=0 ensures that the GIL is effectively disabled.
PYTHON_GIL=0 gdb --args python my_program.py --args ...

# To test under pytest
PYTHON_GIL=0 gdb --args python -m pytest -x -v "test_here.py::TestClass::test_method"

# Using LLDB (under LLVM/clang)
PYTHON_GIL=0 lldb -- $(which python) my_program.py

# Using LLDB (and pyenv)
PYTHON_GIL=0 lldb -- $(pyenv which python) $(pyenv which pytest) -x -v "test_here.py::TestClass::test_method"
```

When Python is run under `gdb`, several Python integration commands will be
available, such commands start with the `py-` prefix. For instance, the `py-bt`
allows to obtain a Python interpreter backtrace whenever the debugger hits a native
frame, this allows to improve the tracking of execution between Python and native
frames[^1].

For more information about `gdb` and `lldb` commands, we encourage reading
the [GDB to LLDB command map](https://lldb.llvm.org/use/map.html) page in the
official LLVM docs.

### Cython debugging

Since Cython produces intermediate C/C++ sources that then are compiled into native
code, stepping through may get difficult if done solely from the C source file.
In order to get through such difficulty, Cython includes the `cygdb` extension,
which enables `gdb` to go through large sections of C code that are equivalent to
a single Cython declaration.

Enabling `cygdb` requires the compilation of Cython sources with the `--gdb`
flag. After the sources are compiled and linked, it can be used as follows:

```bash
# For example, running the tests of scikit-image.
# build/cp313td/ contains the trace files generated by Cython to be compatible
# with cygdb
PYTHON_GIL=0 cygdb build/cp313td/ -- --args python -m  pytest -x -v skimage/
```

Since `cygdb` requires the Python interpreter version used to compile `gdb`
to match the one to be used during the execution of the script, recompiling `gdb`
will be necessary in order to ensure the most complete debugging experience.
We recommend the `gdb` [compilation instructions](https://www.linuxfromscratch.org/blfs/view/svn/general/gdb.html)
provided by the Linux from scratch project.

`cygdb` defines a set of commands prefixed by `cy` that replace the usual `gdb`
commands. For example `cy run` will start the program with the Cython debugging
extensions enabled, `cy break` will define a breakpoint on a function with the
Cython definition name, `cy next` will step over a Cython line, which is equivalent
to several lines in the produced C code.

### Detecting issues in CPython

If a debugging session suggests that an error/bug is incoming from CPython,
we recommend installing a debug instance. The easiest way to accomplish this
is via `pyenv`:

```bash
pyenv install --debug --keep 3.13.1
```

This command will not only install a debug distribution of CPython, but also
will ensure that the source files are kept as well, such files will be loaded
by `gdb`/`lldb` at the moment of debugging. For more information regarding
CPython installation sources, please visit the
[Installing a free-threaded Python](installing-cpython.md) page.

## Compiling CPython and foundational packages with ThreadSanitizer

[Thread sanitizer](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual) (or TSan) helps
to detect C/C++ data races in concurrent systems. This tool can help to reveal free-threading
related bugs in CPython and foundational packages (e.g. `numpy`).
In this section we provide the commands to build a free-threading compatible
CPython interpreter and packages with ThreadSanitizer and other hints to discover
potential data races.

### `cpython_sanity` docker images

To ease working with thread sanitizer in projects that use Python, NumPy, and
SciPy, we have create a set of docker images that contain a pre-built Python
interpreter and common dependencies that can be tricky to build.

See [the `cpython_sanity`
repository](https://github.com/nascheme/cpython_sanity) for more information
about how to use the docker images. Also see [NumPy PR #28808](https://github.com/numpy/numpy/pull/28808/files),
which adjusted NumPy TSAN CI to use the `ghcr.io/nascheme/numpy-tsan:3.14t-dev`
docker image instead of building Python from source, saving ten minutes of
compute time per CI run.

### Compile free-threaded CPython with ThreadSanitizer

- Clone the latest stable branch (`3.13`):

```bash
git clone https://github.com/python/cpython.git -b 3.13
```

- Configure and build the interpreter. Below instructions are for Linux
    (Windows and macOS may require some changes). We skip the instructions on how
    to install the Clang compiler.

```bash
cd cpython
CC=clang-18 CXX=clang++-18 ./configure --disable-gil --with-thread-sanitizer --prefix $PWD/cpython-tsan
make -j 8
make install
```

- To use the built Python interpreter:

```bash
# Create a virtual environment:
$PWD/cpython-tsan/bin/python3.13t -m venv ~/tsanvenv
# Then activate it:
source ~/tsanvenv/bin/activate

python -VV
# Python 3.13.1 experimental free-threading build (tags/v3.13.1:06714517797, Dec 19 2024, 10:06:54) [Clang 18.1.3 (1ubuntu1)]
PYTHON_GIL=0 python -c "import sys; print(sys._is_gil_enabled())"
# False

# Exit the `cpython` folder (preparation for the next step below)
cd ..
```

If you use pyenv, you can also enable a thread sanitizer build with `pyenv install` like so:

```bash
CC=/path/to/clang CXX=/path/to/clang++ CONFIGURE_OPTS="--with-thread-sanitizer" pyenv install 3.14t-dev
```

And then activate the build with e.g. `pyenv local 3.14t-dev`.

!!! note

    On MacOS, you may see messages like this when you start Python:

    ```
    python(7027,0x1f6dfc240) malloc: nano zone abandoned due to inability to reserve vm space.
    ```

    This message is being emitted by the MacOS malloc implementation. As
    [explained
    here](https://stackoverflow.com/questions/64126942/malloc-nano-zone-abandoned-due-to-inability-to-preallocate-reserved-vm-space),
    this happens for any program compiled with ThreadSanitizer on MacOS and can
    be safely ignored by setting the `MallocNanoZone` environment variable to
    0\. You should only set this in session you are running ThreadSanitizer
    under, as this setting will slow down other programs that allocate memory.

### Compile NumPy with ThreadSanitizer

- Get the source code (for example, the `main` branch)

```bash
git clone --recursive https://github.com/numpy/numpy.git
```

- Install the build requirements:

```bash
cd numpy
python -m pip install -r requirements/build_requirements.txt
# Make sure to install a compatible Cython version
# (--pre is needed to get a 3.1.0 beta release)
python -m pip install --pre cython
```

- Build the package

```bash
CC=clang-18 CXX=clang++-18 python -m pip install -v . --no-build-isolation -Csetup-args=-Db_sanitize=thread
# or with debug info
# CC=clang-18 CXX=clang++-18 python -m pip install -v . --no-build-isolation -Csetup-args=-Db_sanitize=thread -Csetup-args=-Dbuildtype=debugoptimized
```

## Running Python under ThreadSanitizer

### Useful ThreadSanitizer options

- By default ThreadSanitizer reports warnings. To stop execution on ThreadSanitizer errors, use:

```bash
TSAN_OPTIONS=halt_on_error=1 python -m pytest test.py
```

See [the ThreadSanitizer documentation](https://github.com/google/sanitizers/wiki/ThreadSanitizerFlags) for a full listing of options accepted by ThreadSanitizer.

- To add ThreadSanitizer suppressions (written in a file: `tsan-suppressions`):

```bash
# Let's show an example content of suppressions,
# more info: https://github.com/google/sanitizers/wiki/ThreadSanitizerSuppressions
cat $PWD/tsan-suppressions

race:llvm::RuntimeDyldELF::registerEHFrames
race:partial_vectorcall_fallback
race:dnnl_sgemm


export TSAN_OPTIONS="suppressions=$PWD/tsan-suppressions" python -m pytest test.py
```

### Running pytest tests under ThreadSanitizer

By default, pytest [captures all output from
tests](https://docs.pytest.org/en/stable/how-to/capture-stdout-stderr.html),
this means that you might only see output like `ThreadSanitizer: reported 2 warnings`, but with no accompanying report with details about the warning.

To ensure that pytest doesn't capture any output from ThreadSanitizer, you can
pass `-s` (short for `--show-capture`) to your pytest invocation.

Some authors of this guide have observed hangs running pytest with
`halt_on_error=1`. If you observe hangs, try setting `halt_on_error=0` in
TSAN_OPTIONS.

The [pytest-xdist](https://github.com/pytest-dev/pytest-xdist) plugin can also
sometimes be problematic if a test runner happens to crash during
execution. While `pytest-xdist` does have some support for detecting crashed
worker, it is not foolproof and the authors of this guide have observed hangs on
CI due to pytest-xdist not properly handling a worker failing due to a ThreadSanitizer
error.

The `pytest-xdist` plugin also [makes it impossible to obtain stdout from
a test runner](https://github.com/pytest-dev/pytest-xdist/issues/82), so there
is no way to see ThreadSanitizer output if there is an issue. This can lead to hangs on CI
machines with no accompanying error report to explain the nature of the
hang. For that reason we suggest uninstalling `pytest-xdist` from your
environment to ensure it isn't used. If you need to use `pytest-xdist` to make
the tests complete in a reasonable amount of time, we suggest using
[`pytest-timeout`](https://pypi.org/project/pytest-timeout/) to ensure hung
tests eventually exit, particularly on CI.

ThreadSanitizer includes a check to ensure allocators never fail. This can lead to runtime
crashes if a test happens to try allocating a very large block of memory
specifically to ensure such an allocation does fail correctly. Set
`allocator_may_return_null=1` in `TSAN_OPTIONS` to avoid this.

If a ThreadSanitizer warning is detected, the exit code of the running process will be set
to a nonzero value (66, by default). If for some reason that is problematic in
your test suite then you can set `exitcode=0` in `TSAN_OPTIONS` to make ThreadSanitizer
quit "successfully" if a warning is detected. For example, you might set this if
a subprocess returning a nonzero exit code unexpectedly breaks a test.

You might also find that running your test suite is very slow under
ThreadSanitizer. Consider skipping tests that do not use threads, for example by only
testing files that import `threading` or
`concurrent.futures.ThreadPoolExecutor`. See [this NumPy CI
workflow](https://github.com/numpy/numpy/blob/da268d45aab791023c8d953db6f4597019f770cb/.github/workflows/compiler_sanitizers.yml#L128)
that runs pytest on a subset of NumPy's tests. This will miss tests that spawn
threads in native code (e.g. with OpenMP or other threading primitives) or use
Python packages that spawn threads, but is a good option if your library doesn't
do that.

Altogether, a pytest invocation using ThreadSanitizer might look like:

```
$ TSAN_OPTIONS='allocator_may_return_null=1 halt_on_error=1' pytest -s
```

### Using AddressSanitizer to detect thread safety issues

Since ThreadSanitizer adds significant overhead to both the Python interpreter
and any native code compiled under ThreadSanitizer, many projects will be unable
to run their full test suite under ThreadSanitizer in CI.

For that reason, we also suggest setting up CI run for your full test suite
using [AddressSanitizer](https://github.com/google/sanitizers/wiki/addresssanitizer) (or
ASan). AddressSanitizer will detect if there are any memory safety issues
triggered by multithreaded tests. While it will not detect data races that do
not lead to observable memory safety issues, it will detect races that could
lead to e.g. a segmentation fault and give precise debugging information about
the nature of the memory safety issue. A developer could then look more closely
at the issue using ThreadSanitizer outside of CI to more fully understand whether
data races contributed to the memory safety issue.

You can build Python with AddressSanitizer by passing `--with-address-sanitizer`
to the CPython configure script. You can build NumPy with AddressSanitizer by passing
`-Csetup-args=-Db_sanitize=address` as an argument to `pip install`.

Like ThreadSanitizer, AddressSanitizer also accepts a number of options to
control its behavior. See [the
documentation](https://github.com/google/sanitizers/wiki/addresssanitizerflags)
for more details. Note that both the CPython interpreter and many extensions
have harmless memory leaks, so consider disabling the [leak
sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizerLeakSanitizer)
built into AddressSanitizer by setting `ASAN_OPTIONS="detect_leaks=0"`.

[^1]: This feature is not correctly working on `lldb` after CPython 3.12.
