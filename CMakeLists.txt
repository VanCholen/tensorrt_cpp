cmake_minimum_required(VERSION 2.6)

project(pro LANGUAGES CUDA CXX)

option(CUDA_USE_STATIC_CUDA_RUNTIME OFF)

include(CheckLanguage)
# ref: https://cmake.org/cmake/help/latest/module/CheckLanguage.html#module:CheckLanguage
check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
  enable_language(CUDA)    # ref: https://cmake.org/cmake/help/latest/command/enable_language.html#command:enable_language
else()
  message(STATUS "No CUDA support!")
endif()


# 指定编译类型
# - Debug
# - Release
set(CMAKE_BUILD_TYPE Debug)

# C++语言层面
# - C++ 11标准
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)


# - 编译时 g++ 时所带的参数 先引用${CMAKE_CXX_FLAGS},然后在后面添加   有点类似export PATH=${PATH}:extra_path
# -std=c++11表示使用C++11标准
# -g 生成只有gdb能看懂的调试信息 常常和 -O0一起使用
# -O0 表示没有优化(默认)  -O3优化最高
# -Wall  开启“所有常见的”编译器警告 至于是哪些警告 参见 https://linux.die.net/man/1/g++
# -Wfatal-errors 告诉编译器如果编译错误，则立即停止编译
# -pthread ###################################################################
# Define additional macros required for using the POSIX threads library. 
# You should use this option consistently for both compilation and linking. 
# This option is supported on GNU/Linux targets, most other Unix derivatives, 
# and also on x86 Cygwin and MinGW targets.
# consistently for both compilation and linking.  This option is supported on GNU/Linux targets, most
# other Unix derivatives, and also on x86 Cygwin and MinGW targets.
# ############################################################################
# -w Suppress all warnings, including those which GNU CPP issues by default.
set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -g -O0 -Wfatal-errors -pthread -w")

# 设置传递给nvcc编译器的参数  好像这种设置方式已经被deprecated
# 更新式的做法应该是使用target_compile_options() 
# 先这样写着吧，后续有时间再重构
set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -std=c++11 -O0 -Xcompiler -fPIC -g -w ${CUDA_GEN_CODE}")


# ref: https://cmake.org/cmake/help/latest/variable/EXECUTABLE_OUTPUT_PATH.html?highlight=executable_output_path
# set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/workspace)
# EXECUTABLE_OUTPUT_PATH 已经被 RUNTIME_OUTPUT_DIRECTORY 变量替代
# set(RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace)
# 用 CMAKE_RUNTIME_OUTPUT_DIRECTORY 初始化 RUNTIME_OUTPUT_DIRECTORY ref: https://cmake.org/cmake/help/v3.0/variable/CMAKE_RUNTIME_OUTPUT_DIRECTORY.html?highlight=cmake_runtime_output_directory
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace)





# 自定义变量， 如果要支持python则设置python路径
set(HAS_PYTHON ON)

# set(PythonRoot "/datav/software/anaconda3")

# 设置你的python版本
set(PythonName "python3.8")

# 依赖
# - 头文件 (many .h)
# - 库文件 (many .so[Linux] or .dll[Windows])

# 1.Python支持
if("${HAS_PYTHON}" STREQUAL "ON")
    message("Usage Python ${PythonRoot}")
    # include_directories(${PythonRoot}/include/${PythonName})
    # link_directories(${PythonRoot}/lib)
    include_directories(/usr/include/${PythonName})
    link_directories(/usr/lib/x86_64-linux-gnu/)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHAS_PYTHON")
endif()

# 2.OpenCV
# set(OpenCV_DIR   "your opencv build dir")
find_package(OpenCV)

# 3.cuDNN   由于我已经将cuDNN的头文件和库文件拷贝到了cuda下 故在此不设置了
set(CUDNN_DIR    "/usr/local/cuda-11.3")

# 5.protobuf 要在tensorrt的前面  因为tensorRTlib下带有protobuf的so文件
set(PROTOBUF_DIR "/usr/local")     
include_directories(${PROTOBUF_DIR}/include)
link_directories(${PROTOBUF_DIR}/lib)

# 4.TensorRT
set(TENSORRT_DIR "/home/morvan/sdk/TensorRT-8.0.1.6")
include_directories(${TENSORRT_DIR}/include)
link_directories(${TENSORRT_DIR}/lib)

# 6. CUDA
set(CUDA_TOOLKIT_ROOT_DIR "/usr/local/cuda-11.3")
include_directories(${CUDA_TOOLKIT_ROOT_DIR}/include)
link_directories(${CUDA_TOOLKIT_ROOT_DIR}/lib64)
link_directories(${CUDA_TOOLKIT_ROOT_DIR}/targets/x86_64-linux/lib/stubs) # for find libcuda.so


# 如果你是不同显卡，请设置为显卡对应的号码参考这里：https://developer.nvidia.com/zh-cn/cuda-gpus#compute
# RTX3090 ====> 8.6 (写86)
# GeForce 940M =====> 5.0  (写50)
set(CUDA_GEN_CODE "-gencode=arch=compute_86,code=sm_86")


find_package(CUDA REQUIRED)
# find_package(OpenCV)

include_directories(
    ${PROJECT_SOURCE_DIR}/src
    ${PROJECT_SOURCE_DIR}/src/application
    ${PROJECT_SOURCE_DIR}/src/tensorRT
    ${PROJECT_SOURCE_DIR}/src/tensorRT/common
    # ${OpenCV_INCLUDE_DIRS}
    # ${CUDA_TOOLKIT_ROOT_DIR}/include
    # ${PROTOBUF_DIR}/include
    # ${TENSORRT_DIR}/include
    # ${CUDNN_DIR}/include
)

# 切记! protobuf的lib目录一定要比tensorRT目录前面，因为tensorRTlib下带有protobuf的so文件
# 这可能带来错误
# link_directories(
#     ${PROTOBUF_DIR}/lib
#     ${TENSORRT_DIR}/lib
#     ${CUDA_TOOLKIT_ROOT_DIR}/lib64
#     ${CUDNN_DIR}/lib
# )

file(GLOB_RECURSE cpp_srcs ${PROJECT_SOURCE_DIR}/src/*.cpp)
file(GLOB_RECURSE cuda_srcs ${PROJECT_SOURCE_DIR}/src/*.cu)

# cuda_add_library deprecated! 
# cuda_add_library(plugin_list SHARED ${cuda_srcs}) 
add_library(plugin_list SHARED ${cuda_srcs}) 
target_link_libraries(plugin_list nvinfer nvinfer_plugin)
target_link_libraries(plugin_list cuda cublas cudart cudnn)
target_link_libraries(plugin_list protobuf pthread)
target_link_libraries(plugin_list ${OpenCV_LIBS})

add_executable(pro ${cpp_srcs})

# 如果提示插件找不到，请使用dlopen(xxx.so, NOW)的方式手动加载可以解决插件找不到问题
target_link_libraries(pro nvinfer nvinfer_plugin)
target_link_libraries(pro cuda cublas cudart cudnn)
target_link_libraries(pro protobuf pthread plugin_list)
target_link_libraries(pro ${OpenCV_LIBS})

if("${HAS_PYTHON}" STREQUAL "ON")
    set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/example-python/pytrt)
    add_library(pytrtc SHARED ${cpp_srcs})
    target_link_libraries(pytrtc nvinfer nvinfer_plugin)
    target_link_libraries(pytrtc cuda cublas cudart cudnn)
    target_link_libraries(pytrtc protobuf pthread plugin_list)
    target_link_libraries(pytrtc ${OpenCV_LIBS})
    target_link_libraries(pytrtc "${PythonName}")
    target_link_libraries(pro "${PythonName}")
endif()

add_custom_target(
    yolo
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro yolo
)

add_custom_target(
    yolo_gpuptr
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro yolo_gpuptr
)

add_custom_target(
    yolo_fast
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro yolo_fast
)

add_custom_target(
    centernet
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro centernet
)

add_custom_target(
    alphapose 
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro alphapose
)

add_custom_target(
    retinaface
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro retinaface
)

add_custom_target(
    dbface
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro dbface
)

add_custom_target(
    arcface 
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro arcface
)

add_custom_target(
    bert 
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro bert
)

add_custom_target(
    fall
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro fall_recognize
)

add_custom_target(
    scrfd
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro scrfd
)

add_custom_target(
    lesson
    DEPENDS pro
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/workspace
    COMMAND ./pro lesson
)

add_custom_target(
    pyscrfd
    DEPENDS pytrtc
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/example-python
    COMMAND python test_scrfd.py
)

add_custom_target(
    pyinstall
    DEPENDS pytrtc
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/example-python
    COMMAND python setup.py install
)

add_custom_target(
    pytorch
    DEPENDS pytrtc
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/example-python
    COMMAND python test_torch.py
)

add_custom_target(
    pyyolov5
    DEPENDS pytrtc
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/example-python
    COMMAND python test_yolov5.py
)

add_custom_target(
    pycenternet
    DEPENDS pytrtc
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/example-python
    COMMAND python test_centernet.py
)