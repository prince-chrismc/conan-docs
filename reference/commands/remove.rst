conan remove
============

.. code-block:: text

    $ conan remove -h
    usage: conan remove [-h] [-v [V]] [-c] [-p PACKAGE_QUERY]
                        [-r REMOTE]
                        reference

    Remove recipes or packages from local cache or a remote.

    - If no remote is specified (-r), the removal will be done in the local conan cache.
    - If a recipe reference is specified, it will remove the recipe and all the packages, unless -p
      is specified, in that case, only the packages matching the specified query (and not the recipe)
      will be removed.
    - If a package reference is specified, it will remove only the package.

    positional arguments:
      reference             Recipe reference or package reference, can contain *
                            aswildcard at any reference field. e.g: lib/*

    optional arguments:
      -h, --help            show this help message and exit
      -v [V]                Level of detail of the output. Valid options from less
                            verbose to more verbose: -vquiet, -verror, -vwarning,
                            -vnotice, -vstatus, -v or -vverbose, -vv or -vdebug,
                            -vvv or -vtrace
      -c, --confirm         Remove without requesting a confirmation
      -p PACKAGE_QUERY, --package-query PACKAGE_QUERY
                            Remove all packages (empty) or provide a query:
                            os=Windows AND (arch=x86 OR compiler=gcc)
      -r REMOTE, --remote REMOTE
                            Will remove from the specified remote


The ``conan remove`` command removes recipes and packages from the local cache or from a
specified remote. Depending on the patterns specified as argument, it is possible to
remove a complete package, or just remove the binaries, leaving still the recipe
available. You can also use the keyword ``!latest`` in the revision part of the pattern to
avoid removing the latest recipe or package revision of a certain Conan package.

To remove recipes and their associated package binaries from the local cache:


.. code-block:: text

    $ conan remove "*"
    # Removes everything from the cache

    $ conan remove zlib/*
    # Remove all possible versions of zlib, including all recipes, revisions and packages

    $ conan remove zlib/1.2.11
    # Remove zlib/1.2.11, all its revisions and package binaries. Leave other zlib versions

    $ conan remove zlib/1.2.11#latest
    # Remove zlib/1.2.11, only its latest recipe revision and binaries of that revision
    # Leave the other zlib/1.2.11 revisions intact

    $ conan remove zlib/1.2.11#!latest
    # Remove all the recipe revisions from zlib/1.2.11 but the latest one
    # Leave the latest zlib/1.2.11 revision intact

    $ conan remove zlib/1.2.11#<revision>
    # Remove zlib/1.2.11, only its exact <revision> and binaries of that revision
    # Leave the other zlib/1.2.11 revisions intact


To remove only package binaries, but leaving the recipes, it is necessary to specify the
pattern including the ``:`` separator of the ``package_id``:

.. code-block:: text

    $ conan remove zlib/1.2.11:*
    # Removes all the zlib/1.2.11 package binaries from all the recipe revisions

    $ conan remove zlib/*:*
    # Removes all the binaries from all the recipe revisions from all zlib versions

    $ conan remove zlib/1.2.11#latest:*
    # Removes all the zlib/1.2.11 package binaries only from the latest zlib/1.2.11 recipe revision

    $ conan remove zlib/1.2.11#!latest:*
    # Removes all the zlib/1.2.11 package binaries from all the recipe revisions but the latest one

    $ conan remove zlib/1.2.11:<package_id>
    # Removes the package binary <package_id> from all the zlib/1.2.11 recipe revisions

    $ conan remove zlib/1.2.11:#latest<package_id>#latest
    # Removes only the latest package revision of the binary identified with <package_id>
    # from the latest recipe revision of zlib/1.2.11
    # WARNING: Recall that having more than 1 package revision is a smell and shouldn't happen
    # in normal situations


Note that you can filter which packages will be removed using the ``--package-query`` argument:

.. code-block:: text

    $ conan remove zlib/1.2.11:* -p compiler=clang
    # Removes all the zlib/1.2.11 packages built with Clang compiler


You can query packages by both their settings and options, including custom ones.
To query for options you need to explicitly add the `options.` prefix, so that
`-p options.shared=False` will work but `-p shared=False` won't.



All the above commands, by default, operate in the Conan cache.
To remove artifacts from a server, use the ``-r=myremote`` argument:

.. code-block:: text

    $ conan remove zlib/1.2.11:* -r=myremote
    # Removes all the zlib/1.2.11 package binaries from all the recipe revisions in 
    # the remote <myremote>
