- https://github.com/milesial/Pytorch-UNet/blob/master/train.py
#### 导入库
```
import argparse
import logging
import os
import random
import sys
import torch
import torch.nn as nn
import torch.nn.functional as F
import torchvision.transforms as transforms
import torchvision.transforms.functional as TF
from pathlib import Path
from torch import optim
from torch.utils.data import DataLoader, random_split
from tqdm import tqdm

import wandb
from evaluate import evaluate
from unet import UNet
from utils.data_loading import BasicDataset, CarvanaDataset
from utils.dice_score import dice_loss

dir_img = Path('./data/imgs/')
dir_mask = Path('./data/masks/')
dir_checkpoint = Path('./checkpoints/')
```

## 1. Create dataset
- https://github.com/171909771/deep-learning/blob/main/UNET_studying/%E5%9B%BE%E5%83%8F%E5%89%8D%E5%A4%84%E7%90%86.md
```
try:
    dataset = CarvanaDataset(dir_img, dir_mask, img_scale)
except (AssertionError, RuntimeError, IndexError):
    dataset = BasicDataset(dir_img, dir_mask, img_scale)
```

## 2. Split into train / validation partitions
```
n_val = int(len(dataset) * val_percent)
n_train = len(dataset) - n_val
train_set, val_set = random_split(dataset, [n_train, n_val], generator=torch.Generator().manual_seed(0))
```

## 3. Create data loaders
```
loader_args = dict(batch_size=batch_size, num_workers=os.cpu_count(), pin_memory=True)
train_loader = DataLoader(train_set, shuffle=True, **loader_args)
val_loader = DataLoader(val_set, shuffle=False, drop_last=True, **loader_args)
```

## 4. Set up the optimizer, the loss, the learning rate scheduler and the loss scaling for AMP
```
optimizer = optim.RMSprop(model.parameters(),
                          lr=learning_rate, weight_decay=weight_decay, momentum=momentum, foreach=True)
scheduler = optim.lr_scheduler.ReduceLROnPlateau(optimizer, 'max', patience=5)  # 需要研究！！！！goal: maximize Dice score
grad_scaler = torch.cuda.amp.GradScaler(enabled=amp)
criterion = nn.CrossEntropyLoss() if model.n_classes > 1 else nn.BCEWithLogitsLoss()
global_step = 0
```
***
***注释***
grad_scaler：用混合精度来提升运算速度
- https://zhuanlan.zhihu.com/p/165152789
































## dice_loss
- https://blog.csdn.net/qq_54000767/article/details/129905320

##  grad_scaler 
- https://zhuanlan.zhihu.com/p/165152789
- 用混合精度来提升运算速度
