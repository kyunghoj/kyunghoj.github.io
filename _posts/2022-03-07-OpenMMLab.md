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
      -DMMDEPLOY_CODEBASES=all
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


