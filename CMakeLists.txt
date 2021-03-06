cmake_minimum_required(VERSION 3.8.0)
project(tensorflow_catkin)

find_package(catkin_simple REQUIRED)
catkin_simple()

set(USE_GPU OFF CACHE BOOL "Build Tensorflow with GPU support")
set(CUDA_ROOT $ENV{CUDA_ROOT} CACHE PATH "CUDA root directory")
set(CUDNN_ROOT $ENV{CUDNN_ROOT} CACHE PATH "CUDNN root directory")
set(NCCL_ROOT $ENV{NCCL_ROOT} CACHE PATH "NCCL root directory")

include(ExternalProject)
file(MAKE_DIRECTORY ${CATKIN_DEVEL_PREFIX}/include)

ExternalProject_Add(tensorflow_src
  GIT_REPOSITORY https://github.com/tensorflow/tensorflow.git
  GIT_TAG b874783ccdf4cc36cb3546e6b6a998cb8f3470bb
  PATCH_COMMAND bash -c "for i in ${CMAKE_SOURCE_DIR}/patches/*.patch\$<SEMICOLON> do git apply -p1 $i\$<SEMICOLON> done"
  UPDATE_COMMAND ""
  SOURCE_DIR tensorflow_src
  SOURCE_SUBDIR tensorflow/contrib/cmake
  BINARY_DIR tensorflow_build
  CMAKE_ARGS
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_INSTALL_PREFIX=${CATKIN_DEVEL_PREFIX}
    # General TF options
    -Dtensorflow_BUILD_SHARED_LIB=ON
    -Dtensorflow_BUILD_ALL_KERNELS=ON
    -Dtensorflow_BUILD_CONTRIB_KERNELS=ON
    -Dtensorflow_ENABLE_GRPC_SUPPORT=ON  # Needed by tf_protos_cc...
    -Dtensorflow_ENABLE_SSL_SUPPORT=OFF
    -Dtensorflow_BUILD_CC_TESTS=OFF
    -Dtensorflow_BUILD_CC_EXAMPLE=OFF
    -Dtensorflow_BUILD_PYTHON_BINDINGS=ON  # Needed by tf_c_python_api...
    -DPYTHON_LIBRARY=${PYTHON_LIBRARY}  # likewise
    -Dtensorflow_BUILD_PYTHON_TESTS=OFF
    # GPU-specific options
    -Dtensorflow_ENABLE_GPU=${USE_GPU}
    -Dtensorflow_PATH_STATIC_LIB=${CUDA_ROOT}/lib64
    -Dtensorflow_CUDA_LIBRARY_PATH=${CUDA_ROOT}/lib64
    -DCUDA_CUDA_LIBRARY=${CUDA_ROOT}/lib64/stubs/libcuda.so
    -Dtensorflow_PATH_NCCL_STATIC_LIB=${NCCL_ROOT}/lib
    -Dtensorflow_CUDNN_INCLUDE=${CUDNN_ROOT}/include
    -Dtensorflow_PATH_CUDNN_STATIC_LIB=${CUDNN_ROOT}/lib64
)

cs_add_library(${PROJECT_NAME} src/dependency_tracker.cc)
add_dependencies(${PROJECT_NAME} tensorflow_src)
target_link_libraries(${PROJECT_NAME}
  ${CATKIN_DEVEL_PREFIX}/lib/libtensorflow${CMAKE_SHARED_LIBRARY_SUFFIX})

cs_install()

cs_export(
  INCLUDE_DIRS ${CATKIN_DEVEL_PREFIX}/include
CFG_EXTRAS tensorflow-extras.cmake)
