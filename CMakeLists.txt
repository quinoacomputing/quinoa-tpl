################################################################################
#
# \file      CMakeLists.txt
# \copyright 2012-2015 J. Bakosi,
#            2016-2018 Los Alamos National Security, LLC.,
#            2019 Triad National Security, LLC.
#            All rights reserved. See the LICENSE file for details.
# \brief     Build quinoa third-party Libraries
#
################################################################################

cmake_minimum_required(VERSION 3.1.0 FATAL_ERROR)

project(TPL C CXX Fortran)
include(ExternalProject)

# Quinoa cmake modules directory
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Disallow in-source builds
include(DisallowInSourceBuilds)
disallow_in_source_builds()

# Set CMAKE_BUILD_TYPE
include(BuildType)

# Detect operating system type and version
include(DetectOS)
detect_os()
message(STATUS "Detected OS: ${HOST_OS}")

# Set BUILD_SHARED_LIBS
set(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared libraries. Possible value: ON")
message(STATUS "BUILD_SHARED_LIBS: ${BUILD_SHARED_LIBS}")

# Include support for multiarch path names
include(GNUInstallDirs)

# Set compiler id
string(TOLOWER "${CMAKE_CXX_COMPILER_ID}" COMP)

# GPU device compiler selection. These are declared here, ahead of the install
# prefix, because a HIP build must not share an install tree with a CUDA or
# CPU-only build: Kokkos permits exactly one GPU architecture per build, and
# target_architecture() reports x86_64 for every x86 host regardless of which
# accelerator is attached.
# NVCC path users can pass on the cmd line
set(NVCC_WRAPPER "" CACHE FILEPATH "Path to Kokkos nvcc_wrapper")
# HIP device compiler (hipcc or amdclang++) users can pass on the cmd line
set(HIP_COMPILER "" CACHE FILEPATH "Path to hipcc or amdclang++")
# User-configurable Kokkos arch token (empty = none)
# Token spelling depends on the Kokkos version bundled with the pinned
# Trilinos: Kokkos < 4.2 spells the AMD archs VEGA908/VEGA90A, 4.2 and later
# spell them AMD_GFX908/AMD_GFX90A.
set(KOKKOS_ARCH "" CACHE STRING "Kokkos architecture token, e.g. VOLTA70, TURING75, AMPERE80, HOPPER90, VEGA908, VEGA90A, AMD_GFX90A, AMD_GFX942")

if (NVCC_WRAPPER AND HIP_COMPILER)
  message(FATAL_ERROR
          "NVCC_WRAPPER and HIP_COMPILER are mutually exclusive: Kokkos can "
          "enable only one GPU backend per build.")
endif()

# Kokkos HIP has no architecture autodetection fallback we can rely on here,
# and an unsuffixed install tree would let a gfx908 and a gfx90a build
# overwrite each other, so require the token explicitly.
set(GPU_INSTALL_SUFFIX "")
if (HIP_COMPILER)
  if (NOT KOKKOS_ARCH)
    message(FATAL_ERROR
            "HIP_COMPILER requires KOKKOS_ARCH, e.g. -DKOKKOS_ARCH=AMD_GFX90A "
            "for MI200/MI250 or -DKOKKOS_ARCH=AMD_GFX908 for MI100.")
  endif()
  string(TOLOWER "${KOKKOS_ARCH}" _KOKKOS_ARCH_LOWER)
  string(REGEX REPLACE "^amd_" "" _KOKKOS_ARCH_LOWER "${_KOKKOS_ARCH_LOWER}")
  set(GPU_INSTALL_SUFFIX "-${_KOKKOS_ARCH_LOWER}")
endif()

# Query target architecture
include(TargetArch)
target_architecture(ARCH)
message(STATUS "Detected target architecture: ${ARCH}")

# Set install directory based on compiler ID if was not given on command line
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX
      "${PROJECT_SOURCE_DIR}/install/${COMP}-${ARCH}${GPU_INSTALL_SUFFIX}"
      CACHE STRING "Third-party install prefix." FORCE)
  message(STATUS "CMAKE_INSTALL_PREFIX not specified, setting to '${PROJECT_SOURCE_DIR}/install/${COMP}-${ARCH}${GPU_INSTALL_SUFFIX}'")
else()
  message(STATUS "CMAKE_INSTALL_PREFIX: " ${CMAKE_INSTALL_PREFIX})
endif()

include(ProcessorCount)
ProcessorCount(PROCESSOR_COUNT)
if(PROCESSOR_COUNT EQUAL 0)     # if ncpus could not be determined, assign 1
  set(PROCESSOR_COUNT "1")
endif()
message(STATUS "Found ${PROCESSOR_COUNT} PEs")

# Convert system name to lower case
string(TOLOWER ${CMAKE_SYSTEM_NAME} SYSTEM_NAME)

# Set which libraries are built/searched by default
set(ENABLE_CHARM ON CACHE BOOL "Enable Charm++")
set(ENABLE_NETCDF ON CACHE BOOL "Enable NetCDF")
set(ENABLE_BOOST ON CACHE BOOL "Enable Boost")
set(ENABLE_PUGIXML ON CACHE BOOL "Enable PugiXML")
set(ENABLE_PEGTL ON CACHE BOOL "Enable PEGTL")
set(ENABLE_LAPACK ON CACHE BOOL "Enable LAPACK")
set(ENABLE_TRILINOS ON CACHE BOOL "Enable Trilinos")
set(ENABLE_TUT ON CACHE BOOL "Enable TUT")
set(ENABLE_NUMDIFF ON CACHE BOOL "Enable NumDiff")
set(ENABLE_BACKWARDCPP ON CACHE BOOL "Enable BackwardCpp")
set(ENABLE_HIGHWAYHASH ON CACHE BOOL "Enable HighwayHash")
set(ENABLE_BRIGAND ON CACHE BOOL "Enable Brigand")
set(ENABLE_SOL2 ON CACHE BOOL "Enable Sol")
set(ENABLE_DOXYGEN ON CACHE BOOL "Enable Doxygen")
set(ENABLE_MCSS ON CACHE BOOL "Enable m.css")
## Mutation is triggered manually by specifying -DENABLE_MUTATIONPP=ON at config
#set(ENABLE_EIGEN ON CACHE BOOL "Enable Eigen")
#set(ENABLE_MUTATIONPP ON CACHE BOOL "Enable Mutation++")

# Set default requested executable targets
set(REQUESTED_EXECUTABLES unittest inciter meshconv)

# If a *_ONLY variable is set, user only wants specific executable(s)
if (UNITTEST_ONLY OR INCITER_ONLY OR MESHCONV_ONLY)

  # Turn everything off first
  set(ENABLE_CHARM OFF)
  set(ENABLE_NETCDF OFF)
  set(ENABLE_BOOST OFF)
  set(ENABLE_PUGIXML OFF)
  set(ENABLE_PEGTL OFF)
  set(ENABLE_LAPACK OFF)
  set(ENABLE_TRILINOS OFF)
  set(ENABLE_TUT OFF)
  set(ENABLE_NUMDIFF OFF)
  set(ENABLE_BACKWARDCPP OFF)
  set(ENABLE_HIGHWAYHASH OFF)
  set(ENABLE_BRIGAND OFF)
  set(ENABLE_SOL2 OFF)
  set(ENABLE_DOXYGEN OFF)
  set(ENABLE_MCSS OFF)
  #set(ENABLE_EIGEN OFF)
  #set(ENABLE_MUTATIONPP OFF)
  set(REQUESTED_EXECUTABLES)

  # Turn on prerequisites for executable targets configured, these blocks
  # should be consistent with the bottom of TPLs.cmake.

  if (UNITTEST_ONLY)
    list(APPEND REQUESTED_EXECUTABLES unittest)
    set(ENABLE_CHARM ON)
    set(ENABLE_PUGIXML ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_HIGHWAYHASH ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_TUT ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_LAPACK ON)
    set(ENABLE_SOL2 ON)
  endif()

  if (INCITER_ONLY)
    list(APPEND REQUESTED_EXECUTABLES inciter)
    set(ENABLE_CHARM ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_HIGHWAYHASH ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_LAPACK ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_NUMDIFF ON)
    set(ENABLE_SOL2 ON)
    #set(ENABLE_MUTATIONPP ON)
  endif()

  if (MESHCONV_ONLY)
    list(APPEND REQUESTED_EXECUTABLES meshconv)
    set(ENABLE_CHARM ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_PUGIXML ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_HIGHWAYHASH ON)
    set(ENABLE_NUMDIFF ON)
    set(ENABLE_SOL2 ON)
  endif()

endif()

# Charm++
if (ENABLE_CHARM)
  find_package(Charm)
  if(CHARM_FOUND)
    set(charm "")
  else()
    set(charm "charm")
  endif()
endif()

# If we are to build our own Charm++, setup prerequisites for its build command
if (charm)

  # Match Charm++ build architecture for system architecture detected
  if(ARCH MATCHES "x86_64")

    set(CHARM_ARCH "-${ARCH}")

  elseif(ARCH MATCHES "ppc64")

    set(SYSTEM_NAME "")
    set(CHARM_ARCH "bluegeneq")

  elseif(ARCH MATCHES "i386")

    if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
      set(CHARM_ARCH "")
    elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
      set(CHARM_ARCH "-x86")
    endif()

  elseif(ARCH MATCHES "arm64")
    set(CHARM_ARCH "-arm8")

  else()

    message(FATAL "Selecting a Charm++ architecture for ${ARCH} is not supported")

  endif()

  if(ENABLE_AMPI)
    message(STATUS "Charm++'s Adaptive MPI (AMPI) enabled")
  endif()

  # Set Charm++ architecture
  if (NOT DEFINED ENV{CRAYPE_VERSION} AND NOT DEFINED ENV{PE_ENV})  # not Cray

    if(NOT ENABLE_AMPI)
      set(CHARM_COMM_SUBSYSTEM "mpi")
    else()

      # AMPI status: (add opts via CHARM_EXTRA_ARGS)
      # ------------
      #set(CHARM_COMM_SUBSYSTEM "netlrts") # opts tried: udp, tcp, to try: smp, pxshm
      # - udp: unittest works but slow; inciter, walker, rngtest immaturely abort
      # - tcp: unittest works; inciter, walker, rngtest immaturely abort
      #set(CHARM_COMM_SUBSYSTEM "ofi")
      # - hdf5-ampi: mpiexec: Error: unknown option "-pmi_args"
      set(CHARM_COMM_SUBSYSTEM "multicore")
      # - works, unittest runs MPI suite, but fails randomlym others immaturely abort
      #set(CHARM_COMM_SUBSYSTEM "verbs")
      #  hdf5-ampi: charmun: error attaching to node '127.0.0.1'
      #set(CHARM_COMM_SUBSYSTEM "sim")  # not attempted
      #set(CHARM_COMM_SUBSYSTEM "uth")  # not attempted

    endif()

  else()  # Cray

    if(NOT ENABLE_AMPI)
      set(CHARM_COMM_SUBSYSTEM "mpi")
    else()
      set(CHARM_COMM_SUBSYSTEM "gni")   # never tested
    endif()

  endif()

  set(CHARM_ARCHITECTURE "${CHARM_COMM_SUBSYSTEM}-${SYSTEM_NAME}${CHARM_ARCH}")

  message(STATUS "Charm++ architecture/version: ${CHARM_ARCHITECTURE}")

  if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Cray")
    set(CHARM_COMPILER "craycc")
  elseif (ENABLE_AMPI AND "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(CHARM_COMPILER "clang")
  elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel" AND
          "${CMAKE_CXX_COMPILER}" MATCHES "mpiicpc")
      set(CHARM_COMPILER "icc") # intel mpi + intel compiler
  else()
    set(CHARM_COMPILER "")  # use mpi wrappers by default
  endif()

  if (CHARM_COMPILER)
    message(STATUS "Charm++ compiler: ${CHARM_COMPILER}")
  endif()

endif(charm)

# Compilers
# =========
# There are two sets of compilers, given by the following CMake variables:
#
# 1. The underlying compiler suite
#    UNDERLYING_C_COMPILER
#    UNDERLYING_CXX_COMPILER
#    UNDERLYING_Fortran_COMPILER
#
# 2. The MPI wrappers (of the underlying compilers)
#    MPI_C_COMPILER
#    MPI_CXX_COMPILER
#    MPI_Fortran_COMPILER

# Find MPI and get its underlying compilers. If a traditional MPI (e.g.,
# OpenMPI) is used, the get_mpi_compilers cmake function uses the built-in
# CMake functionality to find the MPI wrappers and based on the wrappers
# detects the underyling compilers. If Charm++'s Adaptive MPI (AMPI) is used,
# the Charm++ TPL build builds Charm++ as well as AMPI and we use the AMPI
# wrappers to build MPI libraries.
set(ampi "${charm}")  # used as a dependency for MPI libs
set(ENABLE_AMPI OFF CACHE BOOL "Enable AMPI")
if(NOT ENABLE_AMPI)
  include(MPICompilers)
  get_mpi_compilers()
  set(CHARM_BUILD_TARGET "LIBS")
  set(ampi "")
  set(CHARM_RUN "${CMAKE_INSTALL_PREFIX}/charm/bin/charmrun")
else()
  set(MPI_C_COMPILER "${CMAKE_INSTALL_PREFIX}/charm/bin/ampicc")
  set(MPI_CXX_COMPILER "${CMAKE_INSTALL_PREFIX}/charm/bin/ampicxx")
  set(MPI_Fortran_COMPILER "${CMAKE_INSTALL_PREFIX}/charm/bin/ampif90")
  set(AMPI_RUN "${CMAKE_INSTALL_PREFIX}/charm/bin/ampirun")
  set(CHARM_RUN "${CMAKE_INSTALL_PREFIX}/charm/bin/charmrun")
  set(UNDERLYING_C_COMPILER "${CMAKE_C_COMPILER}")
  set(UNDERLYING_CXX_COMPILER "${CMAKE_CXX_COMPILER}")
  set(UNDERLYING_Fortran_COMPILER "${CMAKE_Fortran_COMPILER}")
  set(CHARM_BUILD_TARGET "AMPI")
  MESSAGE(STATUS "AMPIrun: ${AMPI_RUN}")
endif()

MESSAGE(STATUS "MPI C compiler: " ${MPI_C_COMPILER})
MESSAGE(STATUS "MPI C++ compiler: " ${MPI_CXX_COMPILER})
MESSAGE(STATUS "MPI Fortran compiler: " ${MPI_Fortran_COMPILER})
MESSAGE(STATUS "Underlying C compiler: " ${UNDERLYING_C_COMPILER})
MESSAGE(STATUS "Underlying C++ compiler: " ${UNDERLYING_CXX_COMPILER})
MESSAGE(STATUS "Underlying Fortran compiler: " ${UNDERLYING_Fortran_COMPILER})

MESSAGE(STATUS "Charmrun: ${CHARM_RUN}")
MESSAGE(STATUS "Charm build target: ${CHARM_BUILD_TARGET}")

if (CHARM_EXTRA_ARGS)
  MESSAGE(STATUS "Charm++ extra build args (CHARM_EXTRA_ARGS): ${CHARM_EXTRA_ARGS}")
endif()

# Compiler-specific settings
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")  # clang-specific settings

  # Include function that allows to set standard C++ library
  include(libstdcxx)

  # Offer switch between libc++ and libstdc++, using libc++ as default
  if (LIBCXX_FOUND)
    set_libstdcpp_vs_libcpp(CXXFLAGS "libc++")
  endif()

  # Don't error out on functions not returning anything even when supposed to,
  # required only by TestU01
  set(CFLAGS "${CFLAGS} -Wno-error=return-type")

  # Detect compiler version
  include(DetectCompilerVersion)

elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")  # gcc-specific settings

  # Gnu will use libstdc++ by default. We do not offer libc++.

  # Detect compiler version
  include(DetectCompilerVersion)

elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")  # intel-specific settings

  # Include function that allows to set standard C++ library
  include(libstdcxx)

  # Offer switch between libc++ and libstdc++, using libstdc++ as default
  if (LIBCXX_FOUND)
    set_libstdcpp_vs_libcpp(CXXFLAGS "libstdc++")
  endif()

elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "PGI")  # pgi-specific settings

  # Some configure scripts don't like pgi's preprocessor so use gnu's
  set(PGCPP "CPP=cpp")

  # PGI C compiler flags specific to netcdf
  set(CFLAGS "${CFLAGS} -fPIC")
  set(NETCDF_UTILS "--disable-utilities")

elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Cray")  # cray-specific settings

else()

  message(WARNING "Compiler '${CMAKE_CXX_COMPILER_ID}' could not be identified.")

endif()

# Mac OS X specifics
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
  set(MACOSX_RPATH "-DCMAKE_MACOSX_RPATH:BOOL=ON")
endif()

# Attempt to find some TPLs on the system

# NetCDF
# A system NetCDF short-circuits the bundled build below, which leaves Trilinos
# compiling SEACAS against the system netcdf_meta.h. When that NetCDF was built
# with HDF5, exodus (exodusII_int.h includes netcdf_meta.h, ex_utils.c calls
# H5get_libversion under NC_HAS_HDF5 with an extern declaration of its own)
# ends up referencing HDF5 without anything linking it. GNU ld tolerates the
# resulting under-linked libexodus; ld.lld rejects it under
# --no-allow-shlib-undefined. Set FORCE_BUILD_NETCDF to ignore the system copy
# and build the bundled NetCDF without netcdf-4, so NC_HAS_HDF5 is 0 and the
# reference is never generated.
set(FORCE_BUILD_NETCDF OFF CACHE BOOL
    "Ignore any system NetCDF and build the bundled one")
if (ENABLE_NETCDF)
  if(ENABLE_AMPI)
    set(NETCDF_ROOT ${CMAKE_INSTALL_PREFIX})
  endif()
  if (FORCE_BUILD_NETCDF)
    message(STATUS "FORCE_BUILD_NETCDF: ignoring any system NetCDF")
    set(NETCDF_FOUND FALSE)
  else()
    find_package(NetCDF)
  endif()
  if ((ENABLE_AMPI AND NETCDF_FOUND AND
       NETCDF_INCLUDES MATCHES ${CMAKE_INSTALL_PREFIX}) OR
      (NOT ENABLE_AMPI AND NETCDF_FOUND))
    set(netcdf "")
    set(netcdf_include_dirs "${NETCDF_INCLUDES}")
    set(netcdf_library_dirs "${NETCDF_LIBRARY_DIRS}")
  else()
    set(netcdf "netcdf")
    message(STATUS "NetCDF either not found or AMPI is enabled: will build")
    set(netcdf_include_dirs "${CMAKE_INSTALL_PREFIX}/include")
    set(netcdf_library_dirs "${CMAKE_INSTALL_PREFIX}/lib")
  endif()
endif()

# Boost
if (ENABLE_BOOST)
  find_package(Boost 1.74.0)
  if(Boost_FOUND)
    set(boost "")
  else()
    set(boost "boost")
  endif()
endif()

# PEGTL
if (ENABLE_PEGTL)
  find_package(PEGTL 2.0.0)
  if(PEGTL_FOUND)
    set(pegtl "")
  else()
    set(pegtl "pegtl")
  endif()
endif()

# BLAS and LAPACK
if (ENABLE_LAPACK)
  set(lapack "lapack")
  find_package(BLAS)
  find_package(CBLAS)
  if(CBLAS_FOUND)
    set(cblas "")
  endif()
  find_package(LAPACKE)
  if(LAPACKE_FOUND)
    set(lapack "")
  endif()
endif()

# Trilinos
if (ENABLE_TRILINOS)
  find_package(Zoltan2 QUIET)
  find_package(SEACASExodus QUIET)
  find_package(Exodiff QUIET)

  if ((ENABLE_AMPI AND Zoltan2_FOUND AND SEACASExodus_FOUND AND Exodiff_FOUND AND
       Zoltan2_INCLUDE_DIRS MATCHES ${CMAKE_INSTALL_PREFIX}) OR
      (NOT ENABLE_AMPI AND Zoltan2_FOUND AND SEACASExodus_FOUND AND Exodiff_FOUND))
    set(trilinos "")
  else()
    set(trilinos "trilinos")
    message(STATUS "Zoltan2, SEACASExodus, and Exodiff either not found or AMPI is enabled: will build")
  endif()
endif()

# Template Unit Test library
if (ENABLE_TUT)
  find_package(TUT)
  if(TUT_FOUND)
    set(tut "")
  else()
    set(tut "tut")
  endif()
endif()

# Zlib, only for static link and only on ppc64 (we don't build it)
if(ARCH MATCHES "ppc64")
 find_package(ZLIB REQUIRED)
endif()

# NumDiff
if (ENABLE_NUMDIFF)
  find_package(NumDiff)
  if(NumDiff_FOUND)
    set(numdiff "")
  else()
    set(numdiff "numdiff")
  endif()
endif()

# Backward-cpp
if (ENABLE_BACKWARDCPP)
  find_package(BackwardCpp)
  if(BACKWARDCPP_FOUND)
    set(backwardcpp "")
  else()
    set(backwardcpp "backwardcpp")
  endif()
endif()

# Highwayhash
if (ENABLE_HIGHWAYHASH)
  find_package(HighwayHash)
  if(HighwayHash_FOUND)
    set(highwayhash "")
  else()
    set(highwayhash "highwayhash")
  endif()
endif()

# Brigand
if (ENABLE_BRIGAND)
  find_package(Brigand)
  if(Brigand_FOUND)
    set(brigand "")
  else()
    set(brigand "brigand")
  endif()
endif()

# Sol2
if (ENABLE_SOL2)
  find_package(Lua)
  find_package(Sol2)
  if(SOL2_FOUND)
    set(sol2 "")
  else()
    set(sol2 "sol2")
  endif()
endif()

# Doxygen
if (ENABLE_DOXYGEN)
  find_package(Doxygen 1.8.15)
  if(DOXYGEN_FOUND)
    set(doxygen "")
  else()
    find_package(FLEX)
    find_package(BISON)
    if (FLEX_FOUND AND BISON_FOUND)
      set(doxygen "doxygen")
    endif()
  endif()
endif()

# m.css
if (ENABLE_MCSS)
  # Find Python: required for m.css
  find_package(PythonInterp 3.6)
  find_package(MCSS)
  if(MCSS_FOUND)
    set(mcss "")
  else()
    set(mcss "mcss")
  endif()
endif()

# mutationpp
if (ENABLE_MUTATIONPP)
  # mutationpp depends on Eigen
  set(ENABLE_EIGEN ON)
  set(ENABLE_PUGIXML ON)
  set(eigen "eigen")
  set(mutationpp "mutationpp")
endif()

# Pugixml
if (ENABLE_PUGIXML)
  find_package(Pugixml)
  if(PUGIXML_FOUND)
    set(pugixml "")
  else()
    set(pugixml "pugixml")
  endif()
endif()

# Get compiler flags (CFLAGS, CXXFLAGS, FFLAGS) independent of CMAKE_BUILD_TYPE
# and echo flags that will be passed to all TPL builds
include(get_compiler_flags)
get_compiler_flags()

set(tpls2build ${charm} ${netcdf} ${boost} ${pugixml} ${pegtl}
${lapack} ${trilinos} ${tut} ${numdiff} ${backwardcpp} ${highwayhash}
${brigand} ${sol2} ${doxygen} ${mcss} ${mutationpp} ${eigen})

list(LENGTH tpls2build ntpl)

# Output TPLs to be built
if (tpls2build)
  list(SORT tpls2build)
  message(STATUS "Will build the following ${ntpl} TPLs for executable(s) ${REQUESTED_EXECUTABLES}: ${tpls2build}")
else()
  message(STATUS "All TPLs found. No need to build.")
endif()

# Define fine-grained targets
set_property(DIRECTORY PROPERTY EP_STEP_TARGETS
             download patch configure build install test)

#### Charm++ ###################################################################
# http://charm.cs.illinois.edu
set(CHARM_SHARED "--build-shared")
# Enable "--with-production" for Charm++ build for Release builds
if(CMAKE_BUILD_TYPE MATCHES "REL")      # Release | RelWithDebInfo | MinSizeRel
  set(CHARM_RELEASE_BUILD "--with-production")
endif()
if (HOST_OS MATCHES "alpine")
  # To avoid requiring rpc.h on Alpine Linux using musl-libc, see also
  # https://lists.cs.illinois.edu/lists/arc/charm/2016-04/msg00002.html
  set(CHARM_ALPINE "--disable-charmdebug")
endif()
if (charm)
  set(CHARM_CXXFLAGS "${CXXFLAGS}")
  string(REPLACE " " ";" CHARM_CXXFLAGS "${CHARM_CXXFLAGS}")
  ExternalProject_Add(
    charm
    PREFIX charm
    # Move unpacked tarball to install dir and build there
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/charm
                     ${CMAKE_INSTALL_PREFIX}/charm
    # Will apply patch (if any) in SOURCE_DIR which is the same as install dir
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}/charm
    # Build in install dir and leave stuff there - no install step
    BINARY_DIR ${CMAKE_INSTALL_PREFIX}/charm
    PATCH_COMMAND
      patch -p1 -t -i ${PROJECT_SOURCE_DIR}/cmake/migratable_charmcollide.patch
    # Build Charm++
    BUILD_COMMAND ${CMAKE_INSTALL_PREFIX}/charm/buildold
                  ${CHARM_BUILD_TARGET}
                  ${CHARM_ARCHITECTURE}
                  ${CHARM_EXTRA_ARGS}
                  ${CHARM_COMPILER}
                  ${CHARM_PTHREADS}
                  ${CHARM_SHARED}
                  ${CHARM_RELEASE_BUILD}
                  ${CHARM_ALPINE}
                  -j${PROCESSOR_COUNT}
                  ${CHARM_CXXFLAGS}
    CONFIGURE_COMMAND ""
    INSTALL_COMMAND ""
    LOG_CONFIGURE 1
    LOG_BUILD 1
  )
endif()

#### NETCDF ####################################################################
## http://www.unidata.ucar.edu/downloads/netcdf/index.jsp
if (netcdf)
  if(NOT ENABLE_AMPI)
    set(NETCDF_SHARED "--enable-shared")
  else()
    set(NETCDF_SHARED "--enable-static;--disable-shared;--with-pic")
    if(ARCH MATCHES "ppc64")
      set(NETCDF_ZLIB "LIBS=${ZLIB_LIBRARIES}")
      set(NETCDF_DISABLE_TESTS "--disable-testsets")
    endif()
  endif()
  ExternalProject_Add(
    netcdf
    DEPENDS ${ampi}
    PREFIX netcdf
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/netcdf
                     ${PROJECT_BINARY_DIR}/netcdf
    CONFIGURE_COMMAND ${PROJECT_BINARY_DIR}/netcdf/configure
                      CC=${MPI_C_COMPILER}
                      CFLAGS=${CFLAGS}
                      ${NETCDF_UTILS}
                      ${PGCPP}
                      --disable-netcdf4
                      --disable-examples
                      --disable-fsync
                      --disable-dap
                      ${NETCDF_SHARED}
                      ${NETCDF_DISABLE_TESTS}
                      ${NETCDF_ZLIB}
                      --prefix=${CMAKE_INSTALL_PREFIX}
    BUILD_IN_SOURCE 1
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### lapack ####################################################################
# http://www.netlib.org/lapack
if(NOT ENABLE_AMPI)
  set(LAPACK_BUILD_SHARED_LIBS on)
elseif()
  set(LAPACK_BUILD_SHARED_LIBS off)
endif()
if (lapack)
  ExternalProject_Add(
    lapack
    PREFIX lapack
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/lapack
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D CMAKE_Fortran_COMPILER:PATH=${UNDERLYING_Fortran_COMPILER}
               -D CMAKE_Fortran_FLAGS:STRING=${FFLAGS}
               -D BUILD_TESTING:BOOL=OFF
               -D BUILD_SHARED_LIBS:BOOL=${LAPACK_BUILD_SHARED_LIBS}
               -D LAPACKE:BOOL=ON
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
               ${MACOSX_RPATH}
    DOWNLOAD_COMMAND ""
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

##### Trilinos #################################################################
## http://trilinos.sandia.gov

# NVCC_WRAPPER, HIP_COMPILER and KOKKOS_ARCH are declared near the top of this
# file, because the install prefix depends on them.

# Build the device-compiler env launcher only if a device compiler is specified.
# Both MPI wrappers are overridden so this works with OpenMPI and MPICH alike.
set(DEVICE_CXX_ENV)  # empty by default
if (NVCC_WRAPPER)
  message(STATUS "Will build Trilinos/Kokkos with CUDA")
  message(STATUS "NVCC wrapper: ${NVCC_WRAPPER}")
  set(DEVICE_CXX_ENV ${CMAKE_COMMAND} -E env
               OMPI_CXX=${NVCC_WRAPPER}
               MPICH_CXX=${NVCC_WRAPPER})
endif()
if (HIP_COMPILER)
  message(STATUS "Will build Trilinos/Kokkos with HIP")
  message(STATUS "HIP compiler: ${HIP_COMPILER}")
  set(DEVICE_CXX_ENV ${CMAKE_COMMAND} -E env
               OMPI_CXX=${HIP_COMPILER}
               MPICH_CXX=${HIP_COMPILER})
endif()

if(NOT ENABLE_AMPI)
  set(TRILINOS_SHARED "-DBUILD_SHARED_LIBS=ON")
else()
  set(TRILINOS_SHARED "-DBUILD_SHARED_LIBS=OFF;-DTPL_FIND_SHARED_LIBS:BOOL=OFF;-DTrilinos_LINK_SEARCH_START_STATIC:BOOL=ON;-DCMAKE_SKIP_INSTALL_RPATH=TRUE")
endif()
if (HOST_OS MATCHES "alpine")
  set(TRILINOS_ALPINE "-DBLAS_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib;-DLAPACK_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib;-DTrilinos_CXX11_FLAGS=-std=c++11")
endif()
if(BLAS_FOUND)
  set(TRILINOS_BLAS -DTPL_BLAS_LIBRARIES=${BLAS_LIBRARIES})
else()
  set(TRILINOS_BLAS -DBLAS_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib)
endif()
if(LAPACKE_FOUND)
  set(TRILINOS_LAPACK -DTPL_LAPACK_LIBRARIES=${LAPACKE_LIBRARIES})
else()
  set(TRILINOS_LAPACK -DLAPACK_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib)
endif()
if(trilinos)
  # find_package(MPI) has been observed to report the C wrapper (mpicc) as the
  # C++ wrapper on some installations, which silently defeats the OMPI_CXX /
  # MPICH_CXX override below: mpicc honours OMPI_CC, so Trilinos gets built
  # with the host compiler and the first C++ compile dies on a device flag it
  # has never heard of. Locate a genuine C++ wrapper next to the C one instead
  # of trusting the detection.
  get_filename_component(MPI_BIN_DIR "${MPI_C_COMPILER}" DIRECTORY)
  find_program(MPI_CXX_WRAPPER
               NAMES mpicxx mpic++ mpiCC mpiicpc
               HINTS "${MPI_BIN_DIR}"
               DOC "MPI C++ wrapper used to build Trilinos")

  if (MPI_CXX_WRAPPER)
    if (NOT "${MPI_CXX_WRAPPER}" STREQUAL "${MPI_CXX_COMPILER}")
      message(WARNING
              "find_package(MPI) reported '${MPI_CXX_COMPILER}' as the MPI C++ "
              "compiler; building Trilinos with '${MPI_CXX_WRAPPER}' instead.")
    endif()
    set(TRILINOS_CXX_COMPILER "${MPI_CXX_WRAPPER}")
  else()
    if (NVCC_WRAPPER OR HIP_COMPILER)
      message(FATAL_ERROR
              "No MPI C++ wrapper (mpicxx/mpic++/mpiCC) found next to "
              "'${MPI_C_COMPILER}'. A GPU build routes the device compiler "
              "through the C++ wrapper and cannot proceed without one.")
    endif()
    set(TRILINOS_CXX_COMPILER "${MPI_CXX_COMPILER}")
  endif()

  # Verify the wrapper actually routes the device compiler, so that a mismatch
  # fails here in seconds rather than on the first C++ compile of Trilinos.
  set(MPI_WRAPPER_CHECK ON CACHE BOOL
      "Verify the MPI C++ wrapper honours OMPI_CXX/MPICH_CXX in GPU builds")
  if (MPI_WRAPPER_CHECK AND (NVCC_WRAPPER OR HIP_COMPILER))
    if (NVCC_WRAPPER)
      set(DEVICE_CXX "${NVCC_WRAPPER}")
    else()
      set(DEVICE_CXX "${HIP_COMPILER}")
    endif()
    get_filename_component(DEVICE_CXX_NAME "${DEVICE_CXX}" NAME)
    execute_process(
      COMMAND ${CMAKE_COMMAND} -E env
              OMPI_CXX=${DEVICE_CXX} MPICH_CXX=${DEVICE_CXX}
              ${TRILINOS_CXX_COMPILER} -show
      OUTPUT_VARIABLE  WRAPPER_SHOW
      ERROR_VARIABLE   WRAPPER_SHOW_ERR
      RESULT_VARIABLE  WRAPPER_SHOW_RESULT
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    if (NOT WRAPPER_SHOW_RESULT EQUAL 0)
      message(FATAL_ERROR
              "'${TRILINOS_CXX_COMPILER} -show' failed: ${WRAPPER_SHOW_ERR}")
    endif()
    if (NOT "${WRAPPER_SHOW}" MATCHES "${DEVICE_CXX_NAME}")
      message(FATAL_ERROR
              "'${TRILINOS_CXX_COMPILER}' does not honour OMPI_CXX/MPICH_CXX: "
              "with the override in place, '-show' reports '${WRAPPER_SHOW}', "
              "which does not mention '${DEVICE_CXX_NAME}'. Trilinos would be "
              "built with the host compiler and would fail on the first device "
              "flag. Re-run with -DMPI_WRAPPER_CHECK=OFF to bypass this check.")
    endif()
    message(STATUS "MPI C++ wrapper routes to: ${WRAPPER_SHOW}")
  endif()

  set(TRILINOS_ARGS
               -D Trilinos_ENABLE_ALL_PACKAGES:BOOL=OFF
               -D TPL_ENABLE_MPI:BOOL=ON
               -D Trilinos_ENABLE_ALL_OPTIONAL_PACKAGES=OFF
               -D Trilinos_ENABLE_Zoltan2:BOOL=ON
               -D Zoltan2_ENABLE_Experimental:BOOL=ON
               -D Trilinos_ENABLE_SECONDARY_STABLE_CODE:BOOL=ON
               -D Trilinos_ENABLE_SEACASExodus:BOOL=ON
               -D Trilinos_ENABLE_SEACASExodiff:BOOL=ON
               -D Trilinos_ENABLE_Fortran=OFF
               -D Trilinos_ENABLE_DEBUG=OFF
               -D Trilinos_GENERATE_REPO_VERSION_FILE=OFF
               ${TRILINOS_SHARED}
               ${TRILINOS_ALPINE}
               ${TRILINOS_BLAS}
               ${TRILINOS_LAPACK}
               ${MACOSX_RPATH}
               -D TPL_ENABLE_Netcdf:BOOL=ON
               -D Netcdf_LIBRARY_DIRS:PATH=${netcdf_library_dirs}
               -D TPL_Netcdf_INCLUDE_DIRS:PATH=${netcdf_include_dirs}
               -D CMAKE_INSTALL_RPATH:PATH=${CMAKE_INSTALL_PREFIX}/lib
               -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D CMAKE_C_COMPILER:PATH=${MPI_C_COMPILER}
               -D CMAKE_CXX_COMPILER:PATH=${TRILINOS_CXX_COMPILER}
               -D CMAKE_C_FLAGS:STRING=${CFLAGS}
               -D CMAKE_CXX_FLAGS:STRING=${CXXFLAGS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX})

  if (NVCC_WRAPPER)
    list(APPEND TRILINOS_ARGS
         -D TPL_ENABLE_CUDA:BOOL=ON
         -D Kokkos_ENABLE_CUDA=ON)
  endif()

  if (HIP_COMPILER)
    # Unlike CUDA, HIP is not a TriBITS TPL: there is no TPL_ENABLE_HIP, and
    # Trilinos reaches the GPU solely through Kokkos_ENABLE_HIP.
    # CMAKE_CXX_STANDARD is set explicitly because Kokkos only skips that
    # requirement when it detects the compiler id as HIPCC, and here the
    # compiler reaches it through the MPI wrapper so it detects Clang.
    list(APPEND TRILINOS_ARGS
         -D Kokkos_ENABLE_HIP:BOOL=ON
         -D CMAKE_CXX_STANDARD:STRING=17)
  endif()

  if (NVCC_WRAPPER OR HIP_COMPILER)
    if(KOKKOS_ARCH)
      string(TOUPPER "${KOKKOS_ARCH}" _ARCH_UP)
      message(STATUS "Kokkos architecture: ${_ARCH_UP}")
      list(APPEND TRILINOS_ARGS "-D;Kokkos_ARCH_${_ARCH_UP}:BOOL=ON")
    endif()
  endif()

  # ROCm 6 fixups for the Kokkos bundled with the pinned Trilinos.
  #
  # Applied in place to the submodule working tree, only for HIP builds, so the
  # CUDA and CPU paths see an untouched Trilinos. Every entry must be written to
  # be idempotent, because ExternalProject re-runs the patch step whenever the
  # build tree's stamps are gone while the submodule stays patched from the
  # previous build.
  #
  # 1. Kokkos <= 4.2.0 prints hipDeviceProp_t::gcnArch, which ROCm 6.0 removed
  #    (hence hipDeviceProp_tR0600 in the error). Upstream Kokkos renamed the
  #    read to gcnArchName, which has existed since ROCm 4.x, so the edit is
  #    also correct on ROCm 5. Diagnostic output only; no semantic change.
  #    Idempotent: after the substitution "gcnArch\b" no longer matches.
  # 2. SPADDHandle in the bundled kokkos-kernels defines set_sort_option and
  #    get_sort_option, which read a member sort_option that is never declared.
  #    Nothing in the tree calls either accessor, and upstream kokkos-kernels
  #    deleted both (they are gone by Trilinos 16.1), but ROCm's clang
  #    diagnoses them where gcc did not. Deleting the two lines matches
  #    upstream.
  #    Idempotent: deleting an absent line is a no-op.
  set(TRILINOS_PATCH_COMMAND)
  if (HIP_COMPILER)
    set(KOKKOS_SRC "${PROJECT_SOURCE_DIR}/src/trilinos/packages/kokkos")
    set(KOKKOS_KERNELS_SRC
        "${PROJECT_SOURCE_DIR}/src/trilinos/packages/kokkos-kernels")
    set(TRILINOS_PATCH_COMMAND
        PATCH_COMMAND sed -i "s/hipProp\\.gcnArch\\b/hipProp.gcnArchName/g"
                      "${KOKKOS_SRC}/core/src/HIP/Kokkos_HIP_Instance.cpp"
        COMMAND sed -i -e "/void set_sort_option/d" -e "/int get_sort_option/d"
                      "${KOKKOS_KERNELS_SRC}/sparse/src/KokkosSparse_spadd_handle.hpp")
  endif()

  ExternalProject_Add(
    trilinos
    DEPENDS ${netcdf} ${lapack} ${ampi}
    PREFIX trilinos
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/trilinos
    BINARY_DIR ${CMAKE_BINARY_DIR}/src/trilinos
    ${TRILINOS_PATCH_COMMAND}
    CONFIGURE_COMMAND ${DEVICE_CXX_ENV}
      ${CMAKE_COMMAND} <SOURCE_DIR> ${TRILINOS_ARGS}
    BUILD_COMMAND   ${DEVICE_CXX_ENV} ${CMAKE_COMMAND} --build <BINARY_DIR> -j
    INSTALL_COMMAND ${DEVICE_CXX_ENV} ${CMAKE_COMMAND} --install <BINARY_DIR>
    DOWNLOAD_COMMAND ""
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### PEGTL (Parsing Expression Grammar Template Library) #######################
# https://github.com/taocpp/PEGTL
# Header only
if (pegtl)
 ExternalProject_Add(
   pegtl
   PREFIX pegtl
   # Header-only, copy include dir over
   CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/pegtl
                     ${CMAKE_INSTALL_PREFIX}/include/pegtl
   BUILD_COMMAND ""
   INSTALL_COMMAND ""
   DOWNLOAD_COMMAND ""
 )
endif()

#### TUT (Template Unit Test Framework) ########################################
# http://tut-framework.sourceforge.net
# Header only
if(tut)
  ExternalProject_Add(
    tut
    PREFIX tut
    # Header-only, copy include dir over
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                      ${PROJECT_SOURCE_DIR}/src/tut/include
                      ${CMAKE_INSTALL_PREFIX}/include/tut
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    DOWNLOAD_COMMAND ""
  )
endif()

#### Boost ####################################################################
# http://www.boost.org
# Header only, only if not found
if (boost)
  ExternalProject_Add(
    boost
    PREFIX boost
    # Header-only, copy include dir over
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                      ${PROJECT_SOURCE_DIR}/src/boost/boost
                      ${CMAKE_INSTALL_PREFIX}/include/boost/
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    DOWNLOAD_COMMAND ""
  )
endif()

#### pugixml ###################################################################
# http://pugixml.org
if (pugixml)
  ExternalProject_Add(
    pugixml
    PREFIX pugixml
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/pugixml
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D CMAKE_CXX_COMPILER:PATH=${UNDERLYING_CXX_COMPILER}
               -D CMAKE_CXX_FLAGS:STRING=${CXXFLAGS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
               -D BUILD_SHARED_LIBS:BOOL=${BUILD_SHARED_LIBS}
               ${MACOSX_RPATH}
    DOWNLOAD_COMMAND ""
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### eigen #####################################################################
if (eigen)
  ExternalProject_Add(
    eigen
    PREFIX eigen
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/eigen
                     ${PROJECT_BINARY_DIR}/eigen
    SOURCE_DIR  ${PROJECT_BINARY_DIR}/eigen
    BINARY_DIR  ${PROJECT_BINARY_DIR}/eigen/build
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
      -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
      -DBUILD_TESTING:BOOL=OFF
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### mutationpp ################################################################
# https://github.com/mutationpp/Mutationpp
if (mutationpp)
  # Copy Mutationpp/data/ to install directory
  set(TPL_RESOURCE_SRC "${CMAKE_CURRENT_SOURCE_DIR}/src/mutationpp/data")
  set(TPL_RESOURCE_DST "${CMAKE_INSTALL_PREFIX}/include/mutation++/data")

  add_custom_target(mutationpp_copy_resources ALL
    COMMAND "${CMAKE_COMMAND}" -E make_directory "${TPL_RESOURCE_DST}"
    # copy contents
    COMMAND /bin/bash -c "set -euo pipefail; cp -a \"${TPL_RESOURCE_SRC}/.\" \"${TPL_RESOURCE_DST}/\""
    COMMENT "Copying Mutation++ data directory to install folder"
    VERBATIM
  )

   # DEPEND on the conditional variable ${pugixml} (empty or 'pugixml' target)
   ExternalProject_Add(
     mutationpp
     PREFIX mutationpp
     DEPENDS ${eigen} ${pugixml} mutationpp_copy_resources
     DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                      ${PROJECT_SOURCE_DIR}/src/mutationpp
                      ${PROJECT_BINARY_DIR}/mutationpp
     SOURCE_DIR  ${PROJECT_BINARY_DIR}/mutationpp
     BINARY_DIR  ${PROJECT_BINARY_DIR}/mutationpp/build
     CMAKE_ARGS
       -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
       -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
       -DBUILD_SHARED_LIBS:BOOL=ON
       -DBUILD_TESTING:BOOL=OFF
       -DMPP_ENABLE_PYTHON:BOOL=OFF
       -DCMAKE_PREFIX_PATH:PATH=${CMAKE_INSTALL_PREFIX}
     LOG_DOWNLOAD 1
     LOG_CONFIGURE 1
     LOG_BUILD 1
     LOG_INSTALL 1
  )
endif()

#### NumDiff ###################################################################
# https://github.com/MethodicalAcceleratorDesign/MAD-X/tree/master/tools/numdiff
if(numdiff)
  ExternalProject_Add(
    numdiff
    PREFIX numdiff
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/numdiff
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D CMAKE_C_COMPILER:PATH=${UNDERLYING_C_COMPILER}
               -D CMAKE_C_FLAGS:STRING=${CFLAGS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
               -D BUILD_SHARED_LIBS:BOOL=${BUILD_SHARED_LIBS}
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### Backward-cpp, stack trace pretty printer for C++ ##########################
# https://github.com/bombela/backward-cpp
# Header only, only if not found
if (backwardcpp)
  ExternalProject_Add(
    backwardcpp
    PREFIX backwardcpp
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/backward-cpp
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D CMAKE_CXX_COMPILER:PATH=${UNDERLYING_CXX_COMPILER}
               -D CMAKE_CXX_FLAGS:STRING=${CXXFLAGS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### HighwayHash, Fast strong hash functions: SipHash/HighwayHash  #############
# https://github.com/google/highwayhash
# Header only, only if not found
if (highwayhash)
  ExternalProject_Add(
    highwayhash
    PREFIX highwayhash
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                      ${PROJECT_SOURCE_DIR}/src/highwayhash/highwayhash
                      ${CMAKE_INSTALL_PREFIX}/include/highwayhash
    LOG_CONFIGURE 1
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    DOWNLOAD_COMMAND ""
  )
endif()

#### Brigand ###################################################################
# https://github.com/edouarda/brigand
# Header only, only if not found
if (brigand)
  ExternalProject_Add(
    brigand
    PREFIX brigand
    # Header-only, copy include dir over
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/brigand/include/brigand
                     ${CMAKE_INSTALL_PREFIX}/include/brigand
    # Will apply patch (if any) in SOURCE_DIR which is the same as install dir
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}
    PATCH_COMMAND
      patch -p1 -t -i ${PROJECT_SOURCE_DIR}/cmake/cuda_brigand.patch
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
  )
endif()

#### Sol2 ######################################################################
# https://github.com/ThePhD/sol2
# Header only, only if not found
if (sol2)
  ExternalProject_Add(
    sol2
    PREFIX sol2
    # Header-only, copy include dir over
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/sol2/include/sol
                     ${CMAKE_INSTALL_PREFIX}/include/sol
    # Will apply patch (if any) in SOURCE_DIR which is the same as install dir
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}
    PATCH_COMMAND
      patch -p1 -t -i ${PROJECT_SOURCE_DIR}/cmake/emptymacro_sol2.patch
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
  )
endif()

#### Doxygen, documentation generation #########################################
# https://doxygen.org
if (doxygen)
  ExternalProject_Add(
    doxygen
    PREFIX doxygen
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/doxygen
    CMAKE_ARGS -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### m.css, modern, mobile-friendly doxygen output #############################
# http://mcss.mosra.cz
if (mcss)
  ExternalProject_Add(
    mcss
    PREFIX mcss
    # Copy over what's needed to run dox2html5.py
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                      ${PROJECT_SOURCE_DIR}/src/m.css
                      ${CMAKE_INSTALL_PREFIX}/m.css
    LOG_CONFIGURE 1
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    DOWNLOAD_COMMAND ""
  )
endif()
