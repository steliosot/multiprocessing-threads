# Parallel Programming in Python

This tutorial walks you through **serial vs parallel execution** in Python using:

- `multiprocessing.Process`
- `concurrent.futures.ProcessPoolExecutor`
- `multiprocessing.Pool`

It is based on the provided examples (`tutorial1.py` … `tutorial5.py`) and connects directly to the exercises.

---

## 0) Before you start

### What you’ll learn

By the end, you will be able to:

1. Explain **why serial code is slower** for independent tasks.
2. Run work **in parallel** using multiple CPU processes.
3. Measure runtime using `time.perf_counter()`.
4. Choose between `Process`, `Pool`, and `ProcessPoolExecutor`.

### What you need

- Python **3.9+**

- A terminal

- (Exercise 2 only) `requests` package:

  ```bash
  pip install requests
  ```

---

## 1) Serial execution: the baseline idea

**Serial execution** means: do task A, then B, then C… one after another.

### Example: a task that takes time

In the tutorials, the task is a function that sleeps for 1 second (simulates work).

```python
import time

def foo():
    print("Running...")
    time.sleep(1)
```

### Serial runner (runs 3 times in a row)

```python
import time

def serial_runner():
    start = time.perf_counter()
    for _ in range(3):
        foo()
    end = time.perf_counter()
    print(f"Serial: {end - start:.2f} second(s)")
```

✅ Expected behaviour  
If `foo()` takes ~1 second, then 3 runs take ~3 seconds.

---

## 2) Parallel execution with `multiprocessing.Process`

When tasks are independent, you can run them **at the same time** using **processes**.

### Why processes?

Python threads are limited by the **GIL** for CPU-heavy tasks.  
Processes avoid this by running in separate Python interpreters.

### Parallel runner (manual processes)

```python
import time
import multiprocessing

def foo():
    print("Running...")
    time.sleep(1)

def parallel_runner():
    start = time.perf_counter()

    p1 = multiprocessing.Process(target=foo)
    p2 = multiprocessing.Process(target=foo)
    p3 = multiprocessing.Process(target=foo)

    p1.start(); p2.start(); p3.start()
    p1.join();  p2.join();  p3.join()

    end = time.perf_counter()
    print(f"Parallel: {end - start:.2f} second(s)")
```

✅ Expected behaviour  
All 3 processes run at once, so total time is close to **~1 second** (plus overhead).

---

## 3) Parallel execution with a loop (cleaner pattern)

Instead of writing p1/p2/p3 manually, use a list.

```python
import time
import multiprocessing

def foo():
    time.sleep(1)

def parallel_runner_loop():
    start = time.perf_counter()
    processes = []

    for _ in range(3):
        p = multiprocessing.Process(target=foo)
        p.start()
        processes.append(p)

    for p in processes:
        p.join()

    end = time.perf_counter()
    print(f"Parallel (loop): {end - start:.2f} second(s)")
```

**Explanation**

- `.start()` begins the process.
- `.join()` waits for it to finish.
- This pattern scales easily (3 → 30 tasks).

---

## 4) Running tasks with different inputs

Often you want to run the same function with different parameters.

### Example: `boo(sec)` that sleeps for `sec` seconds

```python
import time

def boo(sec):
    print(f"Running for {sec} second(s)")
    time.sleep(sec)
```

### Serial version

```python
import time

def serial_runner(secs):
    start = time.perf_counter()
    for s in secs:
        boo(s)
    end = time.perf_counter()
    print(f"Serial: {end - start:.2f} second(s)")
```

### Parallel version using a process pool executor

```python
import time
import concurrent.futures

def parallel_runner_executor(secs):
    start = time.perf_counter()
    with concurrent.futures.ProcessPoolExecutor() as executor:
        executor.map(boo, secs)
    end = time.perf_counter()
    print(f"Parallel (executor.map): {end - start:.2f} second(s)")
```

**Explanation**

- `ProcessPoolExecutor()` manages a pool of worker processes for you.
- `executor.map(func, inputs)` applies the function to each input in parallel.

✅ Expected behaviour  
For `secs = [1,2,3]`, serial ≈ 6 seconds, parallel ≈ ~3 seconds (dominated by the longest task).

---

## 5) CPU-bound example: Sum of squares

Sleeping is “fake work”. Now we do CPU work.

### Task function: sum of squares

```python
def sum_square(number):
    s = 0
    for i in range(number):
        s += i * i
    return s
```

### Serial runner

```python
import time

def serial_runner(arange):
    start = time.perf_counter()
    for i in range(arange):
        sum_square(i)
    end = time.perf_counter()
    print(f"Serial: {end - start:.2f} second(s)")
```

### Parallel runner using `ProcessPoolExecutor`

```python
import time
import concurrent.futures

def parallel_runner_executor(arange):
    start = time.perf_counter()
    with concurrent.futures.ProcessPoolExecutor() as executor:
        list(executor.map(sum_square, range(arange)))
    end = time.perf_counter()
    print(f"Parallel (executor.map): {end - start:.2f} second(s)")
```

> Why `list(...)`?
>
> `executor.map(...)` is lazy; converting to a list forces completion so timing is correct.

---

## 6) Controlling number of workers

Sometimes you want a fixed number of processes (e.g., 2 or 4) for fair comparison.

```python
import time
import concurrent.futures

def parallel_runner_fixed_workers(arange, workers=2):
    start = time.perf_counter()
    with concurrent.futures.ProcessPoolExecutor(max_workers=workers) as executor:
        list(executor.map(sum_square, range(arange)))
    end = time.perf_counter()
    print(f"Parallel ({workers} workers): {end - start:.2f} second(s)")
```

**Guidance**

- Try `workers=1` to see overhead without real concurrency.
- Try `workers=2`, then `workers=4`.

---

## 7) `multiprocessing.Pool` vs `ProcessPoolExecutor`

You also saw `multiprocessing.Pool()` in Tutorial 5.

### Parallel map with `multiprocessing.Pool`

```python
import time
import multiprocessing

def parallel_map_pool(arange):
    start = time.perf_counter()
    with multiprocessing.Pool() as p:
        p.map(sum_square, range(arange))
    end = time.perf_counter()
    print(f"Parallel (multiprocessing.Pool.map): {end - start:.2f} second(s)")
```

### What’s the difference (simple)

- `multiprocessing.Pool()`:
  - Great for straightforward map-style parallelism
  - Classic API
- `ProcessPoolExecutor`:
  - Modern API (`concurrent.futures`)
  - Easier error handling and integration with futures
  - Generally preferred for new code

---

## 8) Common mistakes (and how to avoid them)

### ✅ Always protect process code on macOS/Windows

If you forget this, your program may spawn processes forever.

```python
if __name__ == "__main__":
    serial_runner(20000)
    parallel_runner_executor(20000)
```

### ✅ Don’t parallelise tiny tasks

For very small tasks, overhead can be bigger than the speedup.

### ✅ Use processes for CPU-bound work

- CPU-bound: sorting, math, hashing → **processes**
- IO-bound: network, file downloads → processes can help too, but threads/async are also options

