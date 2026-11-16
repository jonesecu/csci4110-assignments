# Assignment 4 — See PDF file for assigmment information

SDSC Migration Notes:

> You'll run this on **SDSC's cluster (x86-64, AMD EPYC)** — **not** Bridges2.
> The code and commands below are written for the reference environment. Three
> things change on this machine. Read them before you start or you'll waste
> hours chasing ghosts.

---

## 1. The `-m128bit-long-double` flag does NOT exist here — use `double`

On Bridges2 the machine was 128-bit `long double`. **This machine is x86-64,
where `long double` is only 80-bit extended precision.** The flag

```bash
g++ pi.cpp -o pi -m128bit-long-double   # <-- fails here
```

produces:

```
error: unrecognized command-line option '-m128bit-long-double'
```

There is no 128-bit `long double` on x86-64 — no flag will create one.

**The fix:** switch the accumulator from `long double` to `double`. This is not
a shortcut — it makes the error metric *honest* (see #2). Replace:

```cpp
long double PI = 3, n = 2, sign = 1;
long double cPI = 0.0;
```

with:

```cpp
double PI = 3.0, n = 2.0, sign = 1.0;
double cPI = 0.0;
```

And `calcPI`'s signature:

```cpp
double calcPI(double PI, double n, double sign, double iterations)
```

Everything else in the Nilkantha term is unaffected. This compiles cleanly and
gives you a real, meaningful error number.

---

## 2. Your error number is capped at ~19 digits — stop printing 50

The point of this assignment is comparing **16 error rates** across processors.
The whole goal is to prove your parallel reduction is numerically sound.

- **128-bit `long double` (Bridges2):** ~34 significant digits → `%.50Lf` is meaningful.
- **80-bit `long double` (this machine):** ~19 significant digits.

So `%.50Lf` prints ~31 digits of **garbage** past the precision wall. Worse:
once you're at the 19-digit limit, a *correct* reduction and a *broken* one both
report the same tiny error. You can no longer tell them apart by the number.

**Use `double` and print 17 digits** (the max meaningful precision):

```cpp
printf("PI is approx %.17g, Error is %.17g\n", cPI, fabs(cPI - PI25DT));
```

Do **not** print more than ~17 digits. More than that is noise, and it will make
you think your results are wrong when they aren't.

---

## 3. Bind your processes — NUMA affinity will ruin your scaling curve

This machine is a **2-socket AMD EPYC** (128 cores split into two socket
clusters). The reference code has **no affinity handling at all**. On a
single-socket ARM node that was invisible. Here it's the difference between a
clean scaling curve and a broken one:

- Unbound MPI processes get scheduled across sockets.
- Cross-socket traffic destroys performance.
- Your **one-sided (pi2/pi3)** runs can get *slower* with more processes purely
  from NUMA misses — **not** from the communication pattern being wrong.

You will conclude your implementation is buggy when it's actually topology.

**Bind explicitly.** In your `srun` command, add:

```bash
srun --cpu_bind=cores --ntasks-per-node=NN ./pi 800000
```

or, to pin to one socket for a clean baseline:

```bash
srun --cpu_bind=cores --ntasks-per-node=64 ./pi 800000
```

Document which binding you used in your write-up. It's a variable, not an
implementation detail.

---

## 4. Expect your scaling to collapse at high process counts — that's the point

At 48 processes, each rank does ~0.3ms of work. MPI handshake and one-sided
accumulation overhead **dominate** — you will see the speedup curve plateau or
even go *negative*. That is **the lesson**, not a bug. The assignment is about
*when communication cost eats your compute savings*, not *if it does*.

If your pi2/pi3/4 curves flatten at 24/48 procs, write that up and explain
*why* — that's a full-credit observation, not a failure.

---

## 5. pi4 and GPU nodes — stay CPU-only

The assignment is **pure CPU MPI**. No GPU, no CUDA, no OpenACC is involved.
The assignment's GPU nodes are **fixed-size** (N cores + K GPUs), so:

- 1/12/24/48 processes may **not divide evenly** into a node's geometry.
- You'll be in a **GPU partition** with different walltime and a credit-based
  account — a different billing model than your Bridges allocation.
- The OpenMPI build there may be MPI-over-UVM or SHMEM-tuned, which changes
  one-sided RMA performance — exactly what pi2/pi3 are measuring.

**Keep pi4 CPU-only** (OpenMP, Pthreads, etc.). It's the cleanest way to avoid
all of the above. If you want to involve a GPU, that's a different assignment
and needs a fresh spec — this handout doesn't support it.

---

## Quick reference — compile & run

```bash
# Compile (works here)
g++ pi.cpp -o pi -O2

# Run (12 tasks on one socket, bound)
srun --cpu_bind=cores --ntasks-per-node=12 ./pi 800000

# Run (48 tasks, full node)
srun --cpu_bind=cores --ntasks-per-node=48 ./pi 800000
```

Run each of pi1/pi2/pi3/pi4 across **1, 12, 24, 48** tasks → **16 timings + 16
error rates**. Log them all.

---

*This note supersedes the reference environment's commands. If something from
the Bridges2 handout doesn't work here, it's one of the items above.*
