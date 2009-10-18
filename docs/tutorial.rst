=====================
Overview and Tutorial
=====================

This document provides a high level overview of Fabric's functionality and use,
including a number of real-world examples. Throughout the tutorial, links will
be provided to the rest of the documentation -- be sure to follow them for
details on any particular topic.


.. _introduction:

Introduction
============

Fabric is primarily designed to do two things:

* Run Python functions from the command line, with the ``fab`` tool;
* Execute shell commands locally or on remote servers, with the Fabric library.

We'll tackle these in order, and then see how to use them together, which is
the primary use case.

Python on the command line: the ``fab`` tool
--------------------------------------------

Fabric's main interface is a command-line script called ``fab``, which is
capable of loading a Python module (or "fabfile") and executing one or more of
the functions defined within (also known as "tasks" or "commands".)

A "Hello World" example of this would be creating a ``fabfile.py`` with the
following contents::

    def hello():
        print("Hello, world!")

The ``hello`` task can then be executed with ``fab`` like so, provided you're
in the same directory as ``fabfile.py``::

    $ fab hello
    Hello, world!

    Done.

That's all there is to it: define one or more tasks, then ask ``fab`` to
execute them. For details on ``fab``'s behavior and its options/arguments,
please see :doc:`usage/fab`. We'll also be highlighting a handful of common
options throughout the tutorial.

.. _library-usage:

Local and remote shell commands: the Fabric library
---------------------------------------------------

Fabric provides a number of core API functions (sometimes referred to as
"operations") revolving around the execution of shell commands. Aside from a
convenient function for calling a local shell, most of these functions use the
SSH protocol to interact with remote servers.

Use of this API is relatively simple: set an :doc:`environment variable
<usage/env>` telling Fabric what server to talk to, and call the desired
function or functions.

Here's an interactive Python session making use of the `~fabric.operations.run`
function (which executes the given string in a remote shell and returns the
output) where we list the document-root folders on a hypothetical Web server::

    $ python
    >>> from fabric.api import run, env
    >>> from fabric.state import connections
    >>> env.host_string = 'example.com'
    >>> result = run("ls /var/www")
    [example.com] run: ls
    [example.com] out: www.example.com
    [example.com] out: code.example.com
    [example.com] out: webmail.example.com
    >>> connections['example.com'].close()
    >>> ^D
    $ 

As you can see, `~fabric.operations.run` prints out what it's doing, as well as
the standard output from the remote end, in addition to returning the final result.

.. note::

    The use of the ``connections`` object to close the connection is necessary
    in order to cleanly exit the Python interpreter. This is less than ideal,
    and Fabric's usability as a library is expected to improve in version 1.0.
    In normal use, you won't have to worry about this -- see the next section.

Putting them together
---------------------

While these two aspects of Fabric can be used separately, the main use case is
to combine them by using ``fab`` to execute tasks which import the API
functions.  Most of Fabric's auxiliary functions and tools revolve around using
it in this manner.

Here's an example which simply takes the previous interactive example and drops
it into a fabfile::

    from fabric.api import run, env

    def list_docroots():
        env.host_string = 'example.com'
        result = run("ls /var/www")

.. note::

    When using functions like `~fabric.operations.run` in ``fab``-driven
    fabfiles, you don't need to bother with the ``connections`` object -- it's
    handled for you by ``fab``'s main execution loop. See
    :doc:`usage/execution` for more on how the ``fab`` tool handles
    connections.

The result is much the same as before::

    $ fab list_docroots

    [example.com] run: ls
    [example.com] out: www.example.com
    [example.com] out: code.example.com
    [example.com] out: webmail.example.com

    Done.
    Disconnecting from example.com... done.

From here on, we'll be exploring the rest of Fabric's API and the various nuts
and bolts you'll need to understand in order to use Fabric effectively. We'll
also be creating more realistic examples now that you have the background to
understand them.


Operations
==========

In this section we'll give a quick tour of Fabric's basic building blocks, the
:doc:`operations <api/core/operations>`. These the most commonly utilized parts of
Fabric's API, and also form the foundation for the :ref:`contrib <contrib-api>`
modules.

.. note::

    Follow any hyperlinked function name to see its full API documentation.

`~fabric.operations.run` and `~fabric.operations.sudo`
------------------------------------------------------

You've already seen how `~fabric.operations.run` executes a given command in a
remote shell; it has a close cousin, `~fabric.operations.sudo`, which is
identical save for the fact that it automatically wraps your command inside a
``sudo`` call. `~fabric.operations.sudo` is also capable of detecting
``sudo``'s password prompt and passing it through to your terminal.

.. note::

    Hyperlinked versions of the word "sudo" (e.g. `~fabric.operations.sudo`)
    refer to the Python function; non-hyperlinked, monospaced versions
    (``sudo``) refer to the command-line program which the function uses.

`~fabric.operations.sudo` finds a lot of use in any scenario where you're
interacting with system services, such as in this task you might use to
restart various services via init scripts::

    from fabric.api import sudo

    def restart(service):
        sudo('/etc/init.d/%s restart' % service)

Usage::

    $ fab -H example.com restart:service=apache2
    [example.com] sudo: /etc/init.d/apache2 restart
    Password for username@example.com: 
    [example.com] out: Restarting web server apache2
    [example.com] out: ...done.

    Done.
    Disconnecting from example.com... done.

The above highlights a couple of additional ``fab`` features besides
`~fabric.operations.sudo`'s password prompt detection:

* The ``-H`` option, allowing you to define the host or hosts to connect to.
  See :doc:`usage/execution` below for more on this and other ways of defining host
  connections.
* The ability to specify task arguments on the command line. See
  :doc:`usage/fab` for details on how to specify Python function arguments and
  keyword arguments in this manner.

For more details on how `~fabric.operations.run` and `~fabric.operations.sudo`
interact with the SSH protocol -- including the shell loaded on the remote end,
key-based authentication and more -- please see :doc:`usage/ssh`.

`~fabric.operations.local`
--------------------------

While much of the Fabric API deals with remote servers, it's often necessary to
work locally as well. To handle this, Fabric wraps the stdlib ``subprocess``
module in a function similar to `~fabric.operations.run` and
`~fabric.operations.sudo`, called `~fabric.operations.local`. 

.. note::

    `~fabric.operations.local`'s behavior is not yet a perfect copy of that
    seen in `~fabric.operations.run` and `~fabric.operations.sudo` -- for
    example, it cannot capture **and** print output at the same time. This
    should improve in version 1.0.

Here's a sample taken from Fabric's own internal fabfile, which executes the
test suite and displays the output::

    from fabric.api import local

    def test():
        print(local('nosetests -sv --with-doctest', capture=False))

A truncated version of the output::

    $ fab test
    [localhost] run: nosetests -sv --with-doctest
    Doctest: fabric.operations._shell_escape ... ok
    Aborts if any given roles aren't found ... ok
    Use of @roles and @hosts together results in union of both ... ok
    If @hosts is used it replaces any env.hosts value ... ok
    [...]
    Aliases can be nested ... ok
    Alias expansion ... ok
    warn() should print 'Warning' plus given text ... ok
    indent(strip=True): Sanity check: 1 line string ... ok
    abort() should raise SystemExit ... ok
    ----------------------------------------------------------------------
    Ran 63 tests in 0.606s

    OK

    Done.

`~fabric.operations.put` and `~fabric.operations.get`
-----------------------------------------------------

In addition to executing shell commands over SSH, Fabric can leverage SFTP to
upload and download files, via the `~fabric.operations.put` and
`~fabric.operations.get` functions respectively.

The builtin ``contrib`` function `~fabric.contrib.project.upload_project`
combines `~fabric.operations.local`, `~fabric.operations.run` and
`~fabric.operations.put` to transmit a copy of the current project to the
remote server, and provides a handy example which covers most of the topics
seen so far. What follows is a modified version of the real thing::

    from fabric.api import local, put, run

    def upload_project():
        fname = "project.tgz"
        fpath = "/tmp/%s" % fname
        local("tar -czf %s ." % fpath)
        dest = "/var/www/%s" % fname
        put(fpath, dest)
        run("cd /var/www && tar -xzf %s" % fname)
        run("rm -f %s" % dest)

Running it doesn't provide much output, provided things go well (which is
generally the Unix way -- be silent unless something is wrong -- and this is
followed by the commands we call here: ``tar``, ``cd`` and ``rm``)::

    $ fab -H example.com upload_project
    [localhost] run: tar -czf /tmp/project.tgz .
    [ubuntu904] put: /tmp/project.tgz -> /var/www/project.tgz
    [ubuntu904] run: cd /var/www && tar -xzf project.tgz
    [ubuntu904] run: rm -f /var/www/project.tgz

As always, click any hyperlinked function name to see the API documentation.

`require` and `~fabric.operations.prompt`
-----------------------------------------

In addition to the previous operations, which allow you to effect actual
changes on local or remote machines, Fabric's operations module provides two
convenience methods: `~fabric.operations.require` and
`~fabric.operations.prompt`:

* `~fabric.operations.require` lets you ensure that a task will abort if some
  needed information is not present, which can be handy if you have a small
  network of inter-operating tasks (see :doc:`usage/env` for more.)
* `~fabric.operations.prompt` is a convenience wrapper around Python's
  ``raw_input`` builtin that asks the user to enter a string, which can be
  useful for interactive tasks.




Conclusion
==========

This concludes the tutorial and overview. We've only touched on the basics
here; we hope you've been following the provided links to the detailed
documentation on various topics. For the full documentation list, see :ref:`the
index page <usage-docs>`.