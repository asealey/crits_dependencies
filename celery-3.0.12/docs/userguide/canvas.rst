.. _guide-canvas:

=============================
 Canvas: Designing Workflows
=============================

.. contents::
    :local:
    :depth: 2

.. _canvas-subtasks:

Subtasks
========

.. versionadded:: 2.0

You just learned how to call a task using the tasks ``delay`` method
in the :ref:`calling <guide-calling>` guide, and this is often all you need,
but sometimes you may want to pass the signature of a task invocation to
another process or as an argument to another function, for this Celery uses
something called *subtasks*.

A :func:`~celery.subtask` wraps the arguments, keyword arguments, and execution options
of a single task invocation in a way such that it can be passed to functions
or even serialized and sent across the wire.

- You can create a subtask for the ``add`` task using its name like this::

        >>> from celery import subtask
        >>> subtask('tasks.add', args=(2, 2), countdown=10)
        tasks.add(2, 2)

  This subtask has a signature of arity 2 (two arguments): ``(2, 2)``,
  and sets the countdown execution option to 10.

- or you can create one using the task's ``subtask`` method::

        >>> add.subtask((2, 2), countdown=10)
        tasks.add(2, 2)

- There is also a shortcut using star arguments::

        >>> add.s(2, 2)
        tasks.add(2, 2)

- Keyword arguments are also supported::

        >>> add.s(2, 2, debug=True)
        tasks.add(2, 2, debug=True)

- From any subtask instance you can inspect the different fields::

        >>> s = add.subtask((2, 2), {'debug': True}, countdown=10)
        >>> s.args
        (2, 2)
        >>> s.kwargs
        {'debug': True}
        >>> s.options
        {'countdown': 10}

- It supports the "Calling API" which means it takes the same arguments
  as the :meth:`~@Task.apply_async` method::

    >>> add.apply_async(args, kwargs, **options)
    >>> add.subtask(args, kwargs, **options).apply_async()

    >>> add.apply_async((2, 2), countdown=1)
    >>> add.subtask((2, 2), countdown=1).apply_async()

- You can't define options with :meth:`~@Task.s`, but a chaining
  ``set`` call takes care of that::

    >>> add.s(2, 2).set(countdown=1)
    proj.tasks.add(2, 2)

Partials
--------

A subtask can be applied too::

    >>> add.s(2, 2).delay()
    >>> add.s(2, 2).apply_async(countdown=1)

Specifying additional args, kwargs or options to ``apply_async``/``delay``
creates partials:

- Any arguments added will be prepended to the args in the signature::

    >>> partial = add.s(2)          # incomplete signature
    >>> partial.delay(4)            # 2 + 4
    >>> partial.apply_async((4, ))  # same

- Any keyword arguments added will be merged with the kwargs in the signature,
  with the new keyword arguments taking precedence::

    >>> s = add.s(2, 2)
    >>> s.delay(debug=True)                    # -> add(2, 2, debug=True)
    >>> s.apply_async(kwargs={'debug': True})  # same

- Any options added will be merged with the options in the signature,
  with the new options taking precedence::

    >>> s = add.subtask((2, 2), countdown=10)
    >>> s.apply_async(countdown=1)  # countdown is now 1

You can also clone subtasks to augment these::

    >>> s = add.s(2)
    proj.tasks.add(2)

    >>> s.clone(args=(4, ), kwargs={'debug': True})
    proj.tasks.add(2, 4, debug=True)

Immutability
------------

.. versionadded:: 3.0

Partials are meant to be used with callbacks, any tasks linked or chord
callbacks will be applied with the result of the parent task.
Sometimes you want to specify a callback that does not take
additional arguments, and in that case you can set the subtask
to be immutable::

    >>> add.apply_async((2, 2), link=reset_buffers.subtask(immutable=True))

The ``.si()`` shortcut can also be used to create immutable subtasks::

    >>> add.apply_async((2, 2), link=reset_buffers.si())

Only the execution options can be set when a subtask is immutable,
so it's not possible to call the subtask with partial args/kwargs.

.. note::

    In this tutorial I sometimes use the prefix operator `~` to subtasks.
    You probably shouldn't use it in your production code, but it's a handy shortcut
    when experimenting in the Python shell::

        >>> ~subtask

        >>> # is the same as
        >>> subtask.delay().get()


.. _canvas-callbacks:

Callbacks
---------

.. versionadded:: 3.0

Callbacks can be added to any task using the ``link`` argument
to ``apply_async``::

    add.apply_async((2, 2), link=other_task.subtask())

The callback will only be applied if the task exited successfully,
and it will be applied with the return value of the parent task as argument.

As I mentioned earlier, any arguments you add to `subtask`,
will be prepended to the arguments specified by the subtask itself!

If you have the subtask::

    >>> add.subtask(args=(10, ))

`subtask.delay(result)` becomes::

    >>> add.apply_async(args=(result, 10))

...

Now let's call our ``add`` task with a callback using partial
arguments::

    >>> add.apply_async((2, 2), link=add.subtask((8, )))

As expected this will first launch one task calculating :math:`2 + 2`, then
another task calculating :math:`4 + 8`.

The Primitives
==============

.. versionadded:: 3.0

.. topic:: Overview

    - ``group``

        The group primitive is a subtask that takes a list of tasks that should
        be applied in parallel.

    - ``chain``

        The chain primitive lets us link together subtasks so that one is called
        after the other, essentially forming a *chain* of callbacks.

    - ``chord``

        A chord is just like a group but with a callback.  A chord consists
        of a header group and a body,  where the body is a task that should execute
        after all of the tasks in the header are complete.

    - ``map``

        The map primitive works like the built-in ``map`` function, but creates
        a temporary task where a list of arguments is applied to the task.
        E.g. ``task.map([1, 2])`` results in a single task
        being called, applying the arguments in order to the task function so
        that the result is::

            res = [task(1), task(2)]

    - ``starmap``

        Works exactly like map except the arguments are applied as ``*args``.
        For example ``add.starmap([(2, 2), (4, 4)])`` results in a single
        task calling::

            res = [add(2, 2), add(4, 4)]

    - ``chunks``

        Chunking splits a long list of arguments into parts, e.g the operation::

            >>> add.chunks(zip(xrange(1000), xrange(1000), 10))

        will create 10 tasks that apply 100 items each.


The primitives are also subtasks themselves, so that they can be combined
in any number of ways to compose complex workflows.

Here's some examples:

- Simple chain

    Here's a simple chain, the first task executes passing its return value
    to the next task in the chain, and so on.

    .. code-block:: python

        >>> from celery import chain

        # 2 + 2 + 4 + 8
        >>> res = chain(add.s(2, 2), add.s(4), add.s(8))()
        >>> res.get()
        16

    This can also be written using pipes::

        >>> (add.s(2, 2) | add.s(4) | add.s(8))().get()
        16

- Immutable subtasks

    Signatures can be partial so arguments can be
    added to the existing arguments, but you may not always want that,
    for example if you don't want the result of the previous task in a chain.

    In that case you can mark the subtask as immutable, so that the arguments
    cannot be changed::

        >>> add.subtask((2, 2), immutable=True)

    There's also an ``.si`` shortcut for this::

        >>> add.si(2, 2)

    Now you can create a chain of independent tasks instead::

        >>> res = (add.si(2, 2), add.si(4, 4), add.s(8, 8))()
        >>> res.get()
        16

        >>> res.parent.get()
        8

        >>> res.parent.parent.get()
        4

- Simple group

    You can easily create a group of tasks to execute in parallel::

        >>> from celery import group
        >>> res = group(add.s(i, i) for i in xrange(10))()
        >>> res.get(timeout=1)
        [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

    - For primitives `.apply_async` is special...

        as it will create a temporary task to apply the tasks in,
        for example by *applying the group*::

            >>> g = group(add.s(i, i) for i in xrange(10))
            >>> g()  # << applying

        the act of sending the messages for the tasks in the group
        will happen in the current process,
        but with ``.apply_async`` this happens in a temporary task
        instead::

            >>> g = group(add.s(i, i) for i in xrange(10))
            >>> g.apply_async()

        This is useful because you can e.g. specify a time for the
        messages in the group to be called::

            >>> g.apply_async(countdown=10)

- Simple chord

    The chord primitive enables us to add callback to be called when
    all of the tasks in a group have finished executing, which is often
    required for algorithms that aren't embarrassingly parallel::

        >>> from celery import chord
        >>> res = chord((add.s(i, i) for i in xrange(10)), xsum.s())()
        >>> res.get()
        90

    The above example creates 10 task that all start in parallel,
    and when all of them are complete the return values are combined
    into a list and sent to the ``xsum`` task.

    The body of a chord can also be immutable, so that the return value
    of the group is not passed on to the callback::

        >>> chord((import_contact.s(c) for c in contacts),
        ...       notify_complete.si(import_id)).apply_async()

    Note the use of ``.si`` above which creates an immutable subtask.

- Blow your mind by combining

    Chains can be partial too::

        >>> c1 = (add.s(4) | mul.s(8))

        # (16 + 4) * 8
        >>> res = c1(16)
        >>> res.get()
        160

    Which means that you can combine chains::

        # ((4 + 16) * 2 + 4) * 8
        >>> c2 = (add.s(4, 16) | mul.s(2) | (add.s(4) | mul.s(8)))

        >>> res = c2()
        >>> res.get()
        352

    Chaining a group together with another task will automatically
    upgrade it to be a chord::

        >>> c3 = (group(add.s(i, i) for i in xrange(10) | xsum.s()))
        >>> res = c3()
        >>> res.get()
        90

    Groups and chords accepts partial arguments too, so in a chain
    the return value of the previous task is forwarded to all tasks in the group::


        >>> new_user_workflow = (create_user.s() | group(
        ...                      import_contacts.s(),
        ...                      send_welcome_email.s()))
        ... new_user_workflow.delay(username='artv',
        ...                         first='Art',
        ...                         last='Vandelay',
        ...                         email='art@vandelay.com')


    If you don't want to forward arguments to the group then
    you can make the subtasks in the group immutable::

        >>> res = (add.s(4, 4) | group(add.si(i, i) for i in xrange(10)))()
        >>> res.get()
        <GroupResult: de44df8c-821d-4c84-9a6a-44769c738f98 [
            bc01831b-9486-4e51-b046-480d7c9b78de,
            2650a1b8-32bf-4771-a645-b0a35dcc791b,
            dcbee2a5-e92d-4b03-b6eb-7aec60fd30cf,
            59f92e0a-23ea-41ce-9fad-8645a0e7759c,
            26e1e707-eccf-4bf4-bbd8-1e1729c3cce3,
            2d10a5f4-37f0-41b2-96ac-a973b1df024d,
            e13d3bdb-7ae3-4101-81a4-6f17ee21df2d,
            104b2be0-7b75-44eb-ac8e-f9220bdfa140,
            c5c551a5-0386-4973-aa37-b65cbeb2624b,
            83f72d71-4b71-428e-b604-6f16599a9f37]>

        >>> res.parent.get()
        8


.. _canvas-chain:

Chains
------

.. versionadded:: 3.0

Tasks can be linked together, which in practice means adding
a callback task::

    >>> res = add.apply_async((2, 2), link=mul.s(16))
    >>> res.get()
    4

The linked task will be applied with the result of its parent
task as the first argument, which in the above case will result
in ``mul(4, 16)`` since the result is 4.

The results will keep track of what subtasks a task applies,
and this can be accessed from the result instance::

    >>> res.children
    [<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>]

    >>> res.children[0].get()
    64

The result instance also has a :meth:`~@AsyncResult.collect` method
that treats the result as a graph, enabling you to iterate over
the results::

    >>> list(res.collect())
    [(<AsyncResult: 7b720856-dc5f-4415-9134-5c89def5664e>, 4),
     (<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>, 64)]

By default :meth:`~@AsyncResult.collect` will raise an
:exc:`~@IncompleteStream` exception if the graph is not fully
formed (one of the tasks has not completed yet),
but you can get an intermediate representation of the graph
too::

    >>> for result, value in res.collect(intermediate=True)):
    ....

You can link together as many tasks as you like,
and subtasks can be linked too::

    >>> s = add.s(2, 2)
    >>> s.link(mul.s(4))
    >>> s.link(log_result.s())

You can also add *error callbacks* using the ``link_error`` argument::

    >>> add.apply_async((2, 2), link_error=log_error.s())

    >>> add.subtask((2, 2), link_error=log_error.s())

Since exceptions can only be serialized when pickle is used
the error callbacks take the id of the parent task as argument instead:

.. code-block:: python

    from proj.celery import celery

    @celery.task
    def log_error(task_id):
        result = celery.AsyncResult(task_id)
        result.get(propagate=False)  # make sure result written.
        with open('/var/errors/%s' % (task_id, )) as fh:
            fh.write('--\n\n%s %s %s' % (
                task_id, result.result, result.traceback))

To make it even easier to link tasks together there is
a special subtask called :class:`~celery.chain` that lets
you chain tasks together:

.. code-block:: python

    >>> from celery import chain
    >>> from proj.tasks import add, mul

    # (4 + 4) * 8 * 10
    >>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))
    proj.tasks.add(4, 4) | proj.tasks.mul(8) | proj.tasks.mul(10)


Calling the chain will call the tasks in the current process
and return the result of the last task in the chain::

    >>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()
    >>> res.get()
    640

And calling ``apply_async`` will create a dedicated
task so that the act of calling the chain happens
in a worker::

    >>> res = chain(add.s(4, 4), mul.s(8), mul.s(10)).apply_async()
    >>> res.get()
    640

It also sets ``parent`` attributes so that you can
work your way up the chain to get intermediate results::

    >>> res.parent.get()
    64

    >>> res.parent.parent.get()
    8

    >>> res.parent.parent
    <AsyncResult: eeaad925-6778-4ad1-88c8-b2a63d017933>


Chains can also be made using the ``|`` (pipe) operator::

    >>> (add.s(2, 2) | mul.s(8) | mul.s(10)).apply_async()

Graphs
~~~~~~

In addition you can work with the result graph as a
:class:`~celery.datastructures.DependencyGraph`:

.. code-block:: python

    >>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()

    >>> res.parent.parent.graph
    285fa253-fcf8-42ef-8b95-0078897e83e6(1)
        463afec2-5ed4-4036-b22d-ba067ec64f52(0)
    872c3995-6fa0-46ca-98c2-5a19155afcf0(2)
        285fa253-fcf8-42ef-8b95-0078897e83e6(1)
            463afec2-5ed4-4036-b22d-ba067ec64f52(0)

You can even convert these graphs to *dot* format::

    >>> with open('graph.dot', 'w') as fh:
    ...     res.parent.parent.graph.to_dot(fh)


and create images:

.. code-block:: bash

    $ dot -Tpng graph.dot -o graph.png

.. image:: ../images/graph.png

.. _canvas-group:

Groups
------

.. versionadded:: 3.0

A group can be used to execute several tasks in parallel.

The :class:`~celery.group` function takes a list of subtasks::

    >>> from celery import group
    >>> from proj.tasks import add

    >>> group(add.s(2, 2), add.s(4, 4))
    (proj.tasks.add(2, 2), proj.tasks.add(4, 4))

If you **call** the group, the tasks will be applied
one after one in the current process, and a :class:`~@TaskSetResult`
instance is returned which can be used to keep track of the results,
or tell how many tasks are ready and so on::

    >>> g = group(add.s(2, 2), add.s(4, 4))
    >>> res = g()
    >>> res.get()
    [4, 8]

However, if you call ``apply_async`` on the group it will
send a special grouping task, so that the action of calling
the tasks happens in a worker instead of the current process::

    >>> res = g.apply_async()
    >>> res.get()
    [4, 8]

Group also supports iterators::

    >>> group(add.s(i, i) for i in xrange(100))()

A group is a subtask instance, so it can be used in combination
with other subtasks.

Group Results
~~~~~~~~~~~~~

The group task returns a special result too,
this result works just like normal task results, except
that it works on the group as a whole::

    >>> from celery import group
    >>> from tasks import add

    >>> job = group([
    ...             add.subtask((2, 2)),
    ...             add.subtask((4, 4)),
    ...             add.subtask((8, 8)),
    ...             add.subtask((16, 16)),
    ...             add.subtask((32, 32)),
    ... ])

    >>> result = job.apply_async()

    >>> result.ready()  # have all subtasks completed?
    True
    >>> result.successful() # were all subtasks successful?
    True
    >>> result.join()
    [4, 8, 16, 32, 64]

The :class:`~celery.result.GroupResult` takes a list of
:class:`~celery.result.AsyncResult` instances and operates on them as
if it was a single task.

It supports the following operations:

* :meth:`~celery.result.GroupResult.successful`

    Returns :const:`True` if all of the subtasks finished
    successfully (e.g. did not raise an exception).

* :meth:`~celery.result.GroupResult.failed`

    Returns :const:`True` if any of the subtasks failed.

* :meth:`~celery.result.GroupResult.waiting`

    Returns :const:`True` if any of the subtasks
    is not ready yet.

* :meth:`~celery.result.GroupResult.ready`

    Return :const:`True` if all of the subtasks
    are ready.

* :meth:`~celery.result.GroupResult.completed_count`

    Returns the number of completed subtasks.

* :meth:`~celery.result.GroupResult.revoke`

    Revokes all of the subtasks.

* :meth:`~celery.result.GroupResult.iterate`

    Iterates over the return values of the subtasks
    as they finish, one by one.

* :meth:`~celery.result.GroupResult.join`

    Gather the results for all of the subtasks
    and return a list with them ordered by the order of which they
    were called.

.. _canvas-chord:

Chords
------

.. versionadded:: 2.3

A chord is a task that only executes after all of the tasks in a taskset have
finished executing.


Let's calculate the sum of the expression
:math:`1 + 1 + 2 + 2 + 3 + 3 ... n + n` up to a hundred digits.

First you need two tasks, :func:`add` and :func:`tsum` (:func:`sum` is
already a standard function):

.. code-block:: python

    @celery.task
    def add(x, y):
        return x + y

    @celery.task
    def tsum(numbers):
        return sum(numbers)


Now you can use a chord to calculate each addition step in parallel, and then
get the sum of the resulting numbers::

    >>> from celery import chord
    >>> from tasks import add, tsum

    >>> chord(add.subtask((i, i))
    ...     for i in xrange(100))(tsum.subtask()).get()
    9900


This is obviously a very contrived example, the overhead of messaging and
synchronization makes this a lot slower than its Python counterpart::

    sum(i + i for i in xrange(100))

The synchronization step is costly, so you should avoid using chords as much
as possible. Still, the chord is a powerful primitive to have in your toolbox
as synchronization is a required step for many parallel algorithms.

Let's break the chord expression down::

    >>> callback = tsum.subtask()
    >>> header = [add.subtask((i, i)) for i in xrange(100)]
    >>> result = chord(header)(callback)
    >>> result.get()
    9900

Remember, the callback can only be executed after all of the tasks in the
header have returned.  Each step in the header is executed as a task, in
parallel, possibly on different nodes.  The callback is then applied with
the return value of each task in the header.  The task id returned by
:meth:`chord` is the id of the callback, so you can wait for it to complete
and get the final return value (but remember to :ref:`never have a task wait
for other tasks <task-synchronous-subtasks>`)

.. _chord-important-notes:

Important Notes
~~~~~~~~~~~~~~~

By default the synchronization step is implemented by having a recurring task
poll the completion of the taskset every second, calling the subtask when
ready.

Example implementation:

.. code-block:: python

    def unlock_chord(taskset, callback, interval=1, max_retries=None):
        if taskset.ready():
            return subtask(callback).delay(taskset.join())
        raise unlock_chord.retry(countdown=interval, max_retries=max_retries)


This is used by all result backends except Redis and Memcached, which
increment a counter after each task in the header, then applying the callback
when the counter exceeds the number of tasks in the set. *Note:* chords do not
properly work with Redis before version 2.2; you will need to upgrade to at
least 2.2 to use them.

The Redis and Memcached approach is a much better solution, but not easily
implemented in other backends (suggestions welcome!).


.. note::

    If you are using chords with the Redis result backend and also overriding
    the :meth:`Task.after_return` method, you need to make sure to call the
    super method or else the chord callback will not be applied.

    .. code-block:: python

        def after_return(self, *args, **kwargs):
            do_something()
            super(MyTask, self).after_return(*args, **kwargs)

.. _canvas-map:

Map & Starmap
-------------

:class:`~celery.map` and :class:`~celery.starmap` are built-in tasks
that calls the task for every element in a sequence.

They differ from group in that

- only one task message is sent

- the operation is sequential.

For example using ``map``:

.. code-block:: python

    >>> from proj.tasks import add

    >>> ~xsum.map([range(10), range(100)])
    [45, 4950]

is the same as having a task doing:

.. code-block:: python

    @celery.task
    def temp():
        return [xsum(range(10)), xsum(range(100))]

and using ``starmap``::

    >>> ~add.starmap(zip(range(10), range(10)))
    [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

is the same as having a task doing:

.. code-block:: python

    @celery.task
    def temp():
        return [add(i, i) for i in range(10)]

Both ``map`` and ``starmap`` are subtasks, so they can be used as
other subtasks and combined in groups etc., for example
to call the starmap after 10 seconds::

    >>> add.starmap(zip(range(10), range(10))).apply_async(countdown=10)

.. _canvas-chunks:

Chunks
------

Chunking lets you divide an iterable of work into pieces, so that if
you have one million objects, you can create 10 tasks with hundred
thousand objects each.

Some may worry that chunking your tasks results in a degradation
of parallelism, but this is rarely true for a busy cluster
and in practice since you are avoiding the overhead  of messaging
it may considerably increase performance.

To create a chunks subtask you can use :meth:`@Task.chunks`:

.. code-block:: python

    >>> add.chunks(zip(range(100), range(100)), 10)

As with :class:`~celery.group` the act of **calling**
the chunks will call the tasks in the current process:

.. code-block:: python

    >>> from proj.tasks import add

    >>> res = add.chunks(zip(range(100), range(100)), 10)()
    >>> res.get()
    [[0, 2, 4, 6, 8, 10, 12, 14, 16, 18],
     [20, 22, 24, 26, 28, 30, 32, 34, 36, 38],
     [40, 42, 44, 46, 48, 50, 52, 54, 56, 58],
     [60, 62, 64, 66, 68, 70, 72, 74, 76, 78],
     [80, 82, 84, 86, 88, 90, 92, 94, 96, 98],
     [100, 102, 104, 106, 108, 110, 112, 114, 116, 118],
     [120, 122, 124, 126, 128, 130, 132, 134, 136, 138],
     [140, 142, 144, 146, 148, 150, 152, 154, 156, 158],
     [160, 162, 164, 166, 168, 170, 172, 174, 176, 178],
     [180, 182, 184, 186, 188, 190, 192, 194, 196, 198]]

while calling ``.apply_async`` will create a dedicated
task so that the individual tasks are applied in a worker
instead::

    >>> add.chunks(zip(range(100), range(100), 10)).apply_async()

You can also convert chunks to a group::

    >>> group = add.chunks(zip(range(100), range(100), 10)).group()

and with the group skew the countdown of each task by increments
of one::

    >>> group.skew(start=1, stop=10)()

which means that the first task will have a countdown of 1, the second
a countdown of 2 and so on.
