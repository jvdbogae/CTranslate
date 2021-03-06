cmake_minimum_required(VERSION 3.1.0)
project(onmt)

option(LIB_ONLY "Do not compile clients" OFF)
option(WITH_OPENMP "Use OpenMP if available" ON)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

set(CMAKE_CXX_STANDARD 11)

# Set Release build type by default to get sane performance.
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif(NOT CMAKE_BUILD_TYPE)

message(STATUS "Build type: " ${CMAKE_BUILD_TYPE})

if(WITH_OPENMP)
  find_package(OpenMP)
  if(OPENMP_FOUND)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  else()
    message(WARNING "OpenMP not found: Compilation will not use OpenMP")
  endif()
endif()

if(ANDROID)
  set(LIB_ONLY ON)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  if(ANDROID_STL MATCHES "gnustl")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DANDROID_GNUSTL_COMPAT")
  endif()
endif()

if(MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /Wall")
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
endif()

add_subdirectory(lib/TH)
add_subdirectory(lib/tokenizer)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")

if (EIGEN_ROOT)
  set(ENV{EIGEN3_ROOT} ${EIGEN_ROOT})
endif()

find_package(Eigen3 3.3 REQUIRED)
find_package(CUDA 6.5 QUIET)

set(INCLUDE_DIRECTORIES
  ${CMAKE_CURRENT_SOURCE_DIR}/lib
  ${CMAKE_CURRENT_SOURCE_DIR}/lib/tokenizer/include
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  ${EIGEN3_INCLUDE_DIR}
  )
set(SOURCES
  src/th/Env.cc
  src/th/Obj.cc
  src/th/Utils.cc
  src/Dictionary.cc
  src/PhraseTable.cc
  src/Profiler.cc
  src/ITranslator.cc
  src/TranslatorFactory.cc
  src/TranslationResult.cc
  src/Threads.cc
  )

if (CUDA_FOUND)
  add_definitions(-DWITH_CUDA)
  cuda_add_library(${PROJECT_NAME}
    ${SOURCES}
    src/cuda/Utils.cc
    src/cuda/Kernels.cu
    )
  cuda_add_cublas_to_target(${PROJECT_NAME})
else()
  add_library(${PROJECT_NAME} ${SOURCES})
endif()

target_link_libraries(${PROJECT_NAME}
  OpenNMTTokenizer
  TH)

target_include_directories(${PROJECT_NAME} PUBLIC ${INCLUDE_DIRECTORIES})

if (NOT LIB_ONLY)
  add_subdirectory(cli)
endif()

install(
  TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION lib/
  LIBRARY DESTINATION lib/
  )
install(
  DIRECTORY include/onmt
  DESTINATION include
  FILES_MATCHING PATTERN "*.h*"
  )
