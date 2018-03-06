---
layout: post
title: 'Mutexes And fork()ing In Shared Libraries'
---

![alt text](https://linuxcontainers.org/static/img/containers.png)

#### Disclaimer
In this short - let's call it "semi-informative rant" - I'm going to be looking
at mutexes and `fork()` in shared libraries with threaded users. I'm going to
leave out other locking primitives including semaphores and file locks which
would deserve posts of their own.

#### The Stuff You Came Here For

A mutex is simply put one of the many synchronization primitives to protect
a range of code usually referred to as "critical section" from concurrent
operations. Reasons for using them are many. Examples include:
- avoiding data corruption through multiple writers changing the same data
  structure at the same time
- preventing readers from retrieving inconsistent data because a writer is
  changing the data structure at the same time
- .
- .
- .

In its essence it is actually a pretty easy concept once you think about it.
You want ownership of a resource, you want that ownership to be exclusive, you
want that ownership to be limited from `t_1` to `t_n` where you yield it. In
the language of `C` and the `pthread` implementation this can be expressed in
code e.g. as:
```C
static pthread_mutex_t thread_mutex = PTHREAD_MUTEX_INITIALIZER;

static int some_function(/* parameters of relevance*/)
{
        int ret;

        ret = pthread_mutex_lock(&thread_mutex);
        if (ret != 0) {
                /* handle error */
                _exit(EXIT_FAILURE);
        }

        /* critical section */

        ret = pthread_mutex_unlock(&thread_mutex);
        if (ret != 0) {
                /* handle error */
                _exit(EXIT_FAILURE);
        }
}
```
Using concepts like [mutexes](http://pubs.opengroup.org/onlinepubs/9699919799/)
in a shared library is always a tricky thing. What I mean by that is that if
you can avoid them avoid them. For a start, mutexes usually come with
a performance impact. The size of the impact varies with a couple of different
parameters e.g. how long the critical section is. Depending on what you are
doing these performance impacts might or might not matter to you or not even
register as significant. So the performance impact argument is a difficult one
to make. Usually programmers with a decent understanding of locking can find
ways to minimize the impact of mutexes by toying with the layout and structure
of critical sections ranging from choosing the right data structures to simply
moving code out of critical sections.

There are better arguments to be made against casually using mutexes though.
One is closely coupled to what type of program you're writing. If you're like
me coming from the background of a low-level `C` shared library like
[`LXC`](https://github.com/lxc/lxc) you will at some point find yourself
thinking about the question whether there's any possibility that you might be
used in threaded contexts. If you can confidently answer this question with "no"
you can **likely** stop caring and move on. If you can't then you should think
really really hard in order to avoid mutexes. The problem is a classical one
and I'm not going to do a deep dive as this has been done before all over the
web. What I'm alluding to is of course the mess that is `fork()`ing in threads.
Most shared libraries that do anything interesting will likely want to `fork()`
of helper tasks in API functions. In threaded contexts this quickly becomes
a source of undefined behavior. The way `fork()`ing in threads works is that
only the thread that called `fork()` gets duplicated in the child, the others
are terminated. Given that `fork()` duplicates memory state, locking etc. all
of which is shared amongst threads you quickly run into deadlocks whereby
mutexes that were held in other threads are never unlocked. But it can also
cause nasty undefined behavior whereby file pointers via e.g. `fopen()` although
set up so as to be unique to each thread get corrupted due to inconsistent
locking caused by e.g. dynamically allocating memory via `malloc()` or friends
in the child process because behind the scenes in a lot of libcs mutexes are
used when allocating memory.

The possibilities for bugs are endless. Another good example is the use of the
`exit()` function to terminate child processes. The `exit()` function is not
thread-safe since it runs standard and user-registered exit handlers from
a shared resource. This is a common source of process corruption. The lesson
here is of course to always use `_exit()` instead of `exit()`. The former is
thread-safe and doesn't run exit handlers. But that presupposes that you don't
care about exit handlers.

A lot of these bugs are hard to understand, debug, and - to be honest - even to
explain given that they are a mixture of undefined behavior and legal
thread and `fork()` semantics.

#### Running Handlers At `fork()`

Of course, these problems were realized early on and one way to address those
is to register handlers that would be called at each `fork()`. In the `pthread`
slang the name of the function to register such handlers is appropriately
"`pthread_atfork()`". In the case of mutexes this means you would register
three handlers that would be called at different times at `fork()`. One right
before the `fork()` - *prepare handler* - to e.g. unlock any implicitly held
mutexes. One to be called after `fork()` processing has finished in the child -
*child handler* - and one called after `fork()` processing in the parent
finishes - *parent handler*. In the `pthread` implementation and for a shared
library this would likely look something like this:
```C
void process_lock(void)
{
        int ret;

	ret = pthread_mutex_lock(&thread_mutex);
        if (ret != 0)
                _exit(EXIT_FAILURE);
}

void process_unlock(void)
{
        int ret;

	ret = pthread_mutex_unlock(&thread_mutex);
        if (ret != 0)
                _exit(EXIT_FAILURE);
}

#ifdef HAVE_PTHREAD_ATFORK
__attribute__((constructor)) static void __register_atfork_handlers(void)
{
        /* Acquire lock right before fork() processing to avoid undefined
         * behavior by unlocking an unlocked mutex. Then release mutex in child
         * and parent.
         */
        pthread_atfork(process_lock, process_unlock, process_unlock);
}
#endif
```
While this sounds like a reasonable approach it has various and serious
drawbacks:
1. These atfork handlers come with a cost that - again depending on your
   program - you maybe would like to avoid.
2. They don't allow you to explicitly hold a lock when `fork()`ing in the same
   task depending on what handlers you are registering.

   This is straightforward. Let's reason about the following code sequence for
   a minute ignoring whether holding the mutex would make sense that way:
    ```C
            int ret, status;
            pid_t pid;

            process_lock();
            pid = fork();
            if (pid < 0)
                    return -1;

            if (pid == 0) {
                    /* critical section */
                    process_unlock();
                    _exit(EXIT_SUCCESS);
            }
            process_unlock();

    again:
            ret = waitpid(pid, &status, 0);
            if (ret < 0) {
                    if (errno == EINTR)
                            goto again;

                    return -1;
            }

            if (ret != pid)
                    goto again;

            if (!WIFEXITED(status) || WEXITSTATUS(status) != 0)
                    return -1;

            return 0;
    ```
   No let's add the logic caused by `pthread_atfork()` in there (The mutex
   annotation is slightly misleading but should make things a little easier to
   follow):
    ```C
            int ret, status;
            pid_t pid;

            process_lock(); /* <mutex 1> (explicitly acquired) */
            process_lock(); /* <mutex 2> (implicitly acquired by prepare atfork handler) */
            pid = fork();
            if (pid < 0)
                    return -1;

            if (pid == 0) {
                    /* <mutex 1> held (transparently held) */
                    /* <mutex 2> held (opaquely held) */

                    /* critical section */

                    process_unlock(); /* <mutex 1> (explicitly released) */
                    process_unlock(); /* <mutex 2> (implicitly released by child atfork handler) */
                    _exit(EXIT_SUCCESS);
            }
            process_unlock(); /* mutex_2 (implicitly released by parent atfork handler) */
            process_unlock(); /* mutex_1 (explicitly released) */

    again:
            ret = waitpid(pid, &status, 0);
            if (ret < 0) {
                    if (errno == EINTR)
                            goto again;

                    return -1;
            }

            if (ret != pid)
                    goto again;

            if (!WIFEXITED(status) || WEXITSTATUS(status) != 0)
                    return -1;

            return 0;
    ```
   That doesn't look crazy at a first glance. But let's explicitly look at the
   problem:
    ```C
    int ret, status;
    pid_t pid;

    process_lock(); /* <mutex 1> (explicitly acquired) */
    process_lock(); /* **DEADLOCK** <mutex 2> (implicitly acquired by prepare atfork handler) */
    pid = fork();
    if (pid < 0)
            return -1;
    ```
3. They aren't run when you use `clone()` (which obviously is a big deal for a
   container API like [`LXC`](https://github.com/lxc/lxc)). So scenarios like
   the following are worrying:
    ```C
    /* premise: some other thread holds a mutex */
    pid_t pid;
    void *stack = alloca(/* standard page size */);

    /* Here atfork prepare handler needs to be run but won't. */
    pid = clone(foo, stack + /* standard page size */, SIGCHLD, NULL);
    if (pid < 0)
            return -1;
    ```

   The point about `clone()` is interestingly annoying. Since `clone()` is
   Linux specific there's no `POSIX` standard that gives you a guarantee that
   atfork handlers are run or that they are not run. That's up to the
   implementation (read "libc in question").
   Currently `glibc` doesn't run atfork handlers but if my fellow maintainers
   and I where to build consensus that it would be a great idea to change it in
   the next release then we would be free to do so (Don't worry, we won't.). So
   to make sure that no atfork handlers are run you need to go directly through
   the `syscall()` helper that all libcs should provide. This should give you
   a strong enough guarantee. That is of course an excellent solution if you
   don't care about atfork handlers. However, when you do care about them you
   better not use `clone()`.
4. Running a subset of already registered atfork handlers is a royal pain.

   This relates back to the earlier point about e.g. wanting to explicitly hold
   a lock in a task while `fork()`ing. In this case you might want to
   exclude the handler right before the `fork()` that locks the mutex. If you
   need to do this then you're going to have to venture into the dark dark land
   of function interposition. Something which is really ugly. It's like asking
   how to make
   [Horcruxes](https://en.wikipedia.org/wiki/Magical_objects_in_Harry_Potter#Horcruxes)
   or - excuse the pun - **`fork()`cruxes**. Sure, you'll eventually trick some
   low-level person into explaining it to you because it's just such a weird
   and exotic thing to know or care about but that explanation will ultimately
   end with phrases such as "That's all theoretical, right?" or "You're not
   going to do this, right?" or - the most helpful one (honestly) - "The
   probability that something's wrong with your programm's design is higher
   than the probability that you really need interposition wrappers.".
   In this specific case interposing `pthread_atfork()` would probably involve
   using `pthread_once()` calling `dlsym(RTLD_NEXT, "pthread_atfork")` and
   recording the function pointer in a global variable. Additionally, you
   likely want to start maintaining a jump table (essentially an array of
   function pointers) and register a callback wrapper around the jump table
   entries. You can then go on to call the callback in `pthread_atfork()` with
   different indices into the jump table. If you're super ambitious (read
   "insane") you could then have a different set of callbacks for each `fork()`
   in your program. Also, I just told you how to make a `fork()`crux. Let me
   tell you while I did this for "fun" once there's a limit to how dirty you
   can feel without hating yourself. Also, this is all theoretical, right?

The list could go on and be even more detailed but the gist is: if there's
a chance that your shared library is called in threaded contexts try to come up
with a design that lets you avoid mutexes and atfork handlers.
On the road to `LXC 3.0` we've recently managed to kick out all mutexes and
atfork handlers of which there were very few already. This has greatly improved
our confidence in threaded use cases. This is especially important since we
have API consumers that call `LXC` from inherently threaded contexts such as
the `Go` runtime. [`LXD`](https://github.com/lxc/lxd)
obviously is the prime example but also the general
[`go-lxc`](https://github.com/lxc/go-lxc) bindings are threaded API consumers.
To be fair, we've never had issues before as mutexes were extremely rare in the
first place but one should always remember that no locking is the best locking. :)

#### Addendum

##### 2018-03-06

Coming back once more to the point about running atfork handlers. Atfork
handlers are of course an implementation detail in the `pthread` and `POSIX`
world. They are by no means a conceptual necessity when it comes to mutexes. But
some standard is better than no standard when it comes to system's design. Any
decent libc implementation supporting `pthread` will very likely also support
atfork handlers (even `Bionic` has gained atfork support along the way). But
this immediately raises another problem as it requires programming languages on
`POSIX` systems to go through the system's libc when doing a `fork()`. If they
don't then atfork handlers won't be run even if you call `fork()`. One prime
example is `Go`. The `syscall` and `sys/unix` packages will **not** go through
the system's libc. They will directly do the corresponding syscall. So atfork
handlers are not available when `fork()`ing in `Go`. Now, `Go` is a little
special as it doesn't support `fork()` properly in the first place because of
all the reasons (and more) I outlined above.

Christian
