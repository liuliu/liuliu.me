---
date: '2018-08-15 23:42:00'
layout: post
slug: coroutine-to-coordinate-cuda-streams
status: publish
title: Coroutine to Coordinate CUDA Streams
categories:
- eyes
---

When programming with CUDA, there are several ways to exploit concurrency for CUDA kernel launches. As explained in [some](https://developer.download.nvidia.com/CUDA/training/StreamsAndConcurrencyWebinar.pdf) of [these slides](http://on-demand.gputechconf.com/gtc/2014/presentations/S4158-cuda-streams-best-practices-common-pitfalls.pdf), you can either:

 1. Create thread corresponding each execution flow, execute serially on stream per thread, coordinate with either `cudaEventSynchronize` or `cudaStreamSynchronize`;
 2. Carefully setup CUDA events and streams such that the correct execution flow will follow.

The 2. seems more appealing to untrained eyes (you don't have to deal with threads!) but in practice, often error-prune. One of the major issue, is that the `cudaEventRecord` / `cudaStreamWaitEvent` pair doesn't capture all synchronization needs. Comparing this to Grand Central Dispatch provided primitives: `dispatch_group_enter` / `dispatch_group_leave` / `dispatch_group_notify`, the under-specified part is where the `cudaEventEnter` happens. This often leads to a surprising fact that when you `cudaStreamWaitEvent` on a event not yet recorded on another stream (with `cudaEventRecord`), the current stream will treat as if this event is already happened and won't wait at all.

This is OK if your execution flows is static, thus, all the kernels need to be executed on which stream, are fully specified upfront. Requires some careful arrangement? Yes, but it is doable. However, it all breaks down if some coordinations need to happen after some kernel computations are done. For example, based on the newly computed losses, to determine whether decrease learn rate or not. Generally-speaking, for any computation graph that supports control structure, these coordinations are necessary.

The obvious way to solve this, is to go route 1. However, that imposes other problems, especially given pthread's handling of spawn / join is something much left to be desired.

For a few brave souls wanting to go route 2. to solve this, how?

After CUDA 5.x, a new method `cudaStreamAddCallback` is provided. This method itself carries some major flaws (before Kepler, `cudaStreamAddCallback` could cause unintended kernel launch serializations; the callback itself happens on the driver thread; and you cannot call any CUDA API inside that callback). But if we can gloss over some of these fundamental flaws and imagine, here is how I could make use of it with the imaginary `cudaEventEnter` / `cudaEventLeave` pair.

At the point I need to branch to determine whether to decrease learn rate, before `cudaStreamAddCallback`, I call `cudaEventEnter` to say that a event need to happen before certain stream to continue. Inside the callback, I get the loss from GPU, makes the decision, and call `cudaEventLeave` on the right event to continue the stream I want to branch into.

In real world, the above just cannot happen. We miss `cudaEventEnter` / `cudaEventLeave` primitives, and you cannot do any CUDA API call inside such callback. More over, the code will be complicated with these callbacks anyway (these are old-fashioned callbacks, not even lambda functions or dispatch blocks!).

What if, I can write code as if it is all synchronous, but under the hood, it all happens on one thread, so I don't have to worry about thread spawn / join when just scheduling work from CPU?

In the past a few days, I've been experimenting how to make [coroutines](https://en.wikipedia.org/wiki/Coroutine) work along `cudaStreamAddCallback`, and it seems all working! To make this actually useful in [NNC](https://libnnc.org) probably will take more time, but I just cannot wait to share this first :P

First, we need to have a functional coroutine implementation. There are [a lot](https://gist.github.com/lpereira/2154951) stackful [C coroutine implementations](https://swtch.com/libtask/) online and my implementation borrowed heavily from these sources. This particular coroutine implementation just uses `makecontext` / `swapcontext` / `getcontext`.

Setup basic data structures:
```c
union ptr_splitter {
	void *ptr;
	uint32_t part[2];
};

static const int default_stack_size = 65536;

typedef struct schd_s schd_t;
typedef struct task_s task_t;
typedef void (*task_fn_t)(task_t *task);

struct task_s {
	struct task_s* prev;
	struct task_s* next;
	schd_t* schd;
	int done;
	struct task_s* waitfor;
	// For swapcontext / makecontext / getcontext.
	ucontext_t context;
	char *stack;
	task_fn_t fn;
};

struct schd_s {
	task_t* head;
	task_t* tail;
	struct {
		int suspend;
	} count;
	pthread_cond_t cv;
	pthread_mutex_t mutex;
	ucontext_t caller, callee;
};
```

Setup a main run loop that can schedule coroutines:
```c
static void deltask(schd_t* const schd, task_t* const t)
{
	if (t->prev)
		t->prev->next = t->next;
	else
		schd->head = t->next;
	if (t->next)
		t->next->prev = t->prev;
	else
		schd->tail = t->prev;
}

static void* schdmain(void* userdata)
{
	schd_t* const schd = (schd_t*)userdata;
	for (;;) {
		pthread_mutex_lock(&schd->mutex);
		// No one is waiting, and no more tasks. exit.
		if (schd->head == 0 && schd->count.suspend == 0)
		{
			pthread_mutex_unlock(&schd->mutex);
			break;
		}
		if (schd->head == 0)
		{
			pthread_cond_wait(&schd->cv, &schd->mutex);
			pthread_mutex_unlock(&schd->mutex);
			continue;
		}
		task_t* const t = schd->head;
		deltask(schd, t);
		pthread_mutex_unlock(&schd->mutex);
		swapcontext(&schd->caller, &t->context);
		t->context = schd->callee;
		if (t->done)
			taskfree(t);
	}
	return 0;
}
```

Now, create a new task:
```c
static void _task_entry_point(uint32_t part0, uint32_t part1)
{
	union ptr_splitter p;
	p.part[0] = part0;
	p.part[1] = part1;
	task_t *task = (task_t*)p.ptr;
	task->fn(task);
	task->done = 1;
	swapcontext(&task->schd->callee, &task->schd->caller);
}

static task_t* taskcreate(schd_t* const schd, task_fn_t fn)
{
	task_t *task = (task_t*)calloc(1, sizeof(task_t));

	task->schd = schd;
	task->stack = (char*)calloc(1, default_stack_size);
	task->fn = fn;

	getcontext(&task->context);
	task->context.uc_stack.ss_sp = task->stack;
	task->context.uc_stack.ss_size = default_stack_size;
	task->context.uc_link = 0;

	union ptr_splitter p;
	p.ptr = task;
	makecontext(&task->context, (void (*)(void))_task_entry_point, 2, p.part[0], p.part[1]);
	return task;
}

static void addtask(schd_t* const schd, task_t* const t)
{
	if (schd->tail)
	{
		schd->tail->next = t;
		t->prev = schd->tail;
	} else {
		schd->head = t;
		t->prev = 0;
	}
	schd->tail = t;
	t->next = 0;
}

static void taskfree(task_t* const task)
{
	task_t* waitfor = task->waitfor;
	while (waitfor)
	{
		task_t* const next = waitfor->next;
		addtask(task->schd, waitfor);
		waitfor = next;
	}
	free(task->stack);
	free(task);
}
```

Usual utilities for coroutine (ability to yield, launch a new coroutine, and wait for existing coroutine to finish):
```c
static void taskyield(task_t* const task)
{
	addtask(task->schd, task);
	swapcontext(&task->schd->callee, &task->schd->caller);
}

static void taskresume(task_t* const task)
{
	ucontext_t old_context = task->schd->caller;
	swapcontext(&task->schd->caller, &task->context);
	task->context = task->schd->callee;
	task->schd->caller = old_context;
	if (task->done) // If the task is done here, we should just remove it.
		taskfree(task);
}

static void taskwait(task_t* const task, task_t* const waiton)
{
	task->prev = 0;
	task->next = waiton->waitfor;
	waiton->waitfor = task;
	swapcontext(&task->schd->callee, &task->schd->caller);
}
```

With above utilities, you can already experiment with coroutines:
```c
static void g(task_t* const task)
{
	printf("start task %p\n", task);
	taskyield(task);
	printf("back to task %p to finish\n", task);
}

static void f(task_t* const task)
{
	printf("create a new task to resume %p\n", task);
	task_t* gtask = taskcreate(task->schd, g);
	taskresume(gtask); // Run the gtask directly.
	printf("done task %p\n", task);
}

int main(void)
{
	schd_t schd = {};
	pthread_cond_init(&schd.cv, 0);
	pthread_mutex_init(&schd.mutex, 0);
	task_t* task = taskcreate(&schd, f);
	addtask(&schd, task);
	schdmain(&schd);
	pthread_cond_destroy(&schd.cv);
	pthread_mutex_destroy(&schd.mutex);
	return 0;
}
```

Unsurprisingly, you should be able to see print outs in order of:
```sh
create a new task to resume 0x288d010
start task 0x289d410
done task 0x288d010
back to task 0x289d410 to finish
```

coroutine f first executed, it launches coroutine g. When g gives up control (`taskyield`), coroutine f continues to execute until finish. After that, scheduler resumes coroutine g, and it finishes as well.

You can also try to `taskwait(task, gtask)` in coroutine f, to see that f will finish only after coroutine g is scheduled again until finish.

So far, we have a functional coroutine implementation in C. Some of these code doesn't seem to make sense, for example, why we need a mutex and a condition variable? Because a secret function that enables us to wait on a stream is not included above:
```c
static void taskcudaresume(cudaStream_t stream, cudaError_t status, void* userdata)
{
	task_t* const task = (task_t*)userdata;
	pthread_mutex_lock(&task->schd->mutex);
	addtask(task->schd, task);
	--task->schd->count.suspend;
	pthread_cond_signal(&task->schd->cv);
	pthread_mutex_unlock(&task->schd->mutex);
}

static void taskcudawait(task_t* const task, cudaStream_t stream)
{
	pthread_mutex_lock(&task->schd->mutex);
	++task->schd->count.suspend;
	cudaStreamAddCallback(stream, taskcudaresume, task, 0);
	pthread_mutex_unlock(&task->schd->mutex);
	// Compare to taskyield, this function doesn't do addtask(task->schd, task);
	swapcontext(&task->schd->callee, &task->schd->caller);
}
```

`taskcudawait` will put the current coroutine on-hold until the said stream finishes. Afterwards, you can do branch, and knowing comfortably kernels in the stream above are all done. The condition variable and the mutex is necessary because the callback happens on the driver thread.

You can see the full code that demonstrated the usage here: [https://gist.github.com/liuliu/7366373d0824a915a26ff295c468b6e4](https://gist.github.com/liuliu/7366373d0824a915a26ff295c468b6e4)

It seems above utilities would cover all my usages (the `taskwait` and `taskresume` are important to me because I don't want too much hard to control async-y when launch sub-coroutines). Will report back if some of these doesn't hold and I failed to implement fully-asynchronous, control structure supported computation graph with these cute little coroutines.