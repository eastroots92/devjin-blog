---
title: "[번역] YOLO(v3) object detector를 Pytorch로 처음부터 구현해보기: Part2"
date: 2019-08-01
tags:
  - computer-vision
keywords:
  - yolo
  - computer-vision
  - pytorch
---

# Part 2 - 네트워크 구조의 layer들 구현하기
<br/>

> Opencv DNN, Tensorflow, Pytorch로 YOLO v3를 구현해본 코드를 보려면 [Github repo](https://github.com/dojinkimm/Object_Detection_Video_DNN_Tensorflow_Pytorch) 를 참고하세요. 

<hr/>

### 본문

[본 내용 링크](https://blog.paperspace.com/how-to-implement-a-yolo-v3-object-detector-from-scratch-in-pytorch-part-2/)

지난 [Part 1](https://devjin-blog.com/yolo-part1/)에서는 YOLO가 어떻게 작동되는지 배웠고, 이번 Part 2에서는 Pytorch로 YOLO에 사용되는 layer들을 구현할 서이다. 쉽기 얘기하면, 이번 Part는 모델의 기초를 쌓는 시간이다. 

해당 코드는 `Python 3.5`, `Pytorch 0.4` 에서 실행되게끔 디자인이 되었고, 이 [Github repo](https://github.com/ayooshkathuria/YOLO_v3_tutorial_from_scratch)에서 코드들을 볼 수 있다.

튜토리얼은 총 5 단계로 나뉘어져 있다:

&nbsp;1. [Part 1 : YOLO가 어떻게 작동되는지 이해하기](https://devjin-blog.com/yolo-part1/)<br/>
&nbsp;2. Part 2 : 네트워크 구조의 layers들 구현하기<br/>
&nbsp;3. [Part 3 : 네트워크의 forward pass 구현하기](https://devjin-blog.com/yolo-part3/)<br/>
&nbsp;4. [Part 4 : Objectness 점수 thresholding과 non-maximum suppression(NMS)](https://devjin-blog.com/yolo-part4/)<br/>
&nbsp;5. [Part 5 : 입력과 출력 pipeline 디자인하기](https://devjin-blog.com/yolo-part5/)<br/>
<br/>


# 사전에 알아야할 지식

- Part 1 내용, 어떻게 YOLO가 작동하지는지에 대한 지식
- Pytorch 에 대한 기본 지식, `nn.Module`, `nn.Sequential`, `torch.nn` parameter class들로 커스텀 구조를 어떻게 구현하는지에 대한 지식도 포함한다.

이번 Part는 Pytorch 경험이 어느정도 있다고 가정하고 진행한다. 만약 처음 접해봤다면, 공식 튜토리얼에서 Pytorch를 한번 다뤄보고 이 post를 읽는 것을 추천한다. 

# 시작하기
<br/>

Detector를 만들 directory를 먼저 생성한다. 

그러고 나서 [darknet.py](http://darknet.py) 라는 파일을 생성한다, **Darknet은 YOLO 구조의 이름이다.** 이 파일에 YOLO 네트워크를 구현하는 코드를 작성할 것이다. [util.py](http://util.py) 라는 파일도 만들고 그 안에 다양한 helper 함수들을 구현할 굿이다. 두 파일 모두 directory에 저장을 한다. 

# **Configuration File**
<br/>

공식 코드(C로 작성됨)는 configuration file로 네트워크를 빌드한다. *Cfg* 파일은 블록단위로 네트워크의 layout을 나타낸다. Caffe를 사용해본 경험이 있다면 이 파일이 `.protxt` 와 같다고 보면 된다.

이 튜토리얼에서 네트워크를 빌드할 때 YOLO의 저자가 제공하는 공식 *cfg* 파일을 사용한다. [여기서](https://github.com/pjreddie/darknet/blob/master/cfg/yolov3.cfg) 파일을 다운 받을 수 있고 `cfg` 라는 directory를 만들어서 그 안에 넣어둔다. Linux나 Mac을 사용하고 있다면 작업하고 있는 directory에 terminal로 이동한 다음에 밑에 명령어를 작성한다.

```bash
mkdir cfg
cd cfg
wget https://raw.githubusercontent.com/pjreddie/darknet/master/cfg/yolov3.cfg
```

Configuration 파일을 열면, 밑과 같은 내용이 보여진다.

```bash
[convolutional]
batch_normalize=1
filters=64
size=3
stride=2
pad=1
activation=leaky

[convolutional]
batch_normalize=1
filters=32
size=1
stride=1
pad=1
activation=leaky

[convolutional]
batch_normalize=1
filters=64
size=3
stride=1
pad=1
activation=leaky

[shortcut]
from=-3
activation=linear
```

위 코드에는 4개의 블록이 있고, 그 중 3개는 convolutional layer들이고 하나는 shortcut layer이다. Shortcut layer는 ResNet에서 사용되는 것과 비슷한 skip connection이다. **YOLO에는 5개의 layer들이 사용된다**:

### **Convolutional**

```bash
[convolutional]
batch_normalize=1  
filters=64  
size=3  
stride=1  
pad=1  
activation=leaky
```

### **Shortcut**

```bash
[shortcut]
from=-3  
activation=linear
```

*Shortcut layer*는 skip connection이고 ResNet에서 사용되는 것과 비슷하다. `from`parameter는 `-3` 인데, 이는 shortcut layer의 결과물이 이전 layer와 shortcut layer부터 뒤에서 3번째 layer의 feature map을 더해서 얻어지는 것을 의미한다. 

### **Upsample**

```bash
[upsample]
stride=2
```

Bilinear upsampling을 사용해서 `stride` 값 만큼 이전 layer에서 feature map을 upsample 한다.  

### **Route**

```bash
[route]
layers = -4

[route]
layers = -1, 61
```

*Route* layer는 설명이 조금 필요하다. `layers` 라는 attribute(속성)을 가지고 있고 한개 혹은 2개의 값을 갖는다

`layers` attribute가 한개의 값만 가지고 있으면, 그 값으로 layer의 feature map을 인덱싱할 수 있다. 위 예시에서 layers의 값은 -4이기에, Route layer의 뒤에서 4번째 layer에서 feature map을 내놓게 된다. 

`layers` 가 2개의 값을 가지고 있으면 값으로 인덱싱 된 laye들의 feature map들을 concatenate 한 결과물을 리턴한다.  위 예시에서 layers의 값은 -1, 61이다, layer는 바로 이전 layer(-1)과 61번째 layer의 feature map들을 output으로 내놓게 된다. 

### **YOLO**

```bash
[yolo]
mask = 0,1,2
anchors = 10,13,  16,30,  33,23,  30,61,  62,45,  59,119,  116,90,  156,198,  373,326
classes=80
num=9
jitter=.3
ignore_thresh = .5
truth_thresh = 1
random=1
```

YOLO layer 는 part 1에서 언급한 Detection layer에 해당된다. `anchors` 는 9개의 anchor들을 나타내는데 이중에서 `mask` 태그의 인덱싱(코드에서는 0,1,2번째 anchors들)에 해당하는 anchor들만 사용된다. 3개의 다른 scale detection layer을 갖고 있기 때문에 총 9개의 anchors가 된다.

### **Net**

```bash
[net]
# Testing
batch=1
subdivisions=1
# Training
# batch=64
# subdivisions=16
width= 320
height = 320
channels=3
momentum=0.9
decay=0.0005
angle=0
saturation = 1.5
exposure = 1.5
hue=.1
```

Cfg에는 `net` 이라고 불리는 블록이 있다. 하지만 이 블록은 네트워크 입력과 training parameters만 나타내기 때문에 layer라고 부르기에는 부적합하다. YOLO forward pass에서 사용되지 않는다.하지만, forward pass시에 anchors를 수정하는데 사용되는 네트워크 입력 크기와 같은 정보들을 제공한다. 

# **Parsing the configuration file**

처음에 `darknet.py`  위에 코드 작성에 필요한 패키지들을 import 해와야 한다. 

```python
from __future__ import division

import torch 
import torch.nn as nn
import torch.nn.functional as F 
from torch.autograd import Variable
import numpy as np
```

다음으로는 `parse_cfg` 라는 함수를 정의할 건데, 이 함수는 configuration file을 입력으로 받는다. 

```python
def parse_cfg(cfgfile):
    """
    Configuration 파일을 인자로 받는다 
        
    Blocks 리스트를 리턴한다. 각 block은 neural network를 어떻게 빌드하는지에 대한 
    나타낸다. Blocks는 dictionary들의 리스트이다
    
    """
```

이 함수의 목적은 cfg를 파싱을 해서 **모든 block을 dict 형식으로 저장**하는 것이다. Block들의 attribute들과 value(값)들은 dictionary에 key-value 형식으로 저장된다. 코드에서는 cfg를 파싱하면서 이러한 dicts들을  `block` 변수에 저장하고 `blocks` 라는 리스트 변수에 dicts들을 추가한다. 함수는 리스트 변수를 리턴한다. 

우리는 cfg 파일의 내용을 string 리스트에 저장하는 것으로 시작한다. 밑에는 리스트를 preprocessing(전처리)하는 코드이다. 

```python
file = open(cfgfile, 'r')
lines = file.read().split('\n')                        # list로 lines에 저장한다
lines = [x for x in lines if len(x) > 0]               # 비어있는 line들 제거한다 
lines = [x for x in lines if x[0] != '#']              # 주석들 제거한다
lines = [x.rstrip().lstrip() for x in lines]           # whitespaces 제거한다
```

그 다음에는, string 리스트를 loop으로 돌면서 blocks에 위에서 말한 형식으로 데이터를 옮긴다. 

```python
block = {}
blocks = []

for line in lines:
    if line[0] == "[":               # 새로운 block의 시작
        if len(block) != 0:          # block이 비어있지 않다면, 이전 block뒤에 값을 추가한다
            blocks.append(block)     # blcok list에 추가한다 
            block = {}               # block 다시 비운다
        block["type"] = line[1:-1].rstrip()     
    else:
        ckey,value = line.split("=") 
        block[key.rstrip()] = value.lstrip()
blocks.append(block)

return blocks
```

# **Creating the building blocks**

이제는 `parse_cfg` 에서 리턴된 list로 Pytorch module을 만들 것이다. 

위에서 언급했듯이, 리스트에는 5 종류의 layer들이 있다. Pytorch는  `convolutional` 과 `upsample` 종류에 대해 pre-built(이미 만들어진) layer들을 제공한다. 나머지 layer에 대해서는 `nn.Module` class를 사용해서 커스텀 module을 만들 것이다. 

`create_modules` 함수는 `parse_cfg` 함수에서 리턴받은 `blocks` 리스트를 인자로 받는다. 

```python
def create_modules(blocks):
    net_info = blocks[0]     # pre-processing(전처리)와 입력에 관한 정보 저장한다  
    module_list = nn.ModuleList()
    prev_filters = 3
    output_filters = []
```


Blocks리스트를 loop으로 순회하기 전에 네트워크의 정보를 `net_info` 라는 변수에 저장한다.  

### **nn.ModuleList**

이 함수는 `nn.ModuleList` 를 리턴할 것이다. 이 class는 `nn.Module` object를 담은 일반 리스트에 거의 흡사하다. 다만, `nn.Module` object를 `nn.ModuleList` 에 추가를 할 때 (예를 들어, 우리 네트워크에 module을 추가할 때), `nn.ModuleList` 안에 있는 `nn.Module` object(modules)들의 모든 `parameter` 들이 `nn.Module` object의 `parameter` 로 추가된다. 

새로운 convolutional layer를 정의 할 때, 반드시 kernel의 dimension을 정의해야 한다. Cfg 파일에서 kernel의 높이와 너비를 알 수 있고, kernel의 깊이(feature map의 깊이)는 이전 layer의 filter의 수에서 얻을 수 있다. 이는 **convolutional layer가 적용되는 layer의 filter 수**를 추적해야 한다는 것을 의미한다. `prev_filter` 변수로 해당 작업을 진행한다. 이미지가 RGB channel에 대응되는 3개의 filter를 갖고 있기 때문에 이 변수를 처음에 3으로 초기화한다.

Route layer는 이전 layer들에서 feature maps를 가지고 온다(concat되있을 가능성이 높다). Route layer 바로 앞에 convolutional layer가 있는 경우에 kernel은 route layer가 가지고 온 이전 layer들의 feature map들에 적용이 된다. 따라서, 이전 layer의 filter 수 뿐만 아니라 각 layer의 선행하는 layer들을 추적해야 한다. Iterate(순회)를 하면서 각 block의 output filter의 수를 `output_filters` 에 추가한다.

이제, blocks 리스트를 iterate 하면서 각 block에 대해 Pytorch module을 만들 것이다. 

```python
    for index, x in enumerate(blocks[1:]):
        module = nn.Sequential()

        # block의 type을 확인한다 
        # block에 대해 새로운 module을 만든다
        # module_list에 추가한다 
```

`nn.Sequential` class는 `nn.Module` object들을 sequentially(순차적으로) 실행시키는데 사용된다. Cfg를 보면 block은 하나 이상의 layer를 가질 수 있음을 알 수 있다. 예를 들어, `convolutional` type을 가진 채block은 convolutional layer외에도 batch norm layer와 leaky ReLU activation layer를 지닌다. nn.Sequential과 add_module 함수로 우리는 이 layer들을 묶는다. 밑에 코드는 convolutional과 upsample layer를 만드는 방법이다. 

```python
        if (x["type"] == "convolutional"):
            # layer의 정보를 받아온다
            activation = x["activation"]
            try:
                batch_normalize = int(x["batch_normalize"])
                bias = False
            except:
                batch_normalize = 0
                bias = True

            filters= int(x["filters"])
            padding = int(x["pad"])
            kernel_size = int(x["size"])
            stride = int(x["stride"])

            if padding:
                pad = (kernel_size - 1) // 2
            else:
                pad = 0

            # convolutional layer을 추가한다 
            conv = nn.Conv2d(prev_filters, filters, kernel_size, stride, pad, bias = bias)
            module.add_module("conv_{0}".format(index), conv)

            # batch norm layer을 추가한다
            if batch_normalize:
                bn = nn.BatchNorm2d(filters)
                module.add_module("batch_norm_{0}".format(index), bn)

            # activation을 확인한다
            # YOLO는 Linear 혹은 Leaky ReLU이다
            if activation == "leaky":
                activn = nn.LeakyReLU(0.1, inplace = True)
                module.add_module("leaky_{0}".format(index), activn)
    
        # 만약 upsampling layer이면 
        # Bilinear2dUpsampling을 사용한다
        elif (x["type"] == "upsample"):
            stride = int(x["stride"])
            upsample = nn.Upsample(scale_factor = 2, mode = "bilinear")
            module.add_module("upsample_{}".format(index), upsample)
```

### **Route Layer / Shortcut Layers**

다음은 Route와 Shortcut Layer들을 만드는 코드이다.

```python
        # route layer일 때
        elif (x["type"] == "route"):
            x["layers"] = x["layers"].split(',')
            #Start  of a route
            start = int(x["layers"][0])
            #end, if there exists one.
            try:
                end = int(x["layers"][1])
            except:
                end = 0
            #Positive anotation
            if start > 0: 
                start = start - index
            if end > 0:
                end = end - index
            route = EmptyLayer()
            module.add_module("route_{0}".format(index), route)
            if end < 0:
                filters = output_filters[index + start] + output_filters[index + end]
            else:
                filters= output_filters[index + start]

        # shortcut은 skip connection에 대응된다
        elif x["type"] == "shortcut":
            shortcut = EmptyLayer()
            module.add_module("shortcut_{}".format(index), shortcut)
```

Route layer을 만드는 코드는 설명이 좀 필요한데, 일단 처음에 `layers` attribute의 value(값)들을 추출하고 integer(정수)로 cast한 다음에 리스트에 저장한다. 

다음에는 `EmptyLayer` 이름의 새로운 layer가 오는데 말 그대로 빈 layer이다.

```python
route = EmptyLayer()
```

밑과 같이 정의되어있다.

```python
class EmptyLayer(nn.Module):
    def __init__(self):
        super(EmptyLayer, self).__init__()
```

### **Wait, an empty layer?**

Empty layer가 아무 역할도 없기 때문에 왜 존재하는지 의아할 수 있다. Route Layer, 여느 다른 layer들 처럼 작업(previous layer 앞으로 가져오기/concatenation)을 수행한다. Pytorch에서 새로운 layer를 정의 할 때 `nn.Module` 을 하위 class로 만들고 `nn.Module` object의 `forward` 함수에 layer가 수행하는 작업들을 작성한다. 

Route 블록을 위한 layer를 디자인하기 위해 멤버로서 attribute `layers` 의 값으로 초기화된 `nn.Module` object를 만들어야 한다. 그 다음에 `forward` 함수에 concatenate/feature map 앞으로 가져오는 작업에 대한 코드를 작성할 수 있다. 마지막으로, 네트워크의 `forward` 함수에서 해당 layer를 실행한다. 

그러나 주어진 concatenate된 코드는 간결하다 (feature map에 `[torch.cat](http://torch.cat)` 호출). 위와 같이 layer를 디자인하는 것은 불필요한 추상화를 발생시켜서 boiler plate code만 증가시킨다. 이렇게 하는 대신에, route layer자리에 dummy layer을 두게 되면 darknet을 나타내는 `nn.Module` object의 `forward` 함수에서 바로 concatenate을 할 수 있게 된다. (이 마지막 문장이 잘 이해가 안된다면 `nn.Module` class가 Pytorch에서 어떻게 사용되는지 배우고 오는 것을 추천한다)

Route layer 바로 앞에 있는 convolutional layer는 자신의 kernel(concatenate 되었을 확률이 있다)을 이전 layer의 feature map에 적용한다. 밑에 코드는 route layer에서 나온 filter의 수를 담도록 `fliters` 변수를 업데이트한다. 

```python
if end < 0:
    #If we are concatenating maps
    filters = output_filters[index + start] + output_filters[index + end]
else:
    filters= output_filters[index + start]
```

Shortcut layer도 간단한 작업(addition)을 수행하기에 empty layer을 사용한다. 이 때 단지 이전 layer의 feature map에 바로 뒤 layer의 feature map을 더하는 작업을 하기 때문에 딱히 `filters` 변수를 업데이트할 필요가 없다.

### **YOLO Layer**

마지막으로 YOLO layer을 만드는 코드를 작성할 것이다.

```python
        #Yolo is the detection layer
        elif x["type"] == "yolo":
            mask = x["mask"].split(",")
            mask = [int(x) for x in mask]

            anchors = x["anchors"].split(",")
            anchors = [int(a) for a in anchors]
            anchors = [(anchors[i], anchors[i+1]) for i in range(0, len(anchors),2)]
            anchors = [anchors[i] for i in mask]

            detection = DetectionLayer(anchors)
            module.add_module("Detection_{}".format(index), detection)
```

Bounding box를 detect하는데 사용되는 anchors들을 가지고 있는 새로운 `DetectionLayer` 라는 layer를 정의한다. 

Detection layer의 정의는 다음과 같다.

```python
class DetectionLayer(nn.Module):
    def __init__(self, anchors):
        super(DetectionLayer, self).__init__()
        self.anchors = anchors
```

Loop의 끝에는 저장을 한다.

```python
        module_list.append(module)
        prev_filters = filters
        output_filters.append(filters)
```

이걸로 loop의 bodys는 끝이 난다. `create_modules` 함수 맨 뒤에 `net_info` 와 `module_list` 를 tuple로 만들어서 리턴한다

```python
return (net_info, module_list)
```

# **Testing the code**

`darknet.py` 끝에 다음과 같은 코드를 추가하고 run을 하면 코드를 test할 수 있다.

```python
blocks = parse_cfg("cfg/yolov3.cfg")
print(create_modules(blocks))
```

밑과 같은 엄청 긴 리스트를 보게 된다 (정확히는 106개의 item).

```bash
    .
    .
    
      (9): Sequential(
         (conv_9): Conv2d (128, 64, kernel_size=(1, 1), stride=(1, 1), bias=False)
         (batch_norm_9): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True)
         (leaky_9): LeakyReLU(0.1, inplace)
       )
       (10): Sequential(
         (conv_10): Conv2d (64, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
         (batch_norm_10): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True)
         (leaky_10): LeakyReLU(0.1, inplace)
       )
       (11): Sequential(
         (shortcut_11): EmptyLayer(
         )
       )
    .
    .
    .
```

이걸로 이번 Part 2는 마무리 되었다. 다음 [Part 3](https://devjin-blog.com/yolo-part3/)에서는 이미지에서 output을 산출하기 위해 만든 building blocks들을 조립할 것이다. 

### **Further Reading**

1. [PyTorch tutorial](http://pytorch.org/tutorials/beginner/deep_learning_60min_blitz.html)
2. [nn.Module, nn.Parameter classes](http://pytorch.org/tutorials/beginner/blitz/neural_networks_tutorial.html#define-the-network)
3. [nn.ModuleList and nn.Sequential](https://discuss.pytorch.org/t/when-should-i-use-nn-modulelist-and-when-should-i-use-nn-sequential/5463)