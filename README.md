![Figure generate by visualize.py](ImageNet/weight_visual.png?raw=True)

# APoT Quantization

This repo contains the code and data of the following paper accepeted by [ICLR 2020](https://openreview.net/group?id=ICLR.cc/2020/Conference)

> [Additive Power-of-Two Quantization: An Efficient Non-uniform Discretization For Neural Networks](https://openreview.net/pdf?id=BkgXT24tDS)

**training codes** will be open sourced soon.

<p align="center">
  <img src="https://i.imgur.com/0oxm19W.png">
</p>

```latex
@inproceedings{Li2020Additive,
title={Additive Powers-of-Two Quantization: An Efficient Non-uniform Discretization for Neural Networks},
author={Yuhang Li and Xin Dong and Wei Wang},
booktitle={International Conference on Learning Representations},
year={2020},
url={https://openreview.net/forum?id=BkgXT24tDS}
}
```

## Installation

### Prerequisites

Pytorch 1.1.0 with CUDA

### Dataset Preparation

* Please prepare the ImageNet validation and training dataset, we use [official example code](https://github.com/pytorch/examples/blob/master/imagenet/main.py) here to provide dataloader. 
* The CIFAR10 dataset can be download automatically (update soon). 

## ImageNet

`models.quant_layer.py` contains the configuration for quantization. In particular, you can specify them in the class `QuantConv2d`:

```python
class QuantConv2d(nn.Conv2d):
    def __init__(self, in_channels, out_channels, kernel_size, stride=1, padding=0, dilation=1, groups=1, bias=False):
        super(QuantConv2d, self).__init__(in_channels, out_channels, kernel_size, stride, padding, dilation, groups, bias)
        self.layer_type = 'QuantConv2d'
        self.bit = 4
        self.weight_quant = weight_quantize_fn(w_bit=self.bit, power=True)
        self.act_grid = build_power_value(self.bit, additive=True)
        self.act_alq = act_quantization(self.bit, self.act_grid, power=True)
        self.act_alpha = torch.nn.Parameter(torch.tensor(8.0))
```

Here, `self.bit`  controls the bitwidth;  `weight_quantize_fn` controls the quantization scheme, where `power=True` means using PoT or APoT quantization. `build_power_value` construct the levels set Q^a(1, b) with parameter `bit` and `additive`. 

To train a 5-bit model, just run main.py:

```bash
python main.py -a resnet18 --bit 5
```

Progressive initialization requires checkpoint of higher bitwidth. For example

```bash
python main.py -a resnet18 --bit 4 --pretrained checkpoint/res18_5best.pth.tar
```

We provide a function `show_params()` to print the clipping parameter in both weights and activations



## CIFAR10

The training code is inspired by [pytorch-cifar-code](https://github.com/junyuseu/pytorch-cifar-models) from [junyuseu](https://github.com/junyuseu).

The dataset can be downloaded automatically using torchvision. We provide the shell script to progressively train full precision, 4, 3, and 2 bit models. For example, `train_res20.sh` :

``` bash
#!/usr/bin/env bash
python main.py --arch res20 --bit 32 -id 0,1 --wd 5e-4
python main.py --arch res20 --bit 4 -id 0,1 --wd 1e-4  --lr 4e-2 \
        --init result/res20_32bit/model_best.pth.tar
python main.py --arch res20 --bit 3 -id 0,1 --wd 1e-4  --lr 4e-2 \
        --init result/res20_4bit/model_best.pth.tar
python main.py --arch res20 --bit 2 -id 0,1 --wd 3e-5  --lr 4e-2 \
        --init result/res20_3bit/model_best.pth.tar
```

The checkpoint models for CIFAR10 are released: 

| Model | Precision      | Accuracy  | Checkpoints                                                  |
| :---: | -------------- | --------- | ------------------------------------------------------------ |
| Res20 | Full Precision | 92.96     | [Res20_32bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res20_32bit) |
| Res20 | 4-bit          | **92.45** | [Res20_4bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res20_4bit) |
| Res20 | 3-bit          | **92.49** | [Res20_3bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res20_3bit) |
| Res20 | 2-bit          | **90.96** | [Res20_2bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res20_2bit) |
| Res56 | Full Precision | 94.46     | [Res56_32bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res56_32bit) |
| Res56 | 4-bit          | **93.93** | [Res56_4bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res56_4bit) |
| Res56 | 3-bit          | **93.77** | [Res56_3bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res56_3bit) |
| Res56 | 2-bit          | **93.05** | [Res56_2bit](https://github.com/yhhhli/APoT_Quantization/tree/master/CIFAR10/result/res56_2bit) |

To evluate the models, you can run 

```bash
python main.py -e --init result/res20_3bit/model_best.pth.tar -e -id 0 --bit 3
```

And you will get the output of accuracy and the value of clipping threshold in weights & acts:

```bash
Test: [0/100]   Time 0.221 (0.221)      Loss 0.2144 (0.2144)    Prec 96.000% (96.000%)
 * Prec 92.510%
clipping threshold weight alpha: 1.569000, activation alpha: 1.438000
clipping threshold weight alpha: 1.278000, activation alpha: 0.966000
clipping threshold weight alpha: 1.607000, activation alpha: 1.293000
clipping threshold weight alpha: 1.426000, activation alpha: 1.055000
clipping threshold weight alpha: 1.364000, activation alpha: 1.720000
clipping threshold weight alpha: 1.511000, activation alpha: 1.434000
clipping threshold weight alpha: 1.600000, activation alpha: 2.204000
clipping threshold weight alpha: 1.552000, activation alpha: 1.530000
clipping threshold weight alpha: 0.934000, activation alpha: 1.939000
clipping threshold weight alpha: 1.427000, activation alpha: 2.232000
clipping threshold weight alpha: 1.463000, activation alpha: 1.371000
clipping threshold weight alpha: 1.440000, activation alpha: 2.432000
clipping threshold weight alpha: 1.560000, activation alpha: 1.475000
clipping threshold weight alpha: 1.605000, activation alpha: 2.462000
clipping threshold weight alpha: 1.436000, activation alpha: 1.619000
clipping threshold weight alpha: 1.292000, activation alpha: 2.147000
clipping threshold weight alpha: 1.423000, activation alpha: 2.329000
clipping threshold weight alpha: 1.428000, activation alpha: 1.551000
clipping threshold weight alpha: 1.322000, activation alpha: 2.574000
clipping threshold weight alpha: 1.687000, activation alpha: 1.314000
```



## To Do:

- checkpoints for ImageNet models