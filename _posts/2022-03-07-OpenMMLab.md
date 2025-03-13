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
cmake -DMMDEPLOY_TARGET_BACKENDS=ort -DONNXRUNTIME_DIR=${ONNXRUNTIME_DIR} ..
make -j$(nproc)

# build MMDeploy SDK
cmake .. \
      -DMMDEPLOY_BUILD_SDK=ON \
      -DCMAKE_CXX_COMPILER=g++ \
      -Dpplcv_DIR=${MMDEPLOY_DIR}/ppl.cv/cuda-build/install/lib/cmake/ppl \
      -DONNXRUNTIME_DIR=${ONNXRUNTIME_DIR} \
      -DMMDEPLOY_TARGET_DEVICES="cpu;cuda" \
      -DMMDEPLOY_TARGET_BACKENDS=ort \
      -DMMDEPLOY_CODEBASES=mmdet
make -j$(nproc) && make install
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
make -j$(nproc) && make install
```

Seems to work.

### Model Conversion

> Once we have installed MMDetection, MMDeploy, ONNX Runtime and built plugin for ONNX Runtime, we can convert the Faster R-CNN to a .onnx model file which can be received by ONNX Runtime. Run following commands to use our deploy tools:

First we need to download the pretrained wait (`*.pth` file).

```
cd ${MMDET_DIR}
mkdir -p checkpoints
pip install openmim
mim download mmdet --config faster_rcnn_r50_fpn_1x_coco --dest ./checkpoints
```

```
# Assume you have installed MMDeploy in ${MMDEPLOY_DIR} and MMDetection in ${MMDET_DIR}
# If you do not know where to find the path. Just type `pip show mmdeploy` and `pip show mmdet` in your console.

python ${MMDEPLOY_DIR}/tools/deploy.py \
    ${MMDEPLOY_DIR}/configs/mmdet/detection/detection_onnxruntime_dynamic.py \
    ${MMDET_DIR}/configs/faster_rcnn/faster_rcnn_r50_fpn_1x_coco.py \
    ${MMDET_DIR}/checkpoints/faster_rcnn_r50_fpn_1x_coco_20200130-047c8118.pth \
    ${MMDET_DIR}/demo/demo.jpg \
    --work-dir work_dirs \
    --device cpu \
    --show \
    --dump-info
```

```
python ${MMDEPLOY_DIR}/tools/deploy.py \
    ${MMDEPLOY_DIR}/configs/mmdet/detection/detection_onnxruntime_dynamic.py \
    ./faster_rcnn_r50_fpn_1x_coco.py \
    ./faster_rcnn_r50_fpn_1x_coco_20200130-047c8118.pth \
    ${MMDET_DIR}/demo/demo.jpg \
    --work-dir work_dirs \
    --device cpu \
    --show \
    --dump-info
```

Error!

```
Traceback (most recent call last):
  File "/home/kyungho.jeon/MMDeploy/tools/deploy.py", line 277, in <module>
    main()
  File "/home/kyungho.jeon/MMDeploy/tools/deploy.py", line 86, in main
    dump_info(deploy_cfg, model_cfg, args.work_dir, pth=checkpoint_path)
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/utils/export_info.py", line 347, in dump_info
    deploy_info = get_deploy(deploy_cfg, model_cfg, work_dir=work_dir)
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/utils/export_info.py", line 269, in get_deploy
    deploy_cfg, model_cfg, work_dir=work_dir)
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/utils/export_info.py", line 58, in get_model_name_customs
    model_cfg=model_cfg, deploy_cfg=deploy_cfg, device='cpu')
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/apis/utils.py", line 21, in build_task_processor
    import_codebase(codebase_type)
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/codebase/__init__.py", line 30, in import_codebase
    importlib.import_module(f'mmdeploy.codebase.{lib}')
  File "/opt/conda/envs/openmmlab/lib/python3.7/importlib/__init__.py", line 127, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  File "<frozen importlib._bootstrap>", line 1006, in _gcd_import
  File "<frozen importlib._bootstrap>", line 983, in _find_and_load
  File "<frozen importlib._bootstrap>", line 967, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 677, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 728, in exec_module
  File "<frozen importlib._bootstrap>", line 219, in _call_with_frames_removed
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/codebase/mmdet/__init__.py", line 5, in <module>
    from .models import *  # noqa: F401,F403
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/codebase/mmdet/models/__init__.py", line 3, in <module>
    from .dense_heads import *  # noqa: F401,F403
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/codebase/mmdet/models/dense_heads/__init__.py", line 2, in <module>
    from .base_dense_head import (base_dense_head__get_bbox,
  File "/home/kyungho.jeon/MMDeploy/mmdeploy/codebase/mmdet/models/dense_heads/base_dense_head.py", line 3, in <module>
    from mmdet.core.bbox.coder import (DeltaXYWHBBoxCoder, DistancePointBBoxCoder,
  File "/home/kyungho.jeon/MMDetection/mmdet/core/__init__.py", line 3, in <module>
    from .bbox import *  # noqa: F401, F403
  File "/home/kyungho.jeon/MMDetection/mmdet/core/bbox/__init__.py", line 8, in <module>
    from .samplers import (BaseSampler, CombinedSampler,
  File "/home/kyungho.jeon/MMDetection/mmdet/core/bbox/samplers/__init__.py", line 12, in <module>
    from .score_hlr_sampler import ScoreHLRSampler
  File "/home/kyungho.jeon/MMDetection/mmdet/core/bbox/samplers/score_hlr_sampler.py", line 3, in <module>
    from mmcv.ops import nms_match
  File "/opt/conda/envs/openmmlab/lib/python3.7/site-packages/mmcv/ops/__init__.py", line 2, in <module>
    from .assign_score_withk import assign_score_withk
  File "/opt/conda/envs/openmmlab/lib/python3.7/site-packages/mmcv/ops/assign_score_withk.py", line 6, in <module>
    '_ext', ['assign_score_withk_forward', 'assign_score_withk_backward'])
  File "/opt/conda/envs/openmmlab/lib/python3.7/site-packages/mmcv/utils/ext_loader.py", line 13, in load_ext
    ext = importlib.import_module('mmcv.' + name)
  File "/opt/conda/envs/openmmlab/lib/python3.7/importlib/__init__.py", line 127, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
ImportError: /opt/conda/envs/openmmlab/lib/python3.7/site-packages/mmcv/_ext.cpython-37m-x86_64-linux-gnu.so: undefined symbol: _Z27points_in_boxes_cpu_forwardN2at6TensorES0_S0_
```

Probably problem with `LD_LIBRARY_PATH`?

```
(openmmlab) kyungho.jeon@cuda-11-0-20220307-132532:~/MMDeploy$ echo $LD_LIBRARY_PATH
/home/kyungho.jeon/onnxruntime-linux-x64-1.8.1/lib:/usr/local/cuda/lib64:/usr/local/nccl2/lib:/usr/local/cuda/extras/CUPTI/lib64
```
