.. _consuming_packages_flexibility_of_conanfile_py:

Understanding the flexibility of using conanfile.py vs conanfile.txt
====================================================================

In the previous examples, we declared our dependencies (*Zlib* and *CMake*) in a
*conanfile.txt* file. Let's have a look at that file:

.. code-block:: ini
    :caption: **conanfile.txt**

    [requires]
    zlib/1.2.11

    [tool_requires]
    cmake/3.22.6

    [generators]
    CMakeDeps
    CMakeToolchain

Using a *conanfile.txt* to build your projects using Conan it's enough for simple cases,
but if you need more flexibility you should use a *conanfile.py* file where you can use
Python code to make things such as adding requirements dynamically, changing options
depending on other options or setting options for your requirements. Let's see an example
on how to migrate to a *conanfile.py* and use some of those features.

Please, first clone the sources to recreate this project. You can find them in the
`examples2.0 repository <https://github.com/conan-io/examples2>`_ in GitHub:

.. code-block:: bash

    $ git clone https://github.com/conan-io/examples2.git
    $ cd examples2/tutorial/consuming_packages/conanfile_py

Check the contents of the folder and note that the contents are the same that in the
previous examples but with a *conanfile.py* instead of a *conanfile.txt*.

.. code-block:: bash

    .
    ├── CMakeLists.txt
    ├── conanfile.py
    └── src
        └── main.c

Remember that in the previous examples the *conanfile.txt* had this information:

.. code-block:: ini
    :caption: **conanfile.txt**

    [requires]
    zlib/1.2.11

    [tool_requires]
    cmake/3.22.6

    [generators]
    CMakeDeps
    CMakeToolchain

We will translate that same information to a *conanfile.py*. This file is what is
typically called a **"Conan recipe"**. It can be used for consuming packages, like in this
case, and also to create packages. For our current case, it will define our requirements
(both libraries and build tools) and logic to modify options and set how we want to
consume those packages. In the case of using this file to create packages, it can define
(among other things) how to download the package’s source code, how to build the binaries
from those sources, how to package the binaries, and information for future consumers on
how to consume the package. We will explain how to use Conan recipes to create
packages in the :ref:`Creating Packages<tutorial_creating_packages>` section later.

The equivalent of the *conanfile.txt* in form of Conan recipe could look like this:

.. code-block:: python
    :caption: **conanfile.py**

    from conan import ConanFile


    class CompressorRecipe(ConanFile):
        settings = "os", "compiler", "build_type", "arch"
        generators = "CMakeToolchain", "CMakeDeps"

        def requirements(self):
            self.requires("zlib/1.2.11")
        
        def build_requirements(self):
            self.tool_requires("cmake/3.22.6")


To create the Conan recipe we declared a new class that inherits from the ``ConanFile``
class. This class has different class attributes and methods:

* **settings** this class attribute defines the project-wide variables, like the compiler,
  its version, or the OS itself that may change when we build our project. This is related
  to how Conan manages binary compatibility as these values will affect the value of the
  **package ID** for Conan packages. We will explain how Conan uses this value to manage
  binary compatibility later.
* **generators** this class attribute specifies which Conan generators will be run when we
  call the :command:`conan install` command. In this case, we added **CMakeToolchain** and
  **CMakeDeps** as in the *conanfile.txt*.
* **requirements()** in this method we can use the ``self.requires()`` and
  ``self.tool_requires()`` methods to declare all our dependencies (libraries and build
  tools).

You can check that running the same commands as in the previous examples will lead to the
same results as before.

.. code-block:: bash
    :caption: Windows

    $ conan install . --output-folder=build --build=missing
    $ cd build
    $ conanbuild.bat
    # assuming Visual Studio 15 2017 is your VS version and that it matches your default profile
    $ cmake .. -G "Visual Studio 15 2017" -DCMAKE_TOOLCHAIN_FILE=conan_toolchain.cmake
    $ cmake --build . --config Release
    ...
    Building with CMake version: 3.22.6
    ...
    [100%] Built target compressor

    $ Release\compressor.exe
    Uncompressed size is: 233
    Compressed size is: 147
    ZLIB VERSION: 1.2.11
    $ deactivate_conanbuild.bat

.. code-block:: bash
    :caption: Linux, macOS
    
    $ conan install . --output-folder build --build=missing
    $ cd build
    $ source conanbuild.sh
    Capturing current environment in deactivate_conanbuildenv-release-x86_64.sh
    Configuring environment variables    
    $ cmake .. -DCMAKE_TOOLCHAIN_FILE=conan_toolchain.cmake -DCMAKE_BUILD_TYPE=Release
    $ cmake --build .
    ...
    Building with CMake version: 3.22.6
    ...
    [100%] Built target compressor

    $ ./compressor
    Uncompressed size is: 233
    Compressed size is: 147
    ZLIB VERSION: 1.2.11
    $ source deactivate_conanbuild.sh

So far we have achieved the same functionality we had using a *conanfile.txt*, let's see
how we can take advantage of the capabilities of the *conanfile.py* to define the project
structure we want to follow and also to add some logic using Conan settings and options.

.. _consuming_packages_flexibility_of_conanfile_py_use_layout:

Use the layout() method
-----------------------

In the previous examples, every time we executed a `conan install` command, we had to use
the `--output-folder` argument to define where we wanted to create the files that Conan
generates. There's a neater way to decide where we want Conan to generate the files for
the build system that will allow us to decide, for example, if we want different output
folders depending on the type of CMake generator we are using. You can define this
directly in the `conanfile.py` inside the `layout()` method and make it work for every
platform without adding more changes.


.. code-block:: python
    :caption: **conanfile.py**

    import os

    from conan import ConanFile


    class CompressorRecipe(ConanFile):
        settings = "os", "compiler", "build_type", "arch"
        generators = "CMakeToolchain", "CMakeDeps"

        def requirements(self):
            self.requires("zlib/1.2.11")

        def build_requirements(self):
            self.tool_requires("cmake/3.22.6")

        def layout(self):
            # We make the assumption that if the compiler is msvc the
            # CMake generator is multi-config
            multi = True if self.settings.get_safe("compiler") == "msvc" else False
            if multi:
                self.folders.generators = os.path.join("build", "generators")
            else:
                self.folders.generators = os.path.join("build", str(self.settings.build_type), "generators")


As you can see, we defined the **self.folders.generators** attribute in the `layout()`
method. This is the folder where all the auxiliary files generated by Conan (CMake
toolchain and cmake dependencies files) will be placed.

Note that the definitions of the folders is different if it is a multi-config generator
(like Visual Studio), or a single-config generator (like Unix Makefiles). In the
first case, the folder is the same irrespective of the build type, and the build system
will manage the different build types inside that folder. But single-config generators
like Unix Makefiles, must use a different folder for each different configuration (as a
different build_type Release/Debug). In this case we added a simple logic to consider
multi-config if the compiler name is `msvc`.

Check that running the same commands as in the previous examples without the
`--output-folder` argument will lead to the same results as before:

.. code-block:: bash
    :caption: Windows

    $ conan install . --build=missing
    $ cd build
    $ generators\conanbuild.bat
    # assuming Visual Studio 15 2017 is your VS version and that it matches your default profile
    $ cmake .. -G "Visual Studio 15 2017" -DCMAKE_TOOLCHAIN_FILE=generators\conan_toolchain.cmake
    $ cmake --build . --config Release
    ...
    Building with CMake version: 3.22.6
    ...
    [100%] Built target compressor

    $ Release\compressor.exe
    Uncompressed size is: 233
    Compressed size is: 147
    ZLIB VERSION: 1.2.11
    $ generators\deactivate_conanbuild.bat

.. code-block:: bash
    :caption: Linux, macOS
    
    $ conan install . --build=missing
    $ cd build
    $ source ./Release/generators/conanbuild.sh
    Capturing current environment in deactivate_conanbuildenv-release-x86_64.sh
    Configuring environment variables    
    $ cmake .. -DCMAKE_TOOLCHAIN_FILE=Release/generators/conan_toolchain.cmake -DCMAKE_BUILD_TYPE=Release
    $ cmake --build .
    ...
    Building with CMake version: 3.22.6
    ...
    [100%] Built target compressor

    $ ./compressor
    Uncompressed size is: 233
    Compressed size is: 147
    ZLIB VERSION: 1.2.11
    $ source ./Release/generators/deactivate_conanbuild.sh

There's no need to always write this logic in the `conanfile.py`. There are some
pre-defined layouts you can import and directly use in your recipe. For example, for the
CMake case, there's a :ref:`cmake_layout()<cmake_layout>` already defined in Conan:

.. code-block:: python
    :caption: **conanfile.py**

    from conan import ConanFile
    from conan.tools.cmake import cmake_layout


    class CompressorRecipe(ConanFile):
        settings = "os", "compiler", "build_type", "arch"
        generators = "CMakeToolchain", "CMakeDeps"

        def requirements(self):
            self.requires("zlib/1.2.11")

        def build_requirements(self):
            self.tool_requires("cmake/3.22.6")

        def layout(self):
            cmake_layout(self)


Use the validate() method to raise an error for non-supported configurations
----------------------------------------------------------------------------

The :ref:`validate() method<reference_conanfile_methods_validate>` is evaluated when Conan loads the *conanfile.py* and you can use
it to perform checks of the input settings. If, for example, your project does not support
*armv8* architecture on macOS you can raise the `ConanInvalidConfiguration` exception to
make Conan return with a special error code. This will indicate that the configuration
used for settings or options is not supported.


.. code-block:: python
    :caption: **conanfile.py**

    ...
    from conan.errors import ConanInvalidConfiguration

    class CompressorRecipe(ConanFile):
        ...

        def validate(self):
            if self.settings.os == "Macos" and self.settings.arch == "armv8":
                raise ConanInvalidConfiguration("ARM v8 not supported in Macos")


Conditional requirements using a conanfile.py
---------------------------------------------

You could add some logic to the :ref:`requirements() method<reference_conanfile_methods_requirements>` to add or remove requirements
conditionally. Imagine, for example, that you want to add an additional dependency in
Windows or that you want to use the system's CMake installation instead of using the Conan
`tool_requires`:

.. code-block:: python
    :caption: **conanfile.py**

    from conan import ConanFile


    class CompressorRecipe(ConanFile):
        # Binary configuration
        settings = "os", "compiler", "build_type", "arch"
        generators = "CMakeToolchain", "CMakeDeps"

        def requirements(self):
            self.requires("zlib/1.2.11")
            
            # Add base64 dependency for Windows
            if self.settings.os == "Windows":
                self.requires("base64/0.4.0")

        def build_requirements(self):
            # Use the system's CMake for Windows
            if self.settings.os != "Windows":
                self.tool_requires("cmake/3.22.6")


Read more
---------

.. container:: examples

    - :ref:`Using "cmake_layout" + "CMakeToolchain" + "CMakePresets feature" to build your project<examples-tools-cmake-toolchain-build-project-presets>`.
    - :ref:`Understanding the Conan Package layout<tutorial_package_layout>`.
    - Importing resource files in the generate() method
    - Conditional generators in configure()
