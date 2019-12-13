# 项目介绍

验证码识别 - 该项目是基于 CNN5/ResNet+BLSTM/LSTM+CTC 来实现字符型图片识别. 

# 准备工作
CUDA10+cuDNN7.3.1+TensorFlow 1.12

# 环境安装

1. 安装Python 3.6 环境（包含pip）

2. 安装虚拟环境 virtualenv ```pip3 install virtualenv```

3. 为该项目创建独立的虚拟环境:

   ```bash
   virtualenv -p /usr/bin/python3 venv # venv is the name of the virtual environment.
   cd venv/ # venv is the name of the virtual environment.
   source bin/activate # to activate the current virtual environment.
   cd captcha_trainer # captcha_trainer is the project path.
   ```

4. 安装本项目的依赖列表：```pip install -r requirements.txt```

# 开始

## 1. 架构与流程

本项目依赖于训练配置```config.yaml```和模型配置```model.yaml```，初始化项目的时候请复制```config_demo.yaml```到当前目录下命名为```config.yaml```，```model_demo.yaml```同理。或者可以使用```tutorial.py``` 自动设置模型配置。

**训练流程**：配置好两个配置文件后，执行```trains.py``` 中的代码，读取配置，根据```model.yaml```配置文件构建神经网络计算图，依据```config.yaml```的配置参数进行训练。

**关于```config.yaml```中的训练参数有几点建议：**

1. BatchSize（训练批次大小）与TestBatchSize（测试批次大小）是需要大家关注的，建议根据显卡条件进行调整，显存小的建议BatchSize不要太大，TestBatchSize也是，我提供的默认配置是基于显存8G，使用率50%设置的，请悉知。

2. LearningRate（学习率）也是需要关注的，深度学习本质就是调参，一般的模型可以保持默认的配置无需调整，有些模型想要获得更高的识别精度可以先使用0.01快速收敛，准确率差不多95%左右再使用0.001/0.0001提高精度。

3. TestSetNum（测试集数目），根据给定的测试集数目切割训练集，有一个前提，测试集必须是随机的，随机的，随机的，重要的事说三遍，有些人用Windows资源管理器打开，一拖动选择几百个，默认都是按名称排序的，如果名称是标注，那么就不是随机了，也就是很可能你取的测试集是标注为0~3之间的图片，这样可能导致永远无法收敛。

4. TrainRegex 和 TestRegex，正则匹配，请各位采集样本的时候，尽量和我给的示例保持一致吧，正则问题请谷歌，如果是为1111.jpg这种命名的话，这里提供了一个批量转换的代码：

   ```python
   import re
   import os
   import hashlib
   
   # 训练集路径
   root = r"D:\TrainSet\***"
   all_files = os.listdir(root)
   
   for file in all_files:
       old_path = os.path.join(root, file)
       
       # 已被修改过忽略
       if len(file.split(".")[0]) > 32:
           continue
       
       # 采用标注_文件md5码.图片后缀 进行命名
       with open(old_path, "rb") as f:
           _id = hashlib.md5(f.read()).hexdigest()
       new_path = os.path.join(root, file.replace(".", "_{}.".format(_id)))
       
       # 重复标签的时候会出现形如：abcd (1).jpg 这种形式的文件名
       new_path = re.sub(" \(\d+\)", "", new_path)
       print(new_path)
       os.rename(old_path, new_path)
   ```



## 2. 配置化


1. model.yaml  - Model Config

   ```yaml
   # - requirement.txt  -  GPU: tensorflow-gpu, CPU: tensorflow
    # - If you use the GPU version, you need to install some additional applications.
    System:
      DeviceUsage: 0.7
    
    # ModelName: Corresponding to the model file in the model directory,
    # - such as YourModelName.pb, fill in YourModelName here.
    # CharSet: Provides a default optional built-in solution:
    # - [ALPHANUMERIC, ALPHANUMERIC_LOWER, ALPHANUMERIC_UPPER,
    # -- NUMERIC, ALPHABET_LOWER, ALPHABET_UPPER, ALPHABET, ALPHANUMERIC_LOWER_MIX_CHINESE_3500]
    # - Or you can use your own customized character set like: ['a', '1', '2'].
    # CharMaxLength: Maximum length of characters， used for label padding.
    # CharExclude: CharExclude should be a list, like: ['a', '1', '2']
    # - which is convenient for users to freely combine character sets.
    # - If you don't want to manually define the character set manually,
    # - you can choose a built-in character set
    # - and set the characters to be excluded by CharExclude parameter.
    Model:
      Sites: [
        'YourModelName'
      ]
      ModelName: YourModelName
      ModelType: 150x50
      CharSet: ALPHANUMERIC_LOWER
      CharExclude: []
      CharReplace: {}
      ImageWidth: 150
      ImageHeight: 50
    
    # Binaryzation: [-1: Off, >0 and < 255: On].
    # Smoothing: [-1: Off, >0: On].
    # Blur: [-1: Off, >0: On].
    # Resize: [WIDTH, HEIGHT]
    # - If the image size is too small, the training effect will be poor and you need to zoom in.
    # ReplaceTransparent: [True, False]
    # - True: Convert transparent images in RGBA format to opaque RGB format,
    # - False: Keep the original image
    Pretreatment:
      Binaryzation: -1
      Smoothing: -1
      Blur: -1
      Resize: [150, 50]
      ReplaceTransparent: True
    
    # CNNNetwork: [CNN5, ResNet, DenseNet]
    # RecurrentNetwork: [BLSTM, LSTM, SRU, BSRU, GRU]
    # - The recommended configuration is CNN5+BLSTM / ResNet+BLSTM
    # HiddenNum: [64, 128, 256]
    # - This parameter indicates the number of nodes used to remember and store past states.
    # Optimizer: Loss function algorithm for calculating gradient.
    # - [AdaBound, Adam, Momentum]
    NeuralNet:
      CNNNetwork: CNN5
      RecurrentNetwork: BLSTM
      HiddenNum: 64
      KeepProb: 0.98
      Optimizer: AdaBound
      PreprocessCollapseRepeated: False
      CTCMergeRepeated: True
      CTCBeamWidth: 1
      CTCTopPaths: 1
    
    # TrainsPath and TestPath: The local absolute path of your training and testing set.
    # DatasetPath: Package a sample of the TFRecords format from this path.
    # TrainRegex and TestRegex: Default matching apple_20181010121212.jpg file.
    # - The Default is .*?(?=_.*\.)
    # TestSetNum: This is an optional parameter that is used when you want to extract some of the test set
    # - from the training set when you are not preparing the test set separately.
    # SavedSteps: A Session.run() execution is called a Step,
    # - Used to save training progress, Default value is 100.
    # ValidationSteps: Used to calculate accuracy, Default value is 500.
    # TestSetNum: The number of test sets, if an automatic allocation strategy is used (TestPath not set).
    # EndAcc: Finish the training when the accuracy reaches [EndAcc*100]% and other conditions.
    # EndCost: Finish the training when the cost reaches EndCost and other conditions.
    # EndEpochs: Finish the training when the epoch is greater than the defined epoch and other conditions.
    # BatchSize: Number of samples selected for one training step.
    # TestBatchSize: Number of samples selected for one validation step.
    # LearningRate: Recommended value[0.01: MomentumOptimizer/AdamOptimizer, 0.001: AdaBoundOptimizer]
    Trains:
      TrainsPath: './dataset/mnist-CNN5BLSTM-H64-28x28_trains.tfrecords'
      TestPath: './dataset/mnist-CNN5BLSTM-H64-28x28_test.tfrecords'
      DatasetPath: [
        "D:/***"
      ]
      TrainRegex: '.*?(?=_)'
      TestSetNum: 300
      SavedSteps: 100
      ValidationSteps: 500
      EndAcc: 0.95
      EndCost: 0.1
      EndEpochs: 2
      BatchSize: 128
      TestBatchSize: 300
      LearningRate: 0.001
      DecayRate: 0.98
      DecaySteps: 10000
   ```

# 运行

1. 命令行或终端运行：```python trains.py```
2. 使用 PyCharm 运行，右键 Run

