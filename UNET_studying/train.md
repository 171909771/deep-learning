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

```
try:
    dataset = CarvanaDataset(dir_img, dir_mask, img_scale)
except (AssertionError, RuntimeError, IndexError):
    dataset = BasicDataset(dir_img, dir_mask, img_scale)
```

# 2. Split into train / validation partitions
n_val = int(len(dataset) * val_percent)
n_train = len(dataset) - n_val
train_set, val_set = random_split(dataset, [n_train, n_val], generator=torch.Generator().manual_seed(0))

# 3. Create data loaders
loader_args = dict(batch_size=batch_size, num_workers=os.cpu_count(), pin_memory=True)
train_loader = DataLoader(train_set, shuffle=True, **loader_args)
val_loader = DataLoader(val_set, shuffle=False, drop_last=True, **loader_args)



























## dice_loss
- https://blog.csdn.net/qq_54000767/article/details/129905320

##  grad_scaler 
- https://zhuanlan.zhihu.com/p/165152789
- 用混合精度来提升运算速度
