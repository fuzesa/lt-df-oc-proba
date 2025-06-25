cmake_minimum_required(VERSION 3.26 FATAL_ERROR) # USE ANYTHING LESS THAN 3.27!!!!
project(lt-df-oc-proba
        VERSION 1.0.0
        LANGUAGES CXX CUDA)

set(CMAKE_CXX_STANDARD 20)

find_program(SCCACHE sccache)
if(SCCACHE)
    message(STATUS "Using sccache")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${SCCACHE}")
endif()

if(UNIX) # NOTE, UNIX returns true for both Linux and macOS
    find_program(MOLD mold)
    if(MOLD)
        message(STATUS "Using mold")
        set(CMAKE_LINKER_TYPE MOLD)
    endif()
endif()

set(TLL "${TORCH_LIBRARIES};${OpenCV_LIBS};nfd;onnxruntime::onnxruntime;DataFrame;Threads::Threads")

# CUDA related settings
if(LINUX)
    message(STATUS "Linux detected")

    set(CUSPARSELT_INCLUDE_PATH "/opt/cusparselt/include")
    set(CUSPARSELT_LIBRARY_PATH "/opt/cusparselt/lib")
    
    set(nvtx3_dir "/usr/include/nvtx3")
    
    set(USE_CUDSS ON)
    
    # https://developer.nvidia.com/cuda-gpus
    set(TORCH_CUDA_ARCH_LIST "6.1")
    set(CAFFE2_USE_CUDNN ON)
    set(CAFFE2_USE_CUFILE ON)
    set(CAFFE2_USE_CUSPARSELT ON)
    
    find_package(CUDAToolkit REQUIRED)
    message(STATUS "CUDA version: " ${CUDAToolkit_VERSION})

    list(APPEND TLL CUDA::cudart;CUDA::cublas;CUDA::cusolver;CUDA::curand;CUDA::cusparse)
    
    if(SCCACHE)
        set(CMAKE_CUDA_COMPILER_LAUNCHER "${SCCACHE}")
    endif()
endif()

find_package(Torch REQUIRED)
message(STATUS "PyTorch version: " ${Torch_VERSION})

find_package(OpenCV REQUIRED)
message(STATUS "OpenCV version: " ${OpenCV_VERSION})

find_package(onnxruntime REQUIRED)
message(STATUS "ONNX Runtime version: " ${onnxruntime_VERSION})

find_package(DataFrame REQUIRED)
message(STATUS "CPP DataFrame version: " ${DataFrame_VERSION})

add_definitions(-DASSETS_DIR="${CMAKE_SOURCE_DIR}/assets/")

add_subdirectory("${CMAKE_CURRENT_LIST_DIR}/3rdparty/nfde")

add_executable(${PROJECT_NAME}
    src/util/file-util.cc
    src/util/image-util.cc
    src/util/terminal-util.cc
    src/main.cc)

target_include_directories(${PROJECT_NAME} PRIVATE ${TORCH_INCLUDE_DIRS} ${CMAKE_CURRENT_LIST_DIR}/include ${CMAKE_CURRENT_LIST_DIR}/include/util)

target_link_libraries(${PROJECT_NAME} PRIVATE ${TLL})

if(LINUX)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        CUDA_SEPARABLE_COMPILATION ON
        CUDA_ARCHITECTURES "native")
endif()