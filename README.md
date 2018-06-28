# Caffe

[![Build Status](https://travis-ci.org/BVLC/caffe.svg?branch=master)](https://travis-ci.org/BVLC/caffe)
[![License](https://img.shields.io/badge/license-BSD-blue.svg)](LICENSE)

Caffe is a deep learning framework made with expression, speed, and modularity in mind.
It is developed by Berkeley AI Research ([BAIR](http://bair.berkeley.edu))/The Berkeley Vision and Learning Center (BVLC) and community contributors.

Check out the [project site](http://caffe.berkeleyvision.org) for all the details like

- [DIY Deep Learning for Vision with Caffe](https://docs.google.com/presentation/d/1UeKXVgRvvxg9OUdh_UiC5G71UMscNPlvArsWER41PsU/edit#slide=id.p)
- [Tutorial Documentation](http://caffe.berkeleyvision.org/tutorial/)
- [BAIR reference models](http://caffe.berkeleyvision.org/model_zoo.html) and the [community model zoo](https://github.com/BVLC/caffe/wiki/Model-Zoo)
- [Installation instructions](http://caffe.berkeleyvision.org/installation.html)

and step-by-step examples.

## Custom distributions

 - [Intel Caffe](https://github.com/BVLC/caffe/tree/intel) (Optimized for CPU and support for multi-node), in particular Xeon processors (HSW, BDW, SKX, Xeon Phi).
- [OpenCL Caffe](https://github.com/BVLC/caffe/tree/opencl) e.g. for AMD or Intel devices.
- [Windows Caffe](https://github.com/BVLC/caffe/tree/windows)

## Community

[![Join the chat at https://gitter.im/BVLC/caffe](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/BVLC/caffe?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Please join the [caffe-users group](https://groups.google.com/forum/#!forum/caffe-users) or [gitter chat](https://gitter.im/BVLC/caffe) to ask questions and talk about methods and models.
Framework development discussions and thorough bug reports are collected on [Issues](https://github.com/BVLC/caffe/issues).

Happy brewing!

## License and Citation

Caffe is released under the [BSD 2-Clause license](https://github.com/BVLC/caffe/blob/master/LICENSE).
The BAIR/BVLC reference models are released for unrestricted use.

Please cite Caffe in your publications if it helps your research:

    @article{jia2014caffe,
      Author = {Jia, Yangqing and Shelhamer, Evan and Donahue, Jeff and Karayev, Sergey and Long, Jonathan and Girshick, Ross and Guadarrama, Sergio and Darrell, Trevor},
      Journal = {arXiv preprint arXiv:1408.5093},
      Title = {Caffe: Convolutional Architecture for Fast Feature Embedding},
      Year = {2014}
    }
# Ristretto：面向硬件的卷积神经网络逼近
    Ristretto是一个自动化的CNN近似工具，可以压缩32位浮点网络。
    Ristretto是Caffe的扩展，允许以有限的数字精度测试、训练和微调网络。
# Ristretto速览
    1、Ristretto Tool：Ristretto工具使用不同的比特宽度进行数字表示，执行自动网络量化和评分，
       以在压缩率和网络准确度之间找到一个很好的平衡点。
    2、Ristretto Layers：Ristretto重新实现Caffe层并模拟缩短字宽的算术。
    3、测试和训练：由于将Ristretto平滑集成Caffe，可以改变网络描述文件来量化不同的层。
       不同层所使用的位宽以及其他参数可以在网络的prototxt文件中设置。
       这使得我们能够直接测试和训练压缩后的网络，而不需要重新编译。
# 逼近方案
    Ristretto允许以三种不同的量化策略来逼近卷积神经网络：
        1、动态固定点：修改的定点格式。
        2、Minifloat：缩短位宽的浮点数。
        3、两个幂参数：当在硬件中实现时，具有两个幂参数的层不需要任何乘法器。    
# Ristretto的主要扩展
    这个改进的Caffe版本支持有限数值精度层。
    所讨论的层使用缩短的字宽来表示层参数和层激活（输入和输出）。
    由于Ristretto遵循Caffe的规则，已经熟悉Caffe的用户会很快理解Ristretto。
    下面解释了Ristretto的主要扩展：   
    
## 新的层 Ristretto Layers
    Ristretto引入了新的有限数值精度层类型。
    这些层可以通过传统的Caffe网络描述文件（* .prototxt）使用。 
    下面给出一个minifloat卷积层的例子：
    
    该层将使用半精度（16位浮点）数字表示。卷积内核、偏差以及层激活都被修剪为这种格式。
```
layer {
      name: "conv1"
      type: "ConvolutionRistretto"
      bottom: "data"
      top: "conv1"
      convolution_param {
        num_output: 96
        kernel_size: 7
        stride: 2
        weight_filler {
          type: "xavier"
        }
      }
      quantization_param {
        precision: MINIFLOAT
        mant_bits: 10   # MANT:mantissa，尾数(有效数字) 
        exp_bits: 5     # 指数位数
      }
    }
```

     注意与传统卷积层的三个不同之处：
         1、type变成了ConvolutionRistretto
         2、增加了一个额外的层参数：quantization_param
         3、该层参数包含用于量化的所有信息

## 数据存储 Blobs
      Ristretto允许精确模拟资源有限的硬件加速器。
      为了与Caffe规则保持一致，Ristretto在层参数和输出中重用浮点Blob。
      这意味着有限精度数值实际上都存储在浮点数组中。
      
## 评分 Scoring
     对于量化网络的评分，Ristretto要求：
        1、训练好的32位FP网络参数
        2、网络定义降低精度的层
        
    第一项是Caffe传统训练的结果。Ristretto可以使用全精度参数来测试网络。
           这些参数默认情况下使用最接近的方案，即时转换为有限精度。
    至于第二项——模型说明——您将不得不手动更改Caffe模型的网络描述，
           或使用Ristretto工具自动生成Google Protocol Buffer文件。
```sh
# score the dynamic fixed point SqueezeNet model on the validation set*
    ./build/tools/caffe test --model=models/SqueezeNet/RistrettoDemo/quantized.prototxt \
      --weights=models/SqueezeNet/RistrettoDemo/squeezenet_finetuned.caffemodel \
      --gpu=0 --iterations=2000
```

## 网络微调 Fine-tuning

      了提高精简网络的准确性，应该对其进行微调。
      在Ristretto中，Caffe命令行工具支持精简网络微调。
      与传统训练的唯一区别是网络描述文件应该包含Ristretto层。 


      微调需要以下项目：

         1、32位FP网络参数， 网络参数是Caffe全精度训练的结果。
         2、用于训练的Solver和超参数
            解算器（solver）包含有限精度网络描述文件的路径。
            这个网络描述和我们用来评分的网络描述是一样的。
```sh
# fine-tune dynamic fixed point SqueezeNet*
    ./build/tools/caffe train \
      --solver=models/SqueezeNet/RistrettoDemo/solver_finetune.prototxt \
      --weights=models/SqueezeNet/squeezenet_v1.0.caffemodel
```

## 实施细节

       在这个再训练过程中，网络学习如何用限定字参数对图像进行分类。
       由于网络权重只能具有离散值，所以主要挑战在于权重更新。
       我们采用以前的工作（Courbariaux等1）的思想，它使用全精度因子权重。
       对32位FP权重w应用小权重更新，从中进行采样得到离散权重w'。
       微调中的采样采用 随机舍入方法(Round nearest sampling)  进行，Gupta等2人成功地使用了这种方法，
       以16位固定点来训练网络。
       
![](https://img-blog.csdn.net/20180516141818988?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpcmFuMTAz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

# Ristretto：逼近策略



