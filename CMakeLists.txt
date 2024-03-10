cmake_minimum_required(VERSION 3.18)
project(f2nerf
        VERSION 0.1
        LANGUAGES CUDA CXX)

include(CheckLanguage)
check_language(CUDA)

set(CMAKE_CXX_STANDARD 14)

###############################################################################
# CUDA compiler setup
###############################################################################
set(CMAKE_CUDA_STANDARD 14)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_EXTENSIONS OFF)
set(CUDA_LINK_LIBRARIES_KEYWORD PUBLIC)

add_definitions(-DHALF_PRECISION)

# tiny-cuda-nn
add_definitions(-DJSON_HAS_CPP_14)
set(TCNN_BUILD_BENCHMARK OFF)
set(TCNN_BUILD_EXAMPLES OFF)
add_subdirectory(External/tiny-cuda-nn)
include_directories(${TCNN_INCLUDE_DIRECTORIES})
include_directories(External/tiny-cuda-nn/include)
include_directories(External/tiny-cuda-nn/dependencies)

set(CMAKE_CUDA_ARCHITECTURES ${TCNN_CUDA_ARCHITECTURES})
set_property(GLOBAL PROPERTY CUDA_ARCHITECTURES ${TCNN_CUDA_ARCHITECTURES})

# zlib, for compiling cnpy
find_package(ZLIB REQUIRED)
include_directories(${ZLIB_INCLUDE_DIRS})

# Libtorch
if (EXISTS $ENV{HOME}/Application/anaconda3/envs/nerf-slam)
    set(CONDA_PATH "$ENV{HOME}/Application/anaconda3/envs/nerf-slam")
    set(CMAKE_PREFIX_PATH ${DCMAKE_PREFIX_PATH}$ ${CONDA_PATH}/lib/python3.9/site-packages/torch/ ${CONDA_PATH})
    find_package(Torch REQUIRED PATHS ${CONDA_PATH}/lib/python3.9/site-packages/torch/share/cmake  NO_DEFAULT_PATH)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
    link_directories(${CONDA}/lib/python3.9/site-packages/torch/)
    message(${TORCH_INCLUDE_DIRS})
    include_directories(${TORCH_INCLUDE_DIRS})
else ()
    MESSAGE("Please Rebuilt the anaconda3 environment.")
endif ()


# YAML
add_subdirectory(External/yaml-cpp)
include_directories(External/yaml-cpp/include)

# eigen
include_directories(External/eigen-3.4.0)

include(src/Field/field.cmake)
include(src/Dataset/dataset.cmake)
include(src/PtsSampler/pts_sampler.cmake)
include(src/Renderer/renderer.cmake)
include(src/Shader/shader.cmake)
include(src/Utils/utils.cmake)

list(APPEND SRC_ALL
        ${SRC_FIELD}
        ${SRC_DATASET}
        ${SRC_GEOMETRY}
        ${SRC_PTS_SAMPLER}
        ${SRC_RENDERER}
        ${SRC_SHADER}
        ${SRC_UTILS})

list(APPEND SRC_ALL src/ExpRunner.cpp)


set(ALL_LIBS ${TORCH_LIBRARIES} ${TORCH_CUDA_LIBRARIES} ${ZLIB_LIBRARIES} yaml-cpp fmt::fmt)
set(ALL_LIBS ${ALL_LIBS} tiny-cuda-nn stdc++fs)

# ------------------------------------ Add executables ---------------------------------------
add_executable(main main.cpp ${SRC_ALL})
target_link_libraries(main ${ALL_LIBS})
set_property(TARGET main PROPERTY CXX_STANDARD 17)
set_property(TARGET main PROPERTY CUDA_ARCHITECTURES ${TCNN_CUDA_ARCHITECTURES})

# Install scripts
configure_file(scripts/run.py ${CMAKE_BINARY_DIR}/run.py COPYONLY)
