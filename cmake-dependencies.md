# Managing C++ Dependencies with CMake

Use CMake's `FetchContent` module to download and build dependencies without installing them system-wide.

## 1. Declare the dependency

Add the following after `project()` in your `CMakeLists.txt`. This example uses GoogleTest:

```cmake
include(FetchContent)

FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.17.0
)

FetchContent_MakeAvailable(googletest)
```

Pin `GIT_TAG` to a release or commit for reproducible builds.

## 2. Link the dependency

After declaring your executable or library, link it to the dependency's exported CMake target:

```cmake
add_executable(my_tests tests/test_main.cpp)
target_link_libraries(my_tests PRIVATE GTest::gtest_main)
```

The target name varies by dependency; consult its documentation.

## 3. Configure and build

```bash
cmake -S . -B build
cmake --build build
```

CMake downloads the dependency during configuration and builds it as needed. By default, dependency files are stored under `build/_deps/`.

## Notes

- Requires Git and internet access for the initial download when using a Git repository.
- The dependency must support CMake integration for this workflow.
- No system-wide installation is needed for the fetched library, although its own prerequisites may still be required.
- Add `build/` to `.gitignore`.
