# resource_pack

A CMake library used to embed arbitrary binary data files inside your C++ application binary.

### Overview of the problem

For many applications it is often necessary to use some kind of data, that's not code. Things like shaders, textures,
fonts, csv, sounds, map data, etc. The usual approach to this is distributing the data alongside application's binary as
files and loading these files at runtime.

Although, it's often very convenient to bake the data inside the executable to reduce the potential environment
problems, like path resolving, handling bad formats, permissions, deleted files, etc. It makes the application more
predictable and easier to distribute. On the other hand, it makes the binary bigger, may cause slower startup times and
doesn't allow for conditionally used resources or user-provided resources, like plugins - all resources must be known at
compile time.

### Solution in resource_pack

This library embeds the binary files directly in an executable or library file using platform-specific tools: `ld` with
special parameters on Linux and [PE resources](https://learn.microsoft.com/en-us/windows/win32/menurc/using-resources)
on Windows. The compile time overhead is minimal - there's no need to parse the binaries during application build like
in other popular methods like [xxd](https://linux.die.net/man/1/xxd). There is also little to no runtime overhead. On
Linux it's all done at link time. On Windows we have to execute 4 Windows API calls per resource file. It's performed
implicitly before the `main()` function starts by using static object initialization technique. If this ever becomes a
startup time problem, this can always be changed to an explicit `init_resource_pack()` API function that could be called
conditionally and/or in background thread.

### Usage

To use this library, simply include it in your `CMakeLists.txt` and call the main entrypoint - `resource_pack_add`
function. Specify an absolute path to the resource file, the name of the target to embed the file into, and a resource
group name - an arbitrary user-defined string used for naming the generated header file and some internal bookkeeping. A
target can have multiple groups. Although, it's okay to put all your resources in a single group. Groups are just a way
to logically separate you resources. Note a given group name can be only used for one target.

```cmake
add_executable(MyApplication)

include(resource_pack/resource_pack.cmake)
resource_pack_add(MyApplication images "${CMAKE_SOURCE_DIR}/mona_lisa.png")
resource_pack_add(MyApplication images "${CMAKE_SOURCE_DIR}/the_last_supper.png")
resource_pack_add(MyApplication images "${CMAKE_SOURCE_DIR}/bitwa_pod_grunwaldem.png")
resource_pack_set_source_group(images images) # Optional call to make it look nicer in Visual Studio
```

These instructions will embed the three specified *.png* files into `MyApplication` target's binary. It'll also generate
an interface header used to access the embedded data and set up include directories, so you can write:

```c
#include "resource_pack/images/images.h
```

Note the header is named `images.h`, because we specified `images` as the group name in our `resource_pack_add()` call.
The file will contain the following definitions. They are ready to use and automatically populated.

```c
struct EmbeddedResource {
    const void *data;
    size_t size;
};

extern const EmbeddedResource &mona_lisa;
extern const EmbeddedResource &the_last_supper;
extern const EmbeddedResource &bitwa_pod_grunwaldem;
```

### Other solutions

It is possible to convert binary files into C/C++ header files containing `char` arrays. See tools
like [header_pack](https://github.com/DziubanMaciej/header_pack) or [xxd](https://linux.die.net/man/1/xxd). This can,
however, become a significant burden in terms of compile times, because the compiler has to parse the data byte per
byte.

There's a [proposal](https://en.cppreference.com/w/cpp/preprocessor/embed.html) for C++ to add `#embed` preprocessor
directive. It is quite convenient, however its support is lacking (as of 2026) and it will likely suffer from the same
compile time problems as the previous solution.

There's a possibility to embed files using assembly `.incbin` directive in GNU or `INCBIN` macros on MSVC . It's native
and OS-independent and supported on multiple processor architectures. A very good solution as well.

### Possible improvements

Custom resource names instead of simply using file names.

Namespaces or name prefixes for the `EmbeddedResource` objects.

Support for C applications.

Explicit initialization function to optimize startup time on Windows.

Add `cmake_minimum_required` call. I have no idea what is the minimum version that will work. Would have to download a
bunch of versions and check. So far, I've tested on 4.2.3.

Do not regenerate intermediate files like the interface header or windows `.rc` file if nothing changed. Currently,
running CMake may cause a recompile of files including `resource_pack` headers.
