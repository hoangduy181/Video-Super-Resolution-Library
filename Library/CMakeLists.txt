set(SOURCES Raisr_AVX256.cpp Raisr.cpp RaisrHandler.cpp)
set(HEADERS Raisr_globals.h Raisr_AVX256.h Raisr.h ThreadPool.h RaisrHandler.h RaisrDefaults.h)

if( HAVE_AVX512FP16 )
message("Building AVX512FP16 library")
list(APPEND SOURCES Raisr_AVX512FP16.cpp Raisr_AVX512FP16.h)
endif()
if( HAVE_AVX512 )
message("Building AVX512 library")
list(APPEND SOURCES Raisr_AVX512.cpp Raisr_AVX512.h)
else()
message("Building AVX2 only")
endif()

if (ENABLE_RAISR_OPENCL)
    find_package (PkgConfig REQUIRED)
    pkg_check_modules (OPENCL REQUIRED OpenCL)
    include_directories (${OPENCL_INCLUDE_DIRS})
    link_directories (${OPENCL_LIBRARY_DIRS})
    list(APPEND SOURCES Raisr_OpenCL.cpp)
    list(APPEND HEADERS Raisr_OpenCL.h Raisr_OpenCL_kernel.h)
endif()

add_library(raisr STATIC ${SOURCES} ${HEADERS})

# Raisr.cpp #includes Raisr_AVX256.cpp, so AVX2 intrinsics are compiled in that
# translation unit. A standalone Raisr_AVX256.cpp compile may not instantiate
# those inline bodies, so -march=native alone can appear to "work" for that file
# while Raisr.cpp.o fails (e.g. Docker/VM CPUs where native_tune omits AVX).
if(UNIX)
    target_compile_options(raisr PRIVATE -mavx2 -mfma)
endif()

# Link our library
# for IPP
if( UNIX )
include_directories($ENV{IPPROOT}/include)
find_library(IPP_CORE libippcore.a PATHS $ENV{IPPROOT}/lib/intel64)
find_library(IPP_VM libippvm.a PATHS     $ENV{IPPROOT}/lib/intel64)
find_library(IPP_S libipps.a PATHS       $ENV{IPPROOT}/lib/intel64)
find_library(IPP_I libippi.a PATHS       $ENV{IPPROOT}/lib/intel64)
find_package(Threads REQUIRED)
else()
include_directories("C:/Program Files (x86)/Intel/oneAPI/ipp/2021.2.0/include")
find_library(IPP_CORE ippcore.lib PATHS "C:/Program Files (x86)/Intel/oneAPI/ipp/2021.2.0/lib/intel64")
find_library(IPP_VM ippvm.lib PATHS     "C:/Program Files (x86)/Intel/oneAPI/ipp/2021.2.0/lib/intel64")
find_library(IPP_S ipps.lib PATHS       "C:/Program Files (x86)/Intel/oneAPI/ipp/2021.2.0/lib/intel64")
find_library(IPP_I ippi.lib PATHS       "C:/Program Files (x86)/Intel/oneAPI/ipp/2021.2.0/lib/intel64")
endif(UNIX)
target_link_libraries(raisr ${IPP_I} ${IPP_VM} ${IPP_S} ${IPP_CORE} ${CMAKE_THREAD_LIBS_INIT} ${OPENCL_LIBRARIES})
  include(GNUInstallDirs)
  install(DIRECTORY "." DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/raisr" FILES_MATCHING PATTERN "*.h")
  install(TARGETS "raisr" LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")

