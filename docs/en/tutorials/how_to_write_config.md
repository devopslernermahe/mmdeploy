## How to write config

This tutorial describes how to write a config for model conversion and deployment. A deployment config includes `onnx config`, `codebase config`, `backend config`.

<!-- TOC -->

- [How to write config](#how-to-write-config)
  - [1. How to write onnx config](#1-how-to-write-onnx-config)
    - [Description of onnx config arguments](#description-of-onnx-config-arguments)
      - [Example](#example)
    - [If you need to use dynamic axes](#if-you-need-to-use-dynamic-axes)
      - [Example](#example-1)
  - [2. How to write codebase config](#2-how-to-write-codebase-config)
    - [Description of codebase config arguments](#description-of-codebase-config-arguments)
      - [Example](#example-2)
    - [If you need to use the partition model](#if-you-need-to-use-the-partition-model)
      - [Example](#example-3)
    - [List of tasks in all codebases](#list-of-tasks-in-all-codebases)
  - [3. How to write backend config](#3-how-to-write-backend-config)
    - [Example](#example-4)
  - [4. A complete example of mmcls on TensorRT](#4-a-complete-example-of-mmcls-on-tensorrt)
  - [5. The name rules of our deployment config](#5-the-name-rules-of-our-deployment-config)
    - [Example](#example-5)
  - [6. How to write model config](#6-how-to-write-model-config)
  - [7. Reminder](#7-reminder)
  - [8. FAQs](#8-faqs)

<!-- TOC -->

### 1. How to write onnx config

Onnx config to describe how to export a model from pytorch to onnx.

#### Description of onnx config arguments

- `type`: Type of config dict. Default is `onnx`.
- `export_params`: If specified, all parameters will be exported. Set this to False if you want to export an untrained model.
- `keep_initializers_as_inputs`: If True, all the initializers (typically corresponding to parameters) in the exported graph will also be added as inputs to the graph. If False, then initializers are not added as inputs to the graph, and only the non-parameter inputs are added as inputs.
- `opset_version`: Opset_version is 11 by default.
- `save_file`: Output onnx file.
- `input_names`: Names to assign to the input nodes of the graph.
- `output_names`: Names to assign to the output nodes of the graph.
- `input_shape`: The height and width of input tensor to the model.

##### Example

```python
onnx_config = dict(
    type='onnx',
    export_params=True,
    keep_initializers_as_inputs=False,
    opset_version=11,
    save_file='end2end.onnx',
    input_names=['input'],
    output_names=['output'],
    input_shape=None)
```

#### If you need to use dynamic axes

If the dynamic shape of inputs and outputs is required, you need to add dynamic_axes dict in onnx config.

- `dynamic_axes`: Describe the dimensional information about input and output.

##### Example

```python
    dynamic_axes={
        'input': {
            0: 'batch',
            2: 'height',
            3: 'width'
        },
        'dets': {
            0: 'batch',
            1: 'num_dets',
        },
        'labels': {
            0: 'batch',
            1: 'num_dets',
        },
    }
```

### 2. How to write codebase config

Codebase config part contains information like codebase type and task type.

#### Description of codebase config arguments

- `type`: Model's codebase, including `mmcls`, `mmdet`, `mmseg`, `mmocr`, `mmedit`.
- `task`: Model's task type, referring to [List of tasks in all codebases](#list-of-tasks-in-all-codebases).

##### Example

```python
codebase_config = dict(type='mmcls', task='Classification')
```

#### If you need to use the partition model

If you want to partition model , you need to add partition configuration dict. Note that currently only the MMDetection model supports partitioning.

- `type`: Model's task type, referring to -[List of tasks in all codebases](#list-of-tasks-in-all-codebases).

##### Example

```python
partition_config = dict(type='single_stage', apply_marks=True)
```

#### List of tasks in all codebases

| codebase |       task       | partition |
| :------: | :--------------: | :-------: |
|  mmcls   |  classification  |     N     |
|  mmdet   |    detection     |     Y     |
|  mmseg   |   segmentation   |     N     |
|  mmocr   |  text-detection  |     N     |
|  mmocr   | text-recognition |     N     |
|  mmedit  | supe-resolution  |     N     |

### 3. How to write backend config

The backend config is mainly used to specify the backend on which model runs and provide the information needed when the model runs on the backend , referring to [ONNX Runtime](../backends/onnxruntime.md), [TensorRT](../backends/tensorrt.md), [NCNN](../backends/ncnn.md), [PPLNN](../backends/pplnn.md).

- `type`: Model's backend, including `onnxruntime`, `ncnn`, `pplnn`, `tensorrt`, `openvino`.

#### Example

```python
backend_config = dict(
    type='tensorrt',
    common_config=dict(
        fp16_mode=False, max_workspace_size=1 << 30),
    model_inputs=[
        dict(
            input_shapes=dict(
                input=dict(
                    min_shape=[1, 3, 512, 1024],
                    opt_shape=[1, 3, 1024, 2048],
                    max_shape=[1, 3, 2048, 2048])))
    ])
```

### 4. A complete example of mmcls on TensorRT

Here we provide a complete deployment config from mmcls on TensorRT.

```python

codebase_config = dict(type='mmcls', task='Classification')

backend_config = dict(
    type='tensorrt',
    common_config=dict(
        fp16_mode=False,
        max_workspace_size=1 << 30),
    model_inputs=[
        dict(
            input_shapes=dict(
                input=dict(
                    min_shape=[1, 3, 224, 224],
                    opt_shape=[4, 3, 224, 224],
                    max_shape=[64, 3, 224, 224])))])

onnx_config = dict(
    type='onnx',
    dynamic_axes={
        'input': {
            0: 'batch',
            2: 'height',
            3: 'width'
        },
        'output': {
            0: 'batch'
        }
    },
    export_params=True,
    keep_initializers_as_inputs=False,
    opset_version=11,
    save_file='end2end.onnx',
    input_names=['input'],
    output_names=['output'],
    input_shape=[224, 224])

partition_config = None
```

### 5. The name rules of our deployment config

There is a specific naming convention for the filename of deployment config files.

```bash
(task name)_(partition)_(backend name)_(dynamic or static).py
```

- `task name`: Model's task type.
- `partition`: Optional, whether partition model is supported.
- `backend name`: Backend's name. Note if you use the quantization function, you need to indicate the quantization type. Just like `tensorrt-int8`.
- `dynamic or static`: Dynamic or static export. Note if the backend needs explicit shape information, you need to add a description of input size with `height x width` format. Just like `dynamic-512x1024-2048x2048`, it means that the min input shape is `512x1024` and the max input shape is `2048x2048`.

#### Example

```
single-stage_partition_tensorrt-int8_dynamic-320x320-1344x1344.py
```

### 6. How to write model config

According to model's codebase, write the model config file. Model's config file is used to initialize the model, referring to [MMClassification](https://github.com/open-mmlab/mmclassification/blob/master/docs/tutorials/config.md), [MMDetection](https://github.com/open-mmlab/mmdetection/blob/master/docs_zh-CN/tutorials/config.md), [MMSegmentation](https://github.com/open-mmlab/mmsegmentation/blob/master/docs_zh-CN/tutorials/config.md), [MMOCR](https://github.com/open-mmlab/mmocr/tree/main/configs), [MMEditing](https://github.com/open-mmlab/mmediting/blob/master/docs_zh-CN/config.md).

### 7. Reminder

None

### 8. FAQs

None
