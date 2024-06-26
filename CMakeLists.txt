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

# Query target architecture
include(TargetArch)
target_architecture(ARCH)
message(STATUS "Detected target architecture: ${ARCH}")

# Set install directory based on compiler ID if was not given on command line
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "${PROJECT_SOURCE_DIR}/install/${COMP}-${ARCH}"
      CACHE STRING "Third-party install prefix." FORCE)
  message(STATUS "CMAKE_INSTALL_PREFIX not specified, setting to '${PROJECT_SOURCE_DIR}/install/${COMP}-${ARCH}'")
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
if (ENABLE_NETCDF)
  if(ENABLE_AMPI)
    set(NETCDF_ROOT ${CMAKE_INSTALL_PREFIX})
  endif()
  find_package(NetCDF)
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

# Pugixml
if (ENABLE_PUGIXML)
  find_package(Pugixml)
  if(PUGIXML_FOUND)
    set(pugixml "")
  else()
    set(pugixml "pugixml")
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

# Get compiler flags (CFLAGS, CXXFLAGS, FFLAGS) independent of CMAKE_BUILD_TYPE
# and echo flags that will be passed to all TPL builds
include(get_compiler_flags)
get_compiler_flags()

set(tpls2build ${charm} ${netcdf} ${boost} ${pugixml} ${pegtl}
${lapack} ${trilinos} ${tut} ${numdiff} ${backwardcpp} ${highwayhash}
${brigand} ${sol2} ${doxygen} ${mcss})

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
  ExternalProject_Add(
    trilinos
    DEPENDS ${netcdf} ${lapack} ${ampi}
    PREFIX trilinos
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/trilinos
    CMAKE_ARGS -D Trilinos_ENABLE_ALL_PACKAGES:BOOL=OFF
               -D TPL_ENABLE_MPI:BOOL=ON
               -D Trilinos_ENABLE_ALL_OPTIONAL_PACKAGES=OFF
               -D Trilinos_ENABLE_Zoltan2:BOOL=ON
               -D Zoltan2_ENABLE_Experimental:BOOL=ON
               -D Trilinos_ENABLE_SECONDARY_STABLE_CODE:BOOL=ON
               -D Trilinos_ENABLE_SEACASExodus:BOOL=ON
               -D Trilinos_ENABLE_SEACASExodiff:BOOL=ON
               -D Trilinos_ENABLE_Fortran=OFF
               -D Trilinos_ENABLE_DEBUG=OFF
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
               -D CMAKE_CXX_COMPILER:PATH=${MPI_CXX_COMPILER}
               -D CMAKE_C_FLAGS:STRING=${CFLAGS}
               -D CMAKE_CXX_FLAGS:STRING=${CXXFLAGS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
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
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                      ${PROJECT_SOURCE_DIR}/src/brigand/include/brigand
                      ${CMAKE_INSTALL_PREFIX}/include/brigand
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    DOWNLOAD_COMMAND ""
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
