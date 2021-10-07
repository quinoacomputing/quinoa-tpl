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
include(BuildShared)

# Conditionally enforce static linking for exeuctables added after this point.
# This must precede all add_executable() calls adding executables to link
# statically.
if (NOT BUILD_SHARED_LIBS)
  set(CMAKE_EXE_LINKER_FLAGS "-static")
  set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
endif()

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
set(ENABLE_HDF5 ON CACHE BOOL "Enable HDF5")
set(ENABLE_NETCDF ON CACHE BOOL "Enable NetCDF")
set(ENABLE_BOOST ON CACHE BOOL "Enable Boost")
set(ENABLE_PSTREAMS ON CACHE BOOL "Enable Pstreams")
set(ENABLE_PUGIXML ON CACHE BOOL "Enable PugiXML")
set(ENABLE_PEGTL ON CACHE BOOL "Enable PEGTL")
set(ENABLE_RANDOM123 ON CACHE BOOL "Enable Random123")
set(ENABLE_RNGSSE2 ON CACHE BOOL "Enable RNGSSE2")
set(ENABLE_LAPACK ON CACHE BOOL "Enable LAPACK")
set(ENABLE_AEC ON CACHE BOOL "Enable AEC")
set(ENABLE_H5PART ON CACHE BOOL "Enable H5Part")
set(ENABLE_TESTU01 ON CACHE BOOL "Enable TestU01")
set(ENABLE_TRILINOS ON CACHE BOOL "Enable Trilinos")
set(ENABLE_TUT ON CACHE BOOL "Enable TUT")
set(ENABLE_NUMDIFF ON CACHE BOOL "Enable NumDiff")
set(ENABLE_ROOT OFF CACHE BOOL "Enable Root")
set(ENABLE_BACKWARDCPP ON CACHE BOOL "Enable BackwardCpp")
set(ENABLE_HIGHWAYHASH ON CACHE BOOL "Enable HighwayHash")
set(ENABLE_OMEGAH OFF CACHE BOOL "Enable Omega_H")
set(ENABLE_BRIGAND ON CACHE BOOL "Enable Brigand")
set(ENABLE_SOL2 OFF CACHE BOOL "Enable Sol2")
set(ENABLE_DOXYGEN ON CACHE BOOL "Enable Doxygen")
set(ENABLE_MCSS ON CACHE BOOL "Enable m.css")
set(ENABLE_EXAM2M OFF CACHE BOOL "Enable ExaM2M")

# Set default requested executable targets
set(REQUESTED_EXECUTABLES unittest inciter rngtest meshconv walker fileconv)

# If a *_ONLY variable is set, user only wants specific executable(s)
if (UNITTEST_ONLY OR INCITER_ONLY OR RNGTEST_ONLY OR MESHCONV_ONLY OR
    WALKER_ONLY OR FILECONV_ONLY)

  # Turn everything off first
  set(ENABLE_CHARM OFF)
  set(ENABLE_HDF5 OFF)
  set(ENABLE_NETCDF OFF)
  set(ENABLE_BOOST OFF)
  set(ENABLE_PSTREAMS OFF)
  set(ENABLE_PUGIXML OFF)
  set(ENABLE_PEGTL OFF)
  set(ENABLE_RANDOM123 OFF)
  set(ENABLE_RNGSSE2 OFF)
  set(ENABLE_LAPACK OFF)
  set(ENABLE_AEC OFF)
  set(ENABLE_H5PART OFF)
  set(ENABLE_TESTU01 OFF)
  set(ENABLE_TRILINOS OFF)
  set(ENABLE_TUT OFF)
  set(ENABLE_NUMDIFF OFF)
  set(ENABLE_ROOT OFF)
  set(ENABLE_BACKWARDCPP OFF)
  set(ENABLE_HIGHWAYHASH OFF)
  set(ENABLE_OMEGAH OFF)
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
    set(ENABLE_PSTREAMS ON)
    set(ENABLE_PUGIXML ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_HDF5 ON)
    set(ENABLE_HIGHWAYHASH ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_TUT ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_LAPACK ON)
    set(ENABLE_RANDOM123 ON)
  endif()

  if (INCITER_ONLY)
    list(APPEND REQUESTED_EXECUTABLES inciter)
    set(ENABLE_CHARM ON)
    set(ENABLE_PSTREAMS ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_HDF5 ON)
    set(ENABLE_HIGHWAYHASH ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_LAPACK ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_NUMDIFF ON)
  endif()

  if (RNGTEST_ONLY)
    list(APPEND REQUESTED_EXECUTABLES rngtest)
    set(ENABLE_CHARM ON)
    set(ENABLE_PSTREAMS ON)
    set(ENABLE_TESTU01 ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_RANDOM123 ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_LAPACK ON)
  endif()

  if (MESHCONV_ONLY)
    list(APPEND REQUESTED_EXECUTABLES meshconv)
    set(ENABLE_CHARM ON)
    set(ENABLE_PSTREAMS ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_HDF5 ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_PUGIXML ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_HIGHWAYHASH ON)
    set(ENABLE_NUMDIFF ON)
  endif()

  if (WALKER_ONLY)
    list(APPEND REQUESTED_EXECUTABLES walker)
    set(ENABLE_CHARM ON)
    set(ENABLE_PSTREAMS ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_HDF5 ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_RANDOM123 ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_LAPACK ON)
    set(ENABLE_NUMDIFF ON)
    set(ENABLE_HIGHWAYHASH ON)
    set(ENABLE_H5PART ON)
  endif()

  if (FILECONV_ONLY)
    list(APPEND REQUESTED_EXECUTABLES fileconv)
    set(ENABLE_CHARM ON)
    set(ENABLE_PSTREAMS ON)
    set(ENABLE_TRILINOS ON)
    set(ENABLE_NETCDF ON)
    set(ENABLE_HDF5 ON)
    set(ENABLE_ROOT ON)
    set(ENABLE_PEGTL ON)
    set(ENABLE_PUGIXML ON)
    set(ENABLE_BOOST ON)
    set(ENABLE_BRIGAND ON)
    set(ENABLE_HIGHWAYHASH ON)
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

  # Use mpi wrappers by default
  set(CHARM_COMPILER "")

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
  if (NOT DEFINED ENV{CRAY_PRGENVPGI} AND  # not Cray
      NOT DEFINED ENV{CRAY_PRGENVGNU} AND
      NOT DEFINED ENV{CRAY_PRGENVCRAY} AND
      NOT DEFINED ENV{CRAY_PRGENVINTEL})

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

    set(CHARM_ARCHITECTURE "${CHARM_COMM_SUBSYSTEM}-${SYSTEM_NAME}${CHARM_ARCH}")

  else()  # Cray

    if(NOT ENABLE_AMPI)
      set(CHARM_COMM_SUBSYSTEM "mpi")
    else()
      set(CHARM_COMM_SUBSYSTEM "gni")   # never tested
    endif()

    set(CHARM_ARCHITECTURE "${CHARM_COMM_SUBSYSTEM}-crayxc")

  endif()

  message(STATUS "Charm++ architecture/version: ${CHARM_ARCHITECTURE}")

  if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Cray")
    set(CHARM_COMPILER "craycc")
  elseif (ENABLE_AMPI AND "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(CHARM_COMPILER "clang")
  elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
    set(CHARM_COMPILER "icc")
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
  if (ENABLE_EXAM2M)
    set(CHARM_BUILD_TARGET "LIBS")
  else()
    set(CHARM_BUILD_TARGET "charm++")
  endif()
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

# HDF5
if (ENABLE_HDF5)
  set(HDF5_PREFER_PARALLEL true)
  if(NOT BUILD_SHARED_LIBS)
    set(HDF5_USE_STATIC_LIBRARIES true)
  endif()
  if(ENABLE_AMPI)
    set(HDF5_ROOT ${CMAKE_INSTALL_PREFIX})
  endif()
  find_package(HDF5 COMPONENTS C HL)
  if ((ENABLE_AMPI AND HDF5_FOUND AND HDF5_IS_PARALLEL AND
       HDF5_INCLUDE_DIRS MATCHES ${CMAKE_INSTALL_PREFIX}) OR
      (NOT ENABLE_AMPI AND HDF5_FOUND AND HDF5_IS_PARALLEL))
    set(hdf5 "")
    set(hdf5_include_dirs_flags "${HDF5_INCLUDE_DIRS}")
    string(REGEX REPLACE "([^;]+)" "CPPFLAGS=-I\\1" hdf5_include_dirs_flags
           "${hdf5_include_dirs_flags}")
    set(hdf5_library_dirs "${HDF5_LIBRARIES}")
    list(GET HDF5_LIBRARIES 0 hdf5_root)  # get the first item of cmake list
    get_filename_component(hdf5_root_last ${hdf5_root} NAME)
    get_filename_component(hdf5_root ${hdf5_root} DIRECTORY )
    set(hdf5_library_dirs_flags "${hdf5_root}")
    string(REGEX REPLACE "([^;]+)" "LDFLAGS=-L\\1" hdf5_library_dirs_flags
          "${hdf5_library_dirs_flags}")
    set(hdf5_library_dirs_flags "${hdf5_library_dirs_flags}/lib")
    # strip "/lib" and "/lib64"
    if ("${hdf5_root}" MATCHES "lib(64)?$")
      get_filename_component(hdf5_root ${hdf5_root} DIRECTORY)
    endif()
  else()
    message(STATUS "HDF5 either not found, not parallel, disabled, or AMPI is enabled: will build")
    set(hdf5 "hdf5")
    set(hdf5_ampi "-ampi")
    set(hdf5_include_dirs_flags "CPPFLAGS=-I${CMAKE_INSTALL_PREFIX}/include")
    set(hdf5_library_dirs_flags "LDFLAGS=-L${CMAKE_INSTALL_PREFIX}/lib")
    set(hdf5_library_dirs "${CMAKE_INSTALL_PREFIX}/lib")
    set(hdf5_root "${CMAKE_INSTALL_PREFIX}")
  endif()
endif()

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
  if(NOT BUILD_SHARED_LIBS)
    set(Boost_USE_STATIC_LIBS true)
  endif()
  find_package(Boost 1.56.0)
  if(Boost_FOUND)
    set(boost "")
  else()
    set(boost "boost")
  endif()
endif()

# PStreams
if (ENABLE_PSTREAMS)
  find_package(Pstreams)
  if(PSTREAMS_FOUND)
    set(pstreams "")
  else()
    set(pstreams "pstreams")
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

# Random123
if (ENABLE_RANDOM123)
  find_package(Random123)
  if(Random123_FOUND)
    set(random123 "")
  else()
    set(random123 "random123")
  endif()
endif()

# RNGSSE2
if (ENABLE_RNGSSE2)
  # Do not attempt to build RNGSSE2 if target architecture is not x86
  find_package(RNGSSE2)
  if(RNGSSE2_FOUND OR NOT ARCH MATCHES "x86")
    set(rngsse2 "")
  else()
    set(rngsse2 "rngsse2")
  endif()
endif()

# BLAS and LAPACK
if (ENABLE_LAPACK)
  set(lapack "lapack")
  if (NOT MATHLIB)        # set default
    set(MATHLIB mkl)
  endif()
  if (MATHLIB STREQUAL mkl OR MATHLIB STREQUAL MKL)
    find_package(MKL)     # Prefer MKL if found
  endif()
  if(MKL_FOUND)
    set(lapack "")
  else()
    find_package(BLAS)
    find_package(LAPACKE)
    if(LAPACKE_FOUND)
      set(lapack "")
    endif()
  endif()
endif()

# AEC
if (ENABLE_AEC)
  if(NOT BUILD_SHARED_LIBS)  # only for static link)
    set(aec "aec")
    find_package(AEC)
    if(AEC_FOUND)
      set(aec "")
    endif()
  endif()
endif()

# H5Part
if (ENABLE_H5PART)
  if(ENABLE_AMPI)
    set(H5PART_ROOT ${CMAKE_INSTALL_PREFIX})
  endif()
  find_package(H5Part)
  if ((ENABLE_AMPI AND H5Part_FOUND AND
       H5PART_INCLUDE_DIRS MATCHES ${CMAKE_INSTALL_PREFIX}) OR
      (NOT ENABLE_AMPI AND H5Part_FOUND))
    set(h5part "")
  else()
    set(h5part "h5part")
    message(STATUS "H5Part either not found or AMPI is enabled: will build")
  endif()
endif()

# TestU01
if (ENABLE_TESTU01)
  find_package(TestU01)
  if(TestU01_FOUND)
    set(testu01 "")
  else()
    set(testu01 "testu01")
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

# Root
if (ENABLE_ROOT)
  find_package(Root COMPONENTS RIO Core Tree Hist)
  if(Root_FOUND)
    set(root "")
  elseif(NOT EXISTS ${PROJECT_SOURCE_DIR}/src/root/CMakeLists.txt)
    message(WARNING "ROOT enabled, but not found on system and not found as a git submodule. If you want ROOT, try switching to branch 'all' of the TPL repository (by running 'git checkout all' in the TPL repo).")
    set(root "")
  else()
    set(root "root")
    # Root does not support libc++ on linux, so remove if configured
    if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
      string(FIND "${CXXFLAGS}" "-stdlib=libc++" pos)
      if (NOT "${pos}" STREQUAL "-1")
        message(STATUS "Removing C++ compiler flag '-stdlib=libc++' as Root does not support it")
        string(REPLACE "-stdlib=libc++" "" CXXFLAGS ${CXXFLAGS})
      endif()
    endif()
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

# Omega_h
if (ENABLE_OMEGA_H)
  if(ENABLE_AMPI)
    set(OMEGA_H_ROOT ${CMAKE_INSTALL_PREFIX})
  endif()
  find_package(Omega_h)
  if ((ENABLE_AMPI AND Omega_h_FOUND AND
       OMEGA_H_INCLUDE_DIRS MATCHES ${CMAKE_INSTALL_PREFIX}) OR
      (NOT ENABLE_AMPI AND Omega_h_FOUND))
    set(omega_h "")
  else()
    set(omega_h "omega_h")
    message(STATUS "Omega_h either not found or AMPI is enabled: will build")
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

# ExaM2M
if (ENABLE_EXAM2M)
  find_package(ExaM2M)
  if(ExaM2M_FOUND)
    set(exam2m"")
  else()
    set(exam2m "exam2m")
  endif()
endif()

# Get compiler flags (CFLAGS, CXXFLAGS, FFLAGS) independent of CMAKE_BUILD_TYPE
# and echo flags that will be passed to all TPL builds
include(get_compiler_flags)
get_compiler_flags()

set(tpls2build ${charm} ${hdf5} ${netcdf} ${boost} ${pstreams}
${pugixml} ${pegtl} ${random123} ${rngsse2} ${lapack} ${aec} ${h5part}
${testu01} ${trilinos} ${tut} ${numdiff} ${root} ${backwardcpp} ${highwayhash}
${omega_h} ${brigand} ${sol2} ${doxygen} ${mcss} ${exam2m})

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
if(BUILD_SHARED_LIBS)
  set(CHARM_SHARED "--build-shared")
endif()
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

#### HDF5 ######################################################################
# http://www.hdfgroup.org/HDF5
if (hdf5)
  if(NOT BUILD_SHARED_LIBS)
    set(HDF5_PATCH "patch;-p1;-t;-i;${PROJECT_SOURCE_DIR}/cmake/hdf5_remove_get_time.patch")
  endif()
  if(BUILD_SHARED_LIBS AND NOT ENABLE_AMPI)
    set(HDF5_SHARED "--enable-shared")
  else()
    set(HDF5_SHARED "--disable-shared;--enable-static;--with-pic")
  endif()
  if (ENABLE_AMPI)
    set(AMPI_RUNSERIAL "RUNSERIAL=\'\'${AMPI_RUN} ++quiet\'\'" )
    set(AMPI_RUNPARALLEL "RUNPARALLEL=\'\'${AMPI_RUN} ++quiet +p4\'\'" )
  endif()
  ExternalProject_Add(
    hdf5
    PREFIX hdf5
    DEPENDS ${ampi}
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/hdf5${hdf5_ampi}
                     ${PROJECT_BINARY_DIR}/hdf5
    SOURCE_DIR ${PROJECT_BINARY_DIR}/hdf5
    PATCH_COMMAND ${HDF5_PATCH}
    CONFIGURE_COMMAND ${PROJECT_BINARY_DIR}/hdf5/configure
                      CC=${MPI_C_COMPILER}
                      CFLAGS=${CFLAGS}
                      ${PGCPP}
                      ${AMPI_RUNSERIAL}
                      ${AMPI_RUNPARALLEL}
                      --enable-parallel
                      --enable-symbols=yes
                      ${HDF5_SHARED}
                      --prefix=${CMAKE_INSTALL_PREFIX}
    BUILD_IN_SOURCE 1
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### NETCDF ####################################################################
## http://www.unidata.ucar.edu/downloads/netcdf/index.jsp
if (netcdf)
  if(BUILD_SHARED_LIBS AND NOT ENABLE_AMPI)
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
    DEPENDS ${hdf5} ${ampi}
    PREFIX netcdf
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/netcdf
                     ${PROJECT_BINARY_DIR}/netcdf
    CONFIGURE_COMMAND ${PROJECT_BINARY_DIR}/netcdf/configure
                      CC=${MPI_C_COMPILER}
                      CFLAGS=${CFLAGS}
                      ${NETCDF_UTILS}
                      ${PGCPP}
                      ${hdf5_include_dirs_flags}
                      ${hdf5_library_dirs_flags}
                      --enable-netcdf4
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

#### AEC #######################################################################
## https://gitlab.dkrz.de/k202009/libaec
if (aec)
  if(BUILD_SHARED_LIBS)
    set(AEC_SHARED "--enable-shared")
  else()
    set(AEC_SHARED "--enable-static;--disable-shared")
  endif()
  ExternalProject_Add(
    aec
    PREFIX aec
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/aec
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D CMAKE_C_COMPILER:PATH=${UNDERLYING_C_COMPILER}
               -D CMAKE_C_FLAGS:STRING=${FCLAGS}
               -D BUILD_SHARED_LIBS:BOOL=${BUILD_SHARED_LIBS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    DOWNLOAD_COMMAND ""
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### lapack ####################################################################
# http://www.netlib.org/lapack
if(BUILD_SHARED_LIBS AND NOT ENABLE_AMPI)
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
if(BUILD_SHARED_LIBS AND NOT ENABLE_AMPI)
  set(TRILINOS_SHARED "-DBUILD_SHARED_LIBS=ON")
else()
  set(TRILINOS_SHARED "-DBUILD_SHARED_LIBS=OFF;-DTPL_FIND_SHARED_LIBS:BOOL=OFF;-DTrilinos_LINK_SEARCH_START_STATIC:BOOL=ON;-DCMAKE_SKIP_INSTALL_RPATH=TRUE")
  if(ARCH MATCHES "ppc64")
    set(tr_hdf5 "-DTrilinos_EXTRA_LINK_FLAGS=-L${hdf5_library_dirs} -lhdf5_hl -lhdf5 ${ZLIB_LIBRARIES} -lm -ldl")
  else()
    set(tr_hdf5 "-DTrilinos_EXTRA_LINK_FLAGS=-L${hdf5_library_dirs} -lhdf5_hl -lhdf5 -lz -lm -ldl")
  endif()
endif()
if (HOST_OS MATCHES "alpine")
  set(TRILINOS_ALPINE "-DBLAS_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib;-DLAPACK_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib;-DTrilinos_CXX11_FLAGS=-std=c++11")
endif()
if (MKL_FOUND)
  set(TRILINOS_MKL "-DTPL_ENABLE_BLAS=ON;-DTPL_BLAS_LIBRARIES=${MKL_LIBRARIES};-DTPL_LAPACK_LIBRARIES=${MKL_LIBRARIES}")
  if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(TRILINOS_MKL "-DTPL_ENABLE_MKL=ON;-DMKL_LIBRARY_DIRS=$ENV{MKLROOT}/lib;-DMKL_INCLUDE_DIRS=$ENV{MKLROOT}/include;-DTPL_BLAS_LIBRARIES=${MKL_LIBRARIES};-DTPL_LAPACK_LIBRARIES=${MKL_LIBRARIES}")
  endif()
else()
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
endif()
if(trilinos)
  ExternalProject_Add(
    trilinos
    DEPENDS ${hdf5} ${netcdf} ${lapack} ${ampi}
    PREFIX trilinos
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/trilinos
    PATCH_COMMAND patch -p1 -t -i ${PROJECT_SOURCE_DIR}/cmake/exodiff_cstdint.patch
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
               ${tr_hdf5}
               ${TRILINOS_SHARED}
               ${TRILINOS_ALPINE}
               ${TRILINOS_MKL}
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

#### Random123 (Counter-based random number generators library) ################
# http://www.thesalmons.org/john/random123/releases/latest/docs/index.html
# Header only
if (random123)
 ExternalProject_Add(
   random123
   PREFIX random123
   # Header-only, copy include dir over
   CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/random123/include
                     ${CMAKE_INSTALL_PREFIX}/include
   # Copy examples/uniform.hpp over
   BUILD_COMMAND ${CMAKE_COMMAND} -E copy
                 ${PROJECT_SOURCE_DIR}/src/random123/examples/uniform.hpp
                 ${CMAKE_INSTALL_PREFIX}/include/Random123
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

#### RNGSSELib2 ################################################################
# http://cpc.cs.qub.ac.uk/summaries/AEIT_v2_0.html
# No need for MPI, use underlying compilers
if (rngsse2)
  ExternalProject_Add(
    rngsse2
    PREFIX rngsse2
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/rngsse2
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D BUILD_SHARED_LIBS=${BUILD_SHARED_LIBS}
               ${MACOSX_RPATH}
               -D CMAKE_CXX_COMPILER:PATH=${UNDERLYING_CXX_COMPILER}
               -D CMAKE_CXX_FLAGS:STRING=${CXXFLAGS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    DOWNLOAD_COMMAND ""
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()

#### TestU01 ##################################################################
# http://www.iro.umontreal.ca/~simardr/testu01/tu01.html
# No need for MPI, use underlying compilers
if (testu01)
  ExternalProject_Add(
    testu01
    PREFIX testu01
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/testu01
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D BUILD_SHARED_LIBS=${BUILD_SHARED_LIBS}
               ${MACOSX_RPATH}
               -D CMAKE_C_COMPILER:PATH=${UNDERLYING_C_COMPILER}
               -D CMAKE_C_FLAGS:STRING=${CFLAGS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    DOWNLOAD_COMMAND ""
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
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

#### Pstreams (POSIX Process Control in C++) ###################################
# http://pstreams.sourceforge.net/
# Header only, only if not found
if (pstreams)
  ExternalProject_Add(
    pstreams
    PREFIX pstreams
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E make_directory
                      ${CMAKE_INSTALL_PREFIX}/include/pstreams
    BUILD_COMMAND ${CMAKE_COMMAND} -E copy
                  ${PROJECT_SOURCE_DIR}/src/pstreams/pstream.h
                  ${CMAKE_INSTALL_PREFIX}/include/pstreams
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

#### H5part ####################################################################
# http://vis.lbl.gov/Research/H5Part
if(BUILD_SHARED_LIBS)
  set(H5PART_SHARED "--enable-shared;--disable-static")
else()
  set(H5PART_SHARED "--disable-shared;--enable-static")
endif()
if(h5part)
  ExternalProject_Add(
    h5part
    DEPENDS ${hdf5} ${ampi}
    PREFIX h5part
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
                     ${PROJECT_SOURCE_DIR}/src/h5part
                     ${PROJECT_BINARY_DIR}/h5part
    PATCH_COMMAND autoreconf -fi ${PROJECT_BINARY_DIR}/h5part
    CONFIGURE_COMMAND ${PROJECT_BINARY_DIR}/h5part/configure
                      FC=${MPI_Fortran_COMPILER}
                      FFLAGS=${FFLAGS}
                      CC=${MPI_C_COMPILER}
                      CFLAGS=${CFLAGS}
                      CXX=${MPI_CXX_COMPILER}
                      CXXFLAGS=${CXXFLAGS}
                      --with-hdf5=${hdf5_root}
                      --enable-parallel
                      ${H5PART_SHARED}
                      --prefix=${CMAKE_INSTALL_PREFIX}
    BUILD_IN_SOURCE 1
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

#### Root ######################################################################
# https://root.cern.ch
if(root)
  ExternalProject_Add(
    root
    PREFIX root
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/root
    CMAKE_ARGS -D CMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
               -D CMAKE_CXX_COMPILER:PATH=${UNDERLYING_CXX_COMPILER}
               -D CMAKE_C_COMPILER:PATH=${UNDERLYING_C_COMPILER}
               -D CMAKE_CXX_FLAGS:STRING=${CXXFLAGS}
               -D CMAKE_C_FLAGS:STRING=${CFLAGS}
               -Ddavix=OFF
               -D CMAKE_CXX_STANDARD=17
               -D BUILD_SHARED_LIBS:BOOL=${BUILD_SHARED_LIBS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
               -Dimt=OFF
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

#### Omega_h, tetrahedron and triangle mesh adaptativity #######################
# https://github.com/SNLComputation/omega_h
if (omega_h)
  ExternalProject_Add(
    omega_h
    PREFIX omega_h
    DEPENDS ${ampi}
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/omega_h
    CMAKE_ARGS -D Omega_h_USE_MPI:BOOL=ON
               -D CMAKE_CXX_COMPILER:PATH=${MPI_CXX_COMPILER}
               -D Omega_h_EXTRA_CXX_FLAGS:STRING=${CXXFLAGS}
               -D BUILD_SHARED_LIBS:BOOL=${BUILD_SHARED_LIBS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
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
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_directory
                      ${PROJECT_SOURCE_DIR}/src/sol2/include/sol
                      ${CMAKE_INSTALL_PREFIX}/include/sol
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    DOWNLOAD_COMMAND ""
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

#### ExaM2M, tetrahedron mesh-to-mesh transfer #################################
# https://github.com/Charmworks/ExaM2M
if (exam2m)
  ExternalProject_Add(
    exam2m
    PREFIX exam2m
    DEPENDS ${charm} ${brigand} ${hdf5} ${netcdf} ${highwayhash} ${trilinos}
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/src/exam2m/src
    CMAKE_ARGS -D CMAKE_C_COMPILER:PATH=${MPI_C_COMPILER}
               -D CMAKE_CXX_COMPILER:PATH=${MPI_CXX_COMPILER}
               -D BUILD_SHARED_LIBS:BOOL=${BUILD_SHARED_LIBS}
               -D CMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    LOG_DOWNLOAD 1
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
  )
endif()
