---
layout: post
title: "MMDeploy"
---

OpenMMLab provides many frameworks to solve Computer Vision problems. 
There are many project under OpenMMLab and some depend on others. 
I feel that their installation instructions are sometimes inconsistent or misleading.

In this post, I describe how I installed [MMDetection] and [MMDeploy]. The process was
performed on Ubuntu 20.04 with x86-64 CPU and NVidia T4 GPU.

What I would like to eventually do is to convert a PyTorch model created from MMDetection
to an ONNX-formatted model.

## Installation

### Prerequisite

It's always recommended to create a Python virtual environment. The official documentation
uses `conda`. 

1. Create a conda virtual environment and activate it:

```
conda create -n openmmlab python=3.7 -y
conda activate openmmlab
```

2. Install PyTorch and torchvision by following the [official instructions](https://pytorch.org/get-started/locally/).
   You may use a virtual machine or a container environment. Here, I assume I am working on a local machine with CUDA 11.3 on Linux.

```
conda install pytorch torchvision torchaudio cudatoolkit=11.3 -c pytorch
```

### Installing MMCV

[MMCV]() is the base project for most of the projects under OpenMMLab. 

The official documentation recommends to install MMDetection with MIM, but it threw an error 
when I tried to convert a model to ONNX format by MMDeploy. Thus, I try to install MMCV manually.

```
pip install mmcv-full==1.3.18 -f https://download.openmmlab.com/mmcv/dist/cu113/torch1.10.2
```

Please note that the version is specified. 
When I didn't specify the version of `mmcv-full`, the compilation failed. A resolution is to
specify an older version of mmcv. (Please see https://github.com/open-mmlab/mmcv/issues/1752)

Please keep in mind that t will take a while.

### Installing MMDetection

You may install MMDetection by `pip install mmdet`. But it would be better to install from
the source:

```
git clone -b v2.22.0 https://github.com/open-mmlab/mmdetection.git MMDetection
cd MMDetection
pip install -r requirements/build.txt
pip install -v -e .  # or "python setup.py develop"
```

Install extra dependencies...

```
# for instaboost
pip install instaboostfast
# for panoptic segmentation
pip install git+https://github.com/cocodataset/panopticapi.git
# for LVIS dataset
pip install git+https://github.com/lvis-dataset/lvis-api.git
# for albumentations
pip install -r requirements/albu.txt
```

### Installing MMDeploy

1. Download MMDeploy

```
git clone -b v0.3.0 git@github.com:open-mmlab/mmdeploy.git MMDeploy
cd MMDeploy
git submodule update --init --recursive
```

2. If you haven't install `cmake`

```
sudo apt-get install -y libssl-dev
wget https://github.com/Kitware/CMake/releases/download/v3.20.0/cmake-3.20.0.tar.gz
tar -zxvf cmake-3.20.0.tar.gz
cd cmake-3.20.0
./bootstrap
make
sudo make install
```

3. Install GCC 7+
(This is actually weird. Even if you have GCC 8 or 9, the build process throws and error because the build script seems to refer `gcc-7`)

```
# Add repository if ubuntu < 18.04
sudo add-apt-repository ppa:ubuntu-toolchain-r/test

sudo apt-get install gcc-7
sudo apt-get install g++-7
```

4. Build MMDeploy

```
cd ${MMDEPLOY_DIR}  # To mmdeploy root directory
pip install -e .
```

5. Build SDK

The [official documentation](https://mmdeploy.readthedocs.io/en/latest/build.html#build-sdk) says:

> Readers can skip this chapter if you are only interested in model converter.

But you need dependencies such as `spdlog` described in that section. Thus, it would be better to build SDK too.

```
sudo apt-get install libopencv-dev libspdlog-dev -y
```

Install openPPL (pplcv):

```
git clone git@github.com:openppl-public/ppl.cv.git
cd ppl.cv
./build.sh cuda
```

Now, you need to install onnxruntime:

```
wget https://github.com/microsoft/onnxruntime/releases/download/v1.8.1/onnxruntime-linux-x64-1.8.1.tgz

tar -zxvf onnxruntime-linux-x64-1.8.1.tgz
cd onnxruntime-linux-x64-1.8.1
export ONNXRUNTIME_DIR=$(pwd)
export LD_LIBRARY_PATH=$ONNXRUNTIME_DIR/lib:$LD_LIBRARY_PATH
```

```
cd ${MMDEPLOY_DIR} # To MMDeploy root directory
mkdir -p build && cd build

# build ONNXRuntime custom ops
#cmake -DMMDEPLOY_TARGET_BACKENDS=ort -DONNXRUNTIME_DIR=${ONNXRUNTIME_DIR} ..
#make -j$(nproc)

# build MMDeploy SDK
cmake .. \
      -DMMDEPLOY_BUILD_SDK=ON \
      -DCMAKE_CXX_COMPILER=g++-7 \
      -Dpplcv_DIR=${MMDEPLOY_DIR}/ppl.cv/cuda-build/install/lib/cmake/ppl \
      -DONNXRUNTIME_DIR=${ONNXRUNTIME_DIR} \
      -DMMDEPLOY_TARGET_DEVICES="cuda;cpu" \
      -DMMDEPLOY_TARGET_BACKENDS=ort \
      -DMMDEPLOY_CODEBASES=mmdet
cmake --build . -- -j$(nproc) && cmake --install .
```
      
Error:

```
CMake Error in csrc/preprocess/cuda/CMakeLists.txt:
  Target "mmdeploy_cuda_transform_impl" INTERFACE_INCLUDE_DIRECTORIES
  property contains path:

    "/home/kyungho.jeon/MMDeploy/ppl.cv/cuda-build/install/lib/cmake/ppl/../../../include"

  which is prefixed in the source directory.


-- Generating done
CMake Generate step failed.  Build files cannot be regenerated correctly.
```

So, I tried other instruction (from [Getting Started](https://mmdeploy.readthedocs.io/en/latest/get_started.html)):

```
cmake .. \
      -DMMDEPLOY_BUILD_SDK=ON \
      -DCMAKE_CXX_COMPILER=g++-7 \
      -DONNXRUNTIME_DIR=${ONNXRUNTIME_DIR} \
      -DMMDEPLOY_TARGET_BACKENDS=ort \
      -DMMDEPLOY_CODEBASES=mmdet
cmake --build . -- -j$(nproc) && cmake --install .
```

Now another error occurs:

```
[  1%] Building CXX object csrc/backend_ops/onnxruntime/CMakeFiles/mmdeploy_onnxruntime_ops_obj.dir/grid_sample/grid_sample.cpp.o
[  4%] Building CXX object csrc/backend_ops/onnxruntime/CMakeFiles/mmdeploy_onnxruntime_ops_obj.dir/common/ort_utils.cpp.o
[  4%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/device_impl.cpp.o
[  6%] Building CXX object csrc/backend_ops/onnxruntime/CMakeFiles/mmdeploy_onnxruntime_ops_obj.dir/modulated_deform_conv/modulated_deform_conv.cpp.o
[  7%] Building CXX object csrc/backend_ops/onnxruntime/CMakeFiles/mmdeploy_onnxruntime_ops_obj.dir/onnxruntime_register.cpp.o
[  9%] Building CXX object csrc/backend_ops/onnxruntime/CMakeFiles/mmdeploy_onnxruntime_ops_obj.dir/roi_align/roi_align.cpp.o
[ 10%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/logger.cpp.o
[ 12%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/graph.cpp.o
[ 12%] Built target mmdeploy_onnxruntime_ops_obj
[ 13%] Linking CXX shared library ../../../lib/libmmdeploy_onnxruntime_ops.so
[ 13%] Built target mmdeploy_onnxruntime_ops
[ 15%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/mat.cpp.o
[ 16%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/model.cpp.o
[ 18%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/module.cpp.o
[ 19%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/net.cpp.o
[ 21%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/operator.cpp.o
[ 22%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/status_code.cpp.o
[ 24%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/tensor.cpp.o
[ 25%] Building CXX object csrc/core/CMakeFiles/mmdeploy_core.dir/registry.cpp.o
In file included from /usr/include/spdlog/fmt/fmt.h:20:0,
                 from /usr/include/spdlog/common.h:28,
                 from /usr/include/spdlog/spdlog.h:12,
                 from /home/kyungho.jeon/MMDeploy/csrc/core/logger.h:6,
                 from /home/kyungho.jeon/MMDeploy/csrc/core/utils/formatter.h:6,
                 from /home/kyungho.jeon/MMDeploy/csrc/core/tensor.cpp:8:
/usr/include/spdlog/fmt/bundled/core.h: In instantiation of 'struct fmt::v5::formatter<mmdeploy::DataType, char, void>':
/usr/include/spdlog/fmt/bundled/core.h:622:56:   required from 'static void fmt::v5::internal::value<Context>::format_custom_arg(const void*, Context&) [with T = mmdeploy::DataType; Context = fmt::v5::basic_format_context<std::back_insert_iterator<fmt::v5::internal::basic_buffer<char> >, char>]'
/usr/include/spdlog/fmt/bundled/core.h:608:19:   required from 'fmt::v5::internal::value<Context>::value(const T&) [with T = mmdeploy::DataType; Context = fmt::v5::basic_format_context<std::back_insert_iterator<fmt::v5::internal::basic_buffer<char> >, char>]'
/usr/include/spdlog/fmt/bundled/core.h:636:58:   required from 'constexpr fmt::v5::internal::init<Context, T, TYPE>::operator fmt::v5::internal::value<Context>() const [with Context = fmt::v5::basic_format_context<std::back_insert_iterator<fmt::v5::internal::basic_buffer<char> >, char>; T = const mmdeploy::DataType&; fmt::v5::internal::type TYPE = (fmt::v5::internal::type)13]'
/usr/include/spdlog/fmt/bundled/core.h:1069:35:   required from 'typename std::enable_if<IS_PACKED, fmt::v5::internal::value<Context> >::type fmt::v5::internal::make_arg(const T&) [with bool IS_PACKED = true; Context = fmt::v5::basic_format_context<std::back_insert_iterator<fmt::v5::internal::basic_buffer<char> >, char>; T = mmdeploy::DataType; typename std::enable_if<IS_PACKED, fmt::v5::internal::value<Context> >::type = fmt::v5::internal::value<fmt::v5::basic_format_context<std::back_insert_iterator<fmt::v5::internal::basic_buffer<char> >, char> >]'
/usr/include/spdlog/fmt/bundled/core.h:1180:51:   required from 'fmt::v5::format_arg_store<Context, Args>::format_arg_store(const Args& ...) [with Context = fmt::v5::basic_format_context<std::back_insert_iterator<fmt::v5::internal::basic_buffer<char> >, char>; Args = {mmdeploy::DataType, mmdeploy::DataType}]'
/usr/include/spdlog/fmt/bundled/format.h:3257:38:   required from 'typename fmt::v5::buffer_context<Char>::type::iterator fmt::v5::format_to(fmt::v5::basic_memory_buffer<Char, SIZE>&, const S&, const Args& ...) [with S = const char*; Args = {mmdeploy::DataType, mmdeploy::DataType}; long unsigned int SIZE = 500; Char = char; typename fmt::v5::buffer_context<Char>::type::iterator = std::back_insert_iterator<fmt::v5::internal::basic_buffer<char> >]'
/usr/include/spdlog/details/logger_impl.h:72:23:   required from 'void spdlog::logger::log(spdlog::source_loc, spdlog::level::level_enum, const char*, const Args& ...) [with Args = {mmdeploy::DataType, mmdeploy::DataType}]'
/home/kyungho.jeon/MMDeploy/csrc/core/tensor.cpp:99:5:   required from here
/usr/include/spdlog/fmt/bundled/core.h:492:3: error: static assertion failed: don't know how to format the type, include fmt/ostream.h if it provides an operator<< that should be used
   static_assert(internal::no_formatter_error<T>::value,
   ^~~~~~~~~~~~~
make[2]: *** [csrc/core/CMakeFiles/mmdeploy_core.dir/build.make:202: csrc/core/CMakeFiles/mmdeploy_core.dir/tensor.cpp.o] Error 1
make[2]: *** Waiting for unfinished jobs....
make[1]: *** [CMakeFiles/Makefile2:461: csrc/core/CMakeFiles/mmdeploy_core.dir/all] Error 2
make: *** [Makefile:136: all] Error 2
```
