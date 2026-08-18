# Builds the AutoRemesher core as a Python extension module (autoremesher_core)
# for the Blender add-on. The Qt desktop app still builds with qmake
# (autoremesher.pro); this file intentionally covers only the headless core.
#
# Synced with upstream huxingyi/autoremesher 1.1: native quad parameterizer
# (no Geogram), meshoptimizer for dense-mesh decimation.
cmake_minimum_required(VERSION 3.18)
project(autoremesher_core LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release)
endif()

# Development.SABIModule is required for nanobind's STABLE_ABI to take
# effect (otherwise it silently builds a version-specific module).
find_package(Python 3.11 COMPONENTS Interpreter Development.Module
    Development.SABIModule REQUIRED)

# --- nanobind (fetched by pip as a build requirement; located via its CMake config)
execute_process(
    COMMAND "${Python_EXECUTABLE}" -m nanobind --cmake_dir
    OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE nanobind_ROOT)
find_package(nanobind CONFIG REQUIRED)

# --- oneTBB 2023.1.0 via FetchContent (GCC 14 / CMake 4 compatible).
# The legacy TBB 2017 in thirdparty/tbb fails to build with GCC 14's
# -Wchanges-meaning-as-error; use the official modern release instead.
include(FetchContent)
set(TBB_TEST OFF CACHE BOOL "" FORCE)
set(TBB_STRICT OFF CACHE BOOL "" FORCE)
set(TBB_INSTALL OFF CACHE BOOL "" FORCE)
set(TBB_ENABLE_IPO OFF CACHE BOOL "" FORCE)
set(TBBMALLOC_BUILD OFF CACHE BOOL "" FORCE)
# Build only the static variant so the wheel is self-contained (no .so/.dll to bundle).
set(BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
FetchContent_Declare(
    TBB
    GIT_REPOSITORY https://github.com/uxlfoundation/oneTBB.git
    GIT_TAG        v2023.1.0
    GIT_SHALLOW    TRUE
)
FetchContent_MakeAvailable(TBB)

set(CORE_SOURCES
    src/AutoRemesher/autoremesher.cpp
    src/AutoRemesher/isotropicremesher.cpp
    src/AutoRemesher/parameterizer.cpp
    src/AutoRemesher/quadextractor.cpp
    src/AutoRemesher/quadparameterizer.cpp
    src/AutoRemesher/surfacemesh.cpp
    src/AutoRemesher/singularitysimplifier.cpp
    src/AutoRemesher/constrainedleastsquares.cpp
    src/AutoRemesher/mixedintegerleastsquares.cpp
    src/AutoRemesher/framefield.cpp
    src/AutoRemesher/positionkey.cpp
    src/AutoRemesher/meshseparator.cpp
    thirdparty/isotropicremesher/isotropicremesher.cpp
    thirdparty/isotropicremesher/isotropichalfedgemesh.cpp
    thirdparty/isotropicremesher/axisalignedboundingboxtree.cpp
    thirdparty/meshoptimizer/src/simplifier.cpp
    thirdparty/meshoptimizer/src/indexgenerator.cpp
)

add_library(autoremesher_static STATIC ${CORE_SOURCES})
target_compile_features(autoremesher_static PUBLIC cxx_std_14)
target_include_directories(autoremesher_static PUBLIC
    include
    thirdparty/eigen
    thirdparty/isotropicremesher
    thirdparty/meshoptimizer/src
)
target_compile_definitions(autoremesher_static PUBLIC
    _USE_MATH_DEFINES NOMINMAX EIGEN_MPL2_ONLY)
target_link_libraries(autoremesher_static PUBLIC TBB::tbb)
set_target_properties(autoremesher_static PROPERTIES POSITION_INDEPENDENT_CODE ON)

if(APPLE)
    target_compile_definitions(autoremesher_static PUBLIC AUTO_REMESHER_USE_ACCELERATE=1)
    target_link_libraries(autoremesher_static PUBLIC "-framework Accelerate")
endif()

if(WIN32)
    # Static oneTBB: disable the '#pragma comment(lib, "tbb12")' auto-link.
    # oneTBB 2023 links TBB::tbb which already suppresses auto-link
    target_compile_options(autoremesher_static PRIVATE /bigobj)
    target_link_libraries(autoremesher_static PUBLIC advapi32 shell32 user32)
else()
    target_compile_options(autoremesher_static PRIVATE -O3 -funroll-loops)
    find_package(Threads REQUIRED)
    target_link_libraries(autoremesher_static PUBLIC Threads::Threads)
endif()

nanobind_add_module(autoremesher_core STABLE_ABI src/python/binding.cpp)
target_link_libraries(autoremesher_core PRIVATE autoremesher_static)

install(TARGETS autoremesher_core LIBRARY DESTINATION .)
