Working with Unit Tests
=======================

When developing packages, it's common to have test code along side the source code with support in the build scripts

Many of the build helpers, like <ref:cmake:https://docs.conan.io/en/latest/reference/build_helpers/cmake.html#test>


.. code-block:: python

    cmake = CMake(self)
    cmake.configure()
    cmake.build()
    cmake.test()
    
Conditional Build Tests
-----------------------

https://cpplang.slack.com/archives/C41CWV9HA/p1660814759731709?thread_ts=1660787715.224249&cid=C41CWV9HA
