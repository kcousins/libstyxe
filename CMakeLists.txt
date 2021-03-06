cmake_minimum_required(VERSION 3.3)
project(styxe)


include(CheckCXXCompilerFlag)


# Custom build target to generate code coverage:
# Don't want this to run on every build.
option(STYXE_GTEST_SUPPORT "Build without GTEST" ON)
option(STYXE_COVERALLS "Generate coveralls data" OFF)
option(STYXE_SANITIZE "Enable 'sanitize' compiler flag" OFF)
option(STYXE_LIBSOLACE_SUPPORT "Build without libsolace" ON)


# Require at least C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED on)

# Checks for optional extra compiler options
check_cxx_compiler_flag("-fvtable-verify" WITH_VTABLE_VERIFY)
check_cxx_compiler_flag("-fstack-protector" WITH_STACK_PROTECTOR)
check_cxx_compiler_flag("-fno-omit-frame-pointer" WITH_FRAME_POINTER)
check_cxx_compiler_flag("-fsanitize-address-use-after-scope" WITH_SANITIZE_ADDRESS_USE_AFTER_SCOPE)
check_cxx_compiler_flag("-fsanitize=signed-integer-overflow" WITH_SANITIZE_SIGNED_INTEGER_OVERFLOW)
check_cxx_compiler_flag("-fsanitize=unsigned-integer-overflow" WITH_SANITIZE_UNSIGNED_INTEGER_OVERFLOW)

check_cxx_compiler_flag("-Wlogical-op" WITH_WARN_LOGICAL_OP)
check_cxx_compiler_flag("-Wstrict-null-sentinel" WITH_WARN_STRICT_NULL_SENTINEL)
check_cxx_compiler_flag("-Wimplicit-fallthrough=2" WITH_IMPLICIT_FALLTHROUGH)
check_cxx_compiler_flag("-Wnoexcept" WITH_WNOEXCEPT)


# Set compiler flags:
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pipe -pedantic -pedantic-errors") #  -fno-rtti

# TODO: Make this warning work too!
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wconversion")
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wsign-conversion")
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wswitch-enum -Wswitch-default")

# Enable all the warnings one can get
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wshadow -Wmissing-include-dirs -Wpacked -Wredundant-decls")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wnon-virtual-dtor -Woverloaded-virtual")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wold-style-cast")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wfloat-equal")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wpointer-arith -Wcast-align -Wcast-qual -Wwrite-strings")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wunused -Wunused-function -Wunused-label -Wunused-value -Wunused-variable")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Winit-self -Wdisabled-optimization")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wsign-promo -Wstrict-overflow=5")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wformat=2 -Wformat-security")

# -Werror
# -Wundef -Wmissing-declarations
# Not compatible wit GTest: -Wctor-dtor-privacy
# Not friendly to GTest: -Wmissing-format-attribute")
# Too many lies: set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Weffc++")

# New in gcc-7
if (WITH_IMPLICIT_FALLTHROUGH)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wimplicit-fallthrough=2")
endif(WITH_IMPLICIT_FALLTHROUGH)

# GCC specific
if (WITH_WNOEXCEPT)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wnoexcept")
endif (WITH_WNOEXCEPT)
if (WITH_WARN_LOGICAL_OP)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wlogical-op")
endif (WITH_WARN_LOGICAL_OP)
if (WITH_WARN_STRICT_NULL_SENTINEL)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wstrict-null-sentinel")
endif (WITH_WARN_STRICT_NULL_SENTINEL)
if (WITH_STACK_PROTECTOR)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fstack-protector-strong")
endif (WITH_STACK_PROTECTOR)


# It's better to use fortify_sources if compiler supports it
if (UNIX AND NOT APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_FORTIFY_SOURCE=2")

    # Some extra defins for libstdc++:
#    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_ASSERTIONS -D_GLIBCXX_DEBUG -D_GLIBCXX_SANITIZE_VECTOR")
endif()


# ---------------------------------
# Debug mode flags
# ---------------------------------
set(CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS_DEBUG} -O0 -ggdb3 -D _DEBUG -D DEBUG")
set(CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS_DEBUG} -fstack-protector-all")

if (WITH_FRAME_POINTER)
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fno-omit-frame-pointer")
endif (WITH_FRAME_POINTER)

if (WITH_VTABLE_VERIFY)
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fvtable-verify=std")
endif (WITH_VTABLE_VERIFY)


# ---------------------------------
# Release Optimization flags
# ---------------------------------
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Ofast -D NDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native -mtune=native")

# Extra optimizations on GCC
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" AND NOT "MINGW")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}  -fuse-ld=gold")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fuse-linker-plugin -flto-report -flto")

    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fuse-ld=gold")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=gold")
endif()


# ---------------------------------
# When sanitizers are ON
# ---------------------------------
if (STYXE_SANITIZE)
    check_cxx_compiler_flag("-fsanitize=leak" WITH_SANITIZE_LEAK)
    if (WITH_SANITIZE_LEAK)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address,undefined,leak")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address,undefined")
        message(STATUS, "Leak sanitizer not supported")
    endif(WITH_SANITIZE_LEAK)

    if (WITH_SANITIZE_ADDRESS_USE_AFTER_SCOPE)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize-address-use-after-scope")
    endif(WITH_SANITIZE_ADDRESS_USE_AFTER_SCOPE)

    if (WITH_SANITIZE_SIGNED_INTEGER_OVERFLOW)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=signed-integer-overflow")
    endif(WITH_SANITIZE_SIGNED_INTEGER_OVERFLOW)

    if (WITH_SANITIZE_UNSIGNED_INTEGER_OVERFLOW)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=unsigned-integer-overflow")
    endif(WITH_SANITIZE_UNSIGNED_INTEGER_OVERFLOW)


    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=address")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
endif()


# ---------------------------------
# Debug build with test coverage
# ---------------------------------
if (STYXE_COVERALLS)
    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} --coverage") # enabling coverage
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} --coverage")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
    else()
        message(FATAL_ERROR "Coverage can only be enabled in Debug mode")
    endif()
endif()



# ---------------------------------
# Build project dependencies
# ---------------------------------
set(STYXE_EXTERNAL_DEP_GTEST_DIR "external/gtest/googletest"
    CACHE PATH "The path to the Google Test framework.")

if (STYXE_LIBSOLACE_SUPPORT)
    if(STYXE_GTEST_SUPPORT)
        set(SOLACE_GTEST_SUPPORT OFF CACHE BOOL "Don't use dependencie's gtest" FORCE)
        message(STATUS, "Turning solace GTEST off")
    endif()

    message(STATUS, "Adding local libsolce")
    add_subdirectory(external/libsolace EXCLUDE_FROM_ALL)
    include_directories(${solace_SOURCE_DIR}/include)
endif()

include_directories(include)

add_subdirectory(src)
add_subdirectory(test EXCLUDE_FROM_ALL)
add_subdirectory(examples EXCLUDE_FROM_ALL)


# ---------------------------------
# Show build configuration status:
# ---------------------------------
message(STATUS, "BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
message(STATUS, "CXXFLAGS: ${CMAKE_CXX_FLAGS}")
message(STATUS, "STYXE_SANITIZE: ${STYXE_SANITIZE}")
message(STATUS, "STYXE_COVERALLS: ${STYXE_COVERALLS}")
message(STATUS, "STYXE_LIBSOLACE_SUPPORT: ${STYXE_LIBSOLACE_SUPPORT}")
message(STATUS, "STYXE_GTEST_SUPPORT: ${STYXE_GTEST_SUPPORT}")
