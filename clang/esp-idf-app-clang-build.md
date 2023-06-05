## How to build ESP-IDF application with clang?

If you'd like to try building an ESP-IDF application with clang, follow these steps:

1. Get `clang`. There are two options:
   1. Install `clang` supplied with ESP-IDF via `idf_tools.py install esp-clang`
   2. Use custom `clang` distribution:
      * Download clang based toolchain from https://github.com/espressif/llvm-project/releases/.
      * Extract the toolchain and add its `bin` directory to your `PATH` environment variable.
        ```bash
        export PATH=</path/to/distro>/esp-clang/bin:$PATH
        ```
        or, on Windows (CMD)
        ```cmd
        set PATH=<path\to\distro>\esp-clang\bin;%PATH%
        ```

2. Check that correct `clang` is available: `clang -print-targets` should list `xtensa` and/or `riscv32`.

3. Set `IDF_TOOLCHAIN` environment variable:
   ```bash
   export IDF_TOOLCHAIN=clang
   ```
   or, on Windows (CMD)
   ```cmd
   set IDF_TOOLCHAIN=clang
   ```
4. Clean the project (compiler is cached in build/CMakeCache.txt) and then build it:
   ```
   idf.py fullclean
   idf.py build
   ```
5. Alternatively, you can pass IDF_TOOLCHAIN as a CMake variable to idf.py
   ```
   idf.py fullclean
   idf.py -DIDF_TOOLCHAIN=clang build
   ```