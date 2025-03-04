<div align="center"> <h1>Torch-Pruning <br> <h3>Towards Any Structural Pruning<h3> </h1> </div>
<div align="center">
<img src="assets/intro.png" width="45%">
</div>

Torch-Pruning (TP) is a versatile library that enables structural network pruning for a wide range of neural networks, including Vision Transformers, FasterRCNN, SSD, ResNet, DenseNet, RegNet, ResNext, FCN, DeepLab, VGG, and more. Unlike [torch.nn.utils.prune](https://pytorch.org/tutorials/intermediate/pruning_tutorial.html) that only zeroizes parameters through masking, Torch-Pruning employs a (non-deep) graph algorithm called DepGraph to physically remove coupled parameters (channels) from models. To explore more prunable models, please refer to [benchmarks/prunability](https://github.com/VainF/Torch-Pruning/tree/master/benchmarks/prunability). So far, TP is compatible with **73/85=85.8%** models from Torchvision 0.13.1. Besides, a  [resource list](https://github.com/VainF/Torch-Pruning/blob/master/awesome_structural_pruning.md) for practical structural pruning is continuesly being updated.

For more technical details, please refer to our preprint paper: 

> [**DepGraph: Towards Any Structural Pruning**](https://arxiv.org/abs/2301.12900)   
> [Gongfan Fang](https://fangggf.github.io/), [Xinyin Ma](https://horseee.github.io/), [Mingli Song](https://person.zju.edu.cn/en/msong), [Michael Bi Mi](https://dblp.org/pid/317/0937.html), [Xinchao Wang](https://sites.google.com/site/sitexinchaowang/)   

Please do not hesitate to open a [discussion](https://github.com/VainF/Torch-Pruning/discussions) or [issue](https://github.com/VainF/Torch-Pruning/issues) if you encounter any problems with the library or have any questions related to the paper. We are always happy to assist you and address any concerns you may have. 

### **Features:**
* Structural (Channel) pruning for [CNNs](tests/test_torchvision_models.py) (e.g. ResNet, DenseNet, Deeplab) and [Transformers](tests/test_torchvision_models.py) (e.g. ViT)
* High-level pruners: [MagnitudePruner](https://arxiv.org/abs/1608.08710), [BNScalePruner](https://arxiv.org/abs/1708.06519), [GroupPruner](https://arxiv.org/abs/2301.12900) (a simple pruner used in our paper), RandomPruner, etc.
* Graph tracing and dependency modeling.
* Supported modules: Conv, Linear, BatchNorm, LayerNorm, Transposed Conv, PReLU, Embedding, MultiheadAttention, nn.Parameters and [customized modules](tests/test_customized_layer.py).
* Supported operations: split, concatenation, skip connection, flatten, all element-wise ops, etc.
* [Low-level pruning functions](https://github.com/VainF/Torch-Pruning/blob/master/torch_pruning/pruner/function.py)
* [Benchmarks](benchmarks) and [tutorials](tutorials)
* A [resource list](https://github.com/VainF/Torch-Pruning/blob/master/awesome_structural_pruning.md) for structrual pruning.

### **Plans:**
**We have a wealth of ideas, but unfortunately, only a handful of contributors at the moment. We hope to attract more talented guys to join us in bringing these ideas to fruition and making Torch-Pruning a practical library.**
* A benchmark for [Torchvision](https://pytorch.org/vision/stable/models.html) compatibility (**73/85=85.8**, :heavy_check_mark:) and [timm](https://github.com/huggingface/pytorch-image-models) compatibility.
* GANs and Detectors (We are working on the pruning of YOLO series)
* Pruning from Scratch / at Initialization.
* More high-level pruners like [FisherPruner](https://arxiv.org/abs/2108.00708), [GrowingReg](https://arxiv.org/abs/2012.09243), etc.
* More standard layers: GroupNorm, InstanceNorm, Shuffle Layers, etc.
* More Transformers like Vision Transformers (:heavy_check_mark:), Swin Transformers, PoolFormers.
* Examples for GNNs and RNNs.
* Pruning benchmarks for CIFAR, ImageNet and COCO.
* Block/Layer/Depth Pruning

## Installation
```bash
pip install torch-pruning # v1.1.0
```
or
```bash
git clone https://github.com/VainF/Torch-Pruning.git # recommended
```

## Quickstart
  
Here we provide a quick start for Torch-Pruning. More explained details can be found in [tutorals](./tutorials/)

### 0. How it works

In complex network structures, dependencies can arise among groups of parameters, necessitating their simultaneous pruning. Our work addresses this challenge by providing an automated mechanism for grouping parameters to facilitate their efficient removal for acceleration. Specifically, Torch-Pruning accomplishes this by forwarding your model with a fake input, tracing the network to establish a graph, and recording the dependencies between layers. When you prune a single layer, Torch-Pruning identifies and groups all coupled layers by returning a `Group`. Moreover, all pruning indices will be automatically transformed and aligned if operations like torch.split or torch.cat are present. 

<div align="center">
<img src="assets/dep.png" width="100%">
</div>

With DepGraph, it is easy to design some "group-level" criteria to estimate the importance of a whole group rather than a single layer. In our paper, we craft a simple [GroupPruner](https://github.com/VainF/Torch-Pruning/blob/745f6d6bafba7432474421a8c1e5ce3aad25a5ef/torch_pruning/pruner/algorithms/group_norm_pruner.py#L8) (c) to learn consistent sparsity across coupled layers.

<div align="center">
<img src="assets/group_sparsity.png" width="80%">
</div>


### 1. A minimal example

```python
import torch
from torchvision.models import resnet18
import torch_pruning as tp

model = resnet18(pretrained=True).eval()

# 1. build dependency graph for resnet18
DG = tp.DependencyGraph()
DG.build_dependency(model, example_inputs=torch.randn(1,3,224,224))

# 2. Specify the to-be-pruned channels. Here we prune those channels indexed by [2, 6, 9].
pruning_idxs = [2, 6, 9]
pruning_group = DG.get_pruning_group( model.conv1, tp.prune_conv_out_channels, idxs=pruning_idxs )

print(pruning_group.details())  # or print(pruning_group)

# 3. prune all grouped layers that are coupled with model.conv1 (included).
if DG.check_pruning_group(pruning_group): # avoid full pruning, i.e., channels=0.
    pruning_group.prune()

# 4. save & load the pruned model 
torch.save(model, 'model.pth') # save the model object
model_loaded = torch.load('model.pth') # no load_state_dict
```
  
This example demonstrates the fundamental pruning pipeline using DepGraph. Note that resnet.conv1 is coupled with several layers. Let's print the resulting group and observe how a pruning operation "triggers" other ones. In the following example, ``A => B`` means the pruning operation ``A`` triggers the pruning operation ``B``. group[0] refers to the pruning root specified by ``DG.get_pruning_group``.

```
--------------------------------
          Pruning Group
--------------------------------
[0] prune_out_channels on conv1 (Conv2d(3, 64, kernel_size=(7, 7), stride=(2, 2), padding=(3, 3), bias=False)) => prune_out_channels on conv1 (Conv2d(3, 64, kernel_size=(7, 7), stride=(2, 2), padding=(3, 3), bias=False)), idxs=[2, 6, 9] (Pruning Root)
[1] prune_out_channels on conv1 (Conv2d(3, 64, kernel_size=(7, 7), stride=(2, 2), padding=(3, 3), bias=False)) => prune_out_channels on bn1 (BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)), idxs=[2, 6, 9]
[2] prune_out_channels on bn1 (BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)) => prune_out_channels on _ElementWiseOp(ReluBackward0), idxs=[2, 6, 9]
[3] prune_out_channels on _ElementWiseOp(ReluBackward0) => prune_out_channels on _ElementWiseOp(MaxPool2DWithIndicesBackward0), idxs=[2, 6, 9]
[4] prune_out_channels on _ElementWiseOp(MaxPool2DWithIndicesBackward0) => prune_out_channels on _ElementWiseOp(AddBackward0), idxs=[2, 6, 9]
[5] prune_out_channels on _ElementWiseOp(MaxPool2DWithIndicesBackward0) => prune_in_channels on layer1.0.conv1 (Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)), idxs=[2, 6, 9]
[6] prune_out_channels on _ElementWiseOp(AddBackward0) => prune_out_channels on layer1.0.bn2 (BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)), idxs=[2, 6, 9]
[7] prune_out_channels on _ElementWiseOp(AddBackward0) => prune_out_channels on _ElementWiseOp(ReluBackward0), idxs=[2, 6, 9]
[8] prune_out_channels on _ElementWiseOp(ReluBackward0) => prune_out_channels on _ElementWiseOp(AddBackward0), idxs=[2, 6, 9]
[9] prune_out_channels on _ElementWiseOp(ReluBackward0) => prune_in_channels on layer1.1.conv1 (Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)), idxs=[2, 6, 9]
[10] prune_out_channels on _ElementWiseOp(AddBackward0) => prune_out_channels on layer1.1.bn2 (BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)), idxs=[2, 6, 9]
[11] prune_out_channels on _ElementWiseOp(AddBackward0) => prune_out_channels on _ElementWiseOp(ReluBackward0), idxs=[2, 6, 9]
[12] prune_out_channels on _ElementWiseOp(ReluBackward0) => prune_in_channels on layer2.0.downsample.0 (Conv2d(64, 128, kernel_size=(1, 1), stride=(2, 2), bias=False)), idxs=[2, 6, 9]
[13] prune_out_channels on _ElementWiseOp(ReluBackward0) => prune_in_channels on layer2.0.conv1 (Conv2d(64, 128, kernel_size=(3, 3), stride=(2, 2), padding=(1, 1), bias=False)), idxs=[2, 6, 9]
[14] prune_out_channels on layer1.1.bn2 (BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)) => prune_out_channels on layer1.1.conv2 (Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)), idxs=[2, 6, 9]
[15] prune_out_channels on layer1.0.bn2 (BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)) => prune_out_channels on layer1.0.conv2 (Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)), idxs=[2, 6, 9]
--------------------------------
```
For more details about grouping, please refer to [tutorials/2 - Exploring Dependency Groups](https://github.com/VainF/Torch-Pruning/blob/master/tutorials/2%20-%20Exploring%20Dependency%20Groups.ipynb)

#### How to scan all groups:
Just like what we do in the [MetaPruner](https://github.com/VainF/Torch-Pruning/blob/b607ae3aa61b9dafe19d2c2364f7e4984983afbf/torch_pruning/pruner/algorithms/metapruner.py#L197), one can use ``DG.get_all_groups(ignored_layers, root_module_types)`` to iterate all groups. Specifically, each group will begin with a layer that matches the type specified by the "root_module_types" parameter. These groups contain a full index list ``idxs=[0,1,2,3,...,K]`` that covers all prunable parameters. If you are intended to prune partial channels/dimensions, you can use ``group.prune(idxs=idxs)``.

```python
for group in DG.get_all_groups(ignored_layers=[model.conv1], root_module_types=[nn.Conv2d, nn.Linear]):
    # handle groups in sequential order
    idxs = [2,4,6] # your pruning indices
    group.prune(idxs=idxs)
    print(group)
```



### 2. High-level Pruners

Leveraging the DependencyGraph, we developed several high-level pruners in this repository to facilitate effortless pruning. By specifying the desired channel sparsity, you can prune the entire model and fine-tune it using your own training code. For detailed information on this process, we encourage you to consult the [this tutorial](https://github.com/VainF/Torch-Pruning/blob/master/tutorials/1%20-%20Customize%20Your%20Own%20Pruners.ipynb). Additionally, you can find more practical examples in [benchmarks/main.py](benchmarks/main.py).

```python
import torch
from torchvision.models import resnet18
import torch_pruning as tp

model = resnet18(pretrained=True)

# Importance criteria
example_inputs = torch.randn(1, 3, 224, 224)
imp = tp.importance.MagnitudeImportance(p=2)

ignored_layers = []
for m in model.modules():
    if isinstance(m, torch.nn.Linear) and m.out_features == 1000:
        ignored_layers.append(m) # DO NOT prune the final classifier!

iterative_steps = 5 # progressive pruning
pruner = tp.pruner.MagnitudePruner(
    model,
    example_inputs,
    importance=imp,
    iterative_steps=iterative_steps,
    ch_sparsity=0.5, # remove 50% channels, ResNet18 = {64, 128, 256, 512} => ResNet18_Half = {32, 64, 128, 256}
    ignored_layers=ignored_layers,
)

base_macs, base_nparams = tp.utils.count_ops_and_params(model, example_inputs)
for i in range(iterative_steps):
    pruner.step()
    macs, nparams = tp.utils.count_ops_and_params(model, example_inputs)
    # finetune your model here
    # finetune(model)
    # ...
```

#### Sparse Training
Some pruners like [BNScalePruner](https://github.com/VainF/Torch-Pruning/blob/dd59921365d72acb2857d3d74f75c03e477060fb/torch_pruning/pruner/algorithms/batchnorm_scale_pruner.py#L45) and [GroupNormPruner](https://github.com/VainF/Torch-Pruning/blob/dd59921365d72acb2857d3d74f75c03e477060fb/torch_pruning/pruner/algorithms/group_norm_pruner.py#L53) require sparse training before pruning. This can be easily achieved by inserting just one line of code ``pruner.regularize(model)`` in your training script. The pruner will update the gradient of trainable parameters.
```python
for epoch in range(epochs):
    model.train()
    for i, (data, target) in enumerate(train_loader):
        data, target = data.to(device), target.to(device)
        optimizer.zero_grad()
        out = model(data)
        loss = F.cross_entropy(out, target)
        loss.backward()
        pruner.regularize(model) # <== for sparse learning
        optimizer.step()
```

#### Interactive Pruning
All high-level pruners support interactive pruning. You can use ``pruner.step(interactive=True)`` to get all groups and interactively prune them by calling ``group.prune()``. This feature is useful if you want to control/monitor the pruning process.

```python
for i in range(iterative_steps):
    for group in pruner.step(interactive=True): # Warning: groups must be handled sequentially. Do not keep them as a list.
        print(group) 
        # do whatever you like with the group 
        # ...
        group.prune() # you should manually call the group.prune()
        # group.prune(idxs=[0, 2, 6]) # you can even change the pruning behaviour with the idxs parameter
    macs, nparams = tp.utils.count_ops_and_params(model, example_inputs)
    # finetune your model here
    # finetune(model)
    # ...
```



### 3. Low-level pruning functions

While it is possible to manually prune your model using low-level functions, this approach can be quite laborious, as it requires careful management of the associated dependencies. As a result, we recommend utilizing the aforementioned high-level pruners to streamline the pruning process.

```python
tp.prune_conv_out_channels( model.conv1, idxs=[2,6,9] )

# fix the broken dependencies manually
tp.prune_batchnorm_out_channels( model.bn1, idxs=[2,6,9] )
tp.prune_conv_in_channels( model.layer2[0].conv1, idxs=[2,6,9] )
...
```

The following pruning functions are available:
```python
tp.prune_conv_out_channels,
tp.prune_conv_in_channels,
tp.prune_depthwise_conv_out_channels,
tp.prune_depthwise_conv_in_channels,
tp.prune_batchnorm_out_channels,
tp.prune_batchnorm_in_channels,
tp.prune_linear_out_channels,
tp.prune_linear_in_channels,
tp.prune_prelu_out_channels,
tp.prune_prelu_in_channels,
tp.prune_layernorm_out_channels,
tp.prune_layernorm_in_channels,
tp.prune_embedding_out_channels,
tp.prune_embedding_in_channels,
tp.prune_parameter_out_channels,
tp.prune_parameter_in_channels,
tp.prune_multihead_attention_out_channels,
tp.prune_multihead_attention_in_channels,
```

### 4. Customized Layers

Please refer to [tests/test_customized_layer.py](https://github.com/VainF/Torch-Pruning/blob/master/tests/test_customized_layer.py).

### 5. Benchmarks

Our results on {ResNet-56 / CIFAR-10 / 2.00x}

| Method | Base (%) | Pruned (%) | $\Delta$ Acc (%) | Speed Up |
|:--    |:--:  |:--:    |:--: |:--:      |
| NIPS [[1]](#1)  | -    | -      |-0.03 | 1.76x    |
| Geometric [[2]](#2) | 93.59 | 93.26 | -0.33 | 1.70x |
| Polar [[3]](#3)  | 93.80 | 93.83 | +0.03 |1.88x |
| CP  [[4]](#4)   | 92.80 | 91.80 | -1.00 |2.00x |
| AMC [[5]](#5)   | 92.80 | 91.90 | -0.90 |2.00x |
| HRank [[6]](#6) | 93.26 | 92.17 | -0.09 |2.00x |
| SFP  [[7]](#7)  | 93.59 | 93.36 | +0.23 |2.11x |
| ResRep [[8]](#8) | 93.71 | 93.71 | +0.00 |2.12x |
||
| Ours-L1 | 93.53 | 92.93 | -0.60 | 2.12x |
| Ours-BN | 93.53 | 93.29 | -0.24 | 2.12x |
| Ours-Group | 93.53 | *93.91 | +0.38 | 2.13x |

Please refer to [benchmarks](benchmarks) for more details.

## Citation
```
@article{fang2023depgraph,
  title={DepGraph: Towards Any Structural Pruning},
  author={Fang, Gongfan and Ma, Xinyin and Song, Mingli and Mi, Michael Bi and Wang, Xinchao},
  journal={The IEEE/CVF Conference on Computer Vision and Pattern Recognition},
  year={2023}
}
```
