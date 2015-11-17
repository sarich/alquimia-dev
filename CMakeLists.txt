# Minimum CMake version.
cmake_minimum_required (VERSION 2.8.12)

# Adjust CMake's module path.
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")

# Options for building Alquimia. These come from the xSDK compliance rules.
option(USE_XSDK_DEFAULTS "Set to use xSDK defaults for options [ON]." ON)
option(CMAKE_INSTALL_PREFIX "Sets installation prefix [/usr/local]." "/usr/local")
option(XSDK_ENABLE_DEBUG "Enables Debug mode builds [OFF]." OFF)
option(BUILD_SHARED_LIBS "Builds shared libraries [ON]." ON)
option(XSDK_PRECISION "Sets the precision of floating-point numbers shared libraries (SINGLE,DOUBLE) [DOUBLE].")
option(XSDK_WITH_PFLOTRAN "Enables support for the PFlotran chemistry engine [OFF]." OFF)
option(TPL_PFLOTRAN_LIBRARIES "List of absolute paths to PFlotran link libraries [].")
option(TPL_PFLOTRAN_INCLUDE_DIRS "List of absolute paths to PFlotran include directories [].")
option(XSDK_WITH_CRUNCHFLOW "Enables support for the CrunchFlow chemistry engine [OFF]." OFF)
option(TPL_CRUNCHFLOW_LIBRARIES "List of absolute paths to CrunchFlow link libraries [].")
option(TPL_CRUNCHFLOW_INCLUDE_DIRS "List of absolute paths to CrunchFlow include directories [].")

# Check defaults for xSDK if needed.
if (NOT XSDK_PRECISION)
  set(XSDK_PRECISION DOUBLE)
endif()

# Check third-party library dependencies.
if (XSDK_WITH_PFLOTRAN)
  if (NOT TPL_PFLOTRAN_LIBRARIES)
    message(FATAL_ERROR "TPL_PFLOTRAN_LIBRARIES option be set for PFlotran support to be enabled.")
  endif()
  foreach(lib ${TPL_PFLOTRAN_LIBRARIES})
    if (NOT EXISTS ${lib})
      message(FATAL_ERROR "PFlotran library not found: ${lib}")
    endif()
  endforeach()

  if (NOT TPL_PFLOTRAN_INCLUDE_DIRS)
    message(FATAL_ERROR "TPL_PFLOTRAN_INCLUDE_DIRS option be set for PFlotran support to be enabled.")
  endif()
  foreach(dir ${TPL_PFLOTRAN_INCLUDE_DIRS})
    if (NOT EXISTS ${dir})
      message(FATAL_ERROR "PFlotran include directory not found: ${dir}")
    endif()
  endforeach()

  # Check for PETSc.
  if ($ENV{PETSC_DIR} STREQUAL "")
    message(FATAL_ERROR "PETSC_DIR must be set for PFlotran support to be enabled.")
  endif()
  if ($ENV{PETSC_ARCH} STREQUAL "")
    message(FATAL_ERROR "PETSC_ARCH must be set for PFlotran support to be enabled.")
  endif()

  set(ALQUIMIA_HAVE_PFLOTRAN 1)
endif()

if (XSDK_WITH_CRUNCHFLOW)
  if (NOT TPL_CRUNCHFLOW_LIBRARIES)
    message(FATAL_ERROR "TPL_CRUNCHFLOW_LIBRARIES option be set for CrunchFlow support to be enabled.")
  endif()
  foreach(lib ${TPL_CRUNCHFLOW_LIBRARIES})
    if (NOT EXISTS ${lib})
      message(FATAL_ERROR "CrunchFlow library not found: ${lib}")
    endif()
  endforeach()

  if (NOT TPL_CRUNCHFLOW_INCLUDE_DIRS)
    message(FATAL "TPL_CRUNCHFLOW_INCLUDE_DIRS option be set for CrunchFlow support to be enabled.")
  endif()
  foreach(dir ${TPL_CRUNCHFLOW_INCLUDE_DIRS})
    if (NOT EXISTS ${dir})
      message(FATAL "CrunchFlow include directory not found: ${dir}")
    endif()
  endforeach()

  set(ALQUIMIA_HAVE_CRUNCHFLOW 1)
endif()

# Make sure compilers are set. This must be done before enabling languages.
if (NOT CMAKE_C_COMPILER)
  set(CMAKE_C_COMPILER "${CC}")
endif()
if (NOT CMAKE_C_FLAGS)
  set(CMAKE_C_FLAGS "${CFLAGS}")
endif()
if (NOT CMAKE_Fortran_COMPILER)
  set(CMAKE_Fortran_COMPILER "${FC}")
endif()
if (NOT CMAKE_Fortran_FLAGS)
  set(CMAKE_Fortran_FLAGS "${FCFLAGS}")
endif()
enable_language(C)
enable_language(Fortran)

# We declare the project here.
project (alquimia)

message("-- C compiler is ${CMAKE_C_COMPILER} (${CMAKE_C_COMPILER_ID})")
message("-- Fortran compiler is ${CMAKE_Fortran_COMPILER} (${CMAKE_Fortran_COMPILER_ID})")

# Version numbers.
set (ALQUIMIA_MAJOR_VERSION 0)
set (ALQUIMIA_MINOR_VERSION 3)
set (ALQUIMIA_PATCH_VERSION 1)
set (ALQUIMIA_VERSION "${ALQUIMIA_MAJOR_VERSION}.${ALQUIMIA_MINOR_VERSION}.${ALQUIMIA_PATCH_VERSION}")

# Set up floating point numbers.
if (XSDK_PRECISION STREQUAL "SINGLE")
  set(HAVE_SINGLE_PRECISION 1)
  set(HAVE_DOUBLE_PRECISION 0)
  set(ALQUIMIA_REAL_TYPE float)
elseif(XSDK_PRECISION STREQUAL "DOUBLE")
  set(HAVE_SINGLE_PRECISION 0)
  set(HAVE_DOUBLE_PRECISION 1)
  set(ALQUIMIA_REAL_TYPE double)
else()
  message(FATAL_ERROR "Unsupported precision: ${XSDK_PRECISION}")
endif()
message("-- Real number representation is ${ALQUIMIA_REAL_TYPE}")

# Figure out the system type.
if (APPLE EQUAL 1)
  set(SYS_FLAGS "-DAPPLE=1")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -framework Accelerate")
else ()
  if (LINUX EQUAL 1)
    set(SYS_FLAGS "-DLINUX=1")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pthread")
  endif ()
endif ()

# General C compiler flags.
if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -Wall -pedantic-errors -Wextra -Werror-implicit-function-declaration")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-sign-compare -Wno-unused-parameter -Wno-unused-but-set-variable -Wno-int-to-pointer-cast -Wno-pointer-to-int-cast")

  if (LINUX EQUAL 1)
    # Counter some of GCC's more recent stinginess on Linux.
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_POSIX_C_SOURCE=200809L")# -D_BSD_SOURCE")
    # Pass some more needed flags to the compiler.
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread")
  endif()

elseif (CMAKE_C_COMPILER_ID STREQUAL "Clang")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -Wall -pedantic-errors -Wextra -Werror-implicit-function-declaration -fno-builtin")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-sign-compare -Wno-unused-parameter -Wno-int-to-pointer-cast -Wno-pointer-to-int-cast -Wno-unused-function")
elseif (CMAKE_C_COMPILER_ID STREQUAL "Intel")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -Wall")
endif()
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${SYS_FLAGS}")

# Fortran compiler flags.
if (CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")
  set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -W -Wall -std=gnu -pedantic -Wno-unused-variable -Wno-unused-parameter")
endif()

# Include the binary directory in the header file search path,
# since it's where we place the third-party libraries.
include_directories("${PROJECT_BINARY_DIR}")
include_directories("${PROJECT_BINARY_DIR}/include")
link_directories("${PROJECT_BINARY_DIR}/lib")

# Unit testing.
enable_testing()

# Source code itself.
include_directories("${PROJECT_SOURCE_DIR}")
add_subdirectory(alquimia)

# Now that we have gathered all our libraries, generate an alquimia.cmake 
# file that contains all the vital information.
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Templates/alquimia.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/alquimia.cmake"
  @ONLY
)

# Install all source headers.
install(DIRECTORY ${PROJECT_BINARY_DIR}/include/ DESTINATION include/alquimia)

# Install miscellaneous build/test files.
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/alquimia.cmake DESTINATION share/alquimia)
