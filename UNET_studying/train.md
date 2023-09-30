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
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = UNet(n_channels=3, n_classes=args.classes, bilinear=args.bilinear)
model = model.to(memory_format=torch.channels_last,device=device) # 都是为了加快速度
optimizer = optim.RMSprop(model.parameters(),
                          lr=learning_rate, weight_decay=weight_decay, momentum=momentum, foreach=True)
scheduler = optim.lr_scheduler.ReduceLROnPlateau(optimizer, 'max', patience=5)  # 需要研究！！！！goal: maximize Dice score
grad_scaler = torch.cuda.amp.GradScaler(enabled=amp)
criterion = nn.CrossEntropyLoss() if model.n_classes > 1 else nn.BCEWithLogitsLoss()
global_step = 0
```
***
***注释***
- grad_scaler：用混合精度来提升运算速度  https://zhuanlan.zhihu.com/p/165152789

## (Initialize logging) 需要研究下面的用法
```
# 注释1
experiment = wandb.init(project='U-Net', resume='allow', anonymous='must')
# 注释2
experiment.config.update(
    dict(epochs=epochs, batch_size=batch_size, learning_rate=learning_rate,
         val_percent=val_percent, save_checkpoint=save_checkpoint, img_scale=img_scale, amp=amp)
)

logging.info(f'''Starting training:
    Epochs:          {epochs}
    Batch size:      {batch_size}
    Learning rate:   {learning_rate}
    Training size:   {n_train}
    Validation size: {n_val}
    Checkpoints:     {save_checkpoint}
    Device:          {device.type}
    Images scaling:  {img_scale}
    Mixed Precision: {amp}
''')
```
***
***注释***
1. wandb初始化（以匿名的方式进行，可以不用登录自己的账号），注释信息中提供的网址就可以进去可视化
2. 对该项目的具体参数进行记录，如下
![image](https://github.com/171909771/deep-learning/assets/41554601/e48503ab-712d-4ad4-b631-f9f43dfaa3c3)


## 5. Begin training
```
for epoch in range(1, epochs + 1):
    model.train()  # 注释2
    epoch_loss = 0
    with tqdm(total=n_train, desc=f'Epoch {epoch}/{epochs}', unit='img') as pbar:  # tqdm后缀set_postfix的用法
        for batch in train_loader:
            images, true_masks = batch['image'], batch['mask']

            assert images.shape[1] == model.n_channels, \
                f'Network has been defined with {model.n_channels} input channels, ' \
                f'but loaded images have {images.shape[1]} channels. Please check that ' \
                'the images are loaded correctly.' # 注释3

            images = images.to(device=device, dtype=torch.float32, memory_format=torch.channels_last)
            true_masks = true_masks.to(device=device, dtype=torch.long)

            with torch.autocast(device.type if device.type != 'mps' else 'cpu', enabled=amp): # 注释1
                masks_pred = model(images)
                if model.n_classes == 1:
                    loss = criterion(masks_pred.squeeze(1), true_masks.float())
                    loss += dice_loss(F.sigmoid(masks_pred.squeeze(1)), true_masks.float(), multiclass=False)
                else:
                    loss = criterion(masks_pred, true_masks)
                    loss += dice_loss(   # 注释4
                        F.softmax(masks_pred, dim=1).float(),
                        F.one_hot(true_masks, model.n_classes).permute(0, 3, 1, 2).float(),
                        multiclass=True
                    )

            optimizer.zero_grad(set_to_none=True)
            grad_scaler.scale(loss).backward()  # 注释1
            torch.nn.utils.clip_grad_norm_(model.parameters(), gradient_clipping)
            grad_scaler.step(optimizer)  # 注释1
            grad_scaler.update()  # 注释1

            pbar.update(images.shape[0]) # tqdm 参数更新
            global_step += 1
            epoch_loss += loss.item()
            # 注释5
            experiment.log({
                'train loss': loss.item(),
                'step': global_step,
                'epoch': epoch
            })
            pbar.set_postfix(**{'loss (batch)': loss.item()}) # tqdm 添加后缀参数

            # Evaluation round
            division_step = (n_train // (5 * batch_size))
            if division_step > 0:
                if global_step % division_step == 0:
                    histograms = {}
                    for tag, value in model.named_parameters():
                        tag = tag.replace('/', '.')
                        if not (torch.isinf(value) | torch.isnan(value)).any():
                            histograms['Weights/' + tag] = wandb.Histogram(value.data.cpu())
                        if not (torch.isinf(value.grad) | torch.isnan(value.grad)).any():
                            histograms['Gradients/' + tag] = wandb.Histogram(value.grad.data.cpu())

                    val_score = evaluate(model, val_loader, device, amp)  # 评估参考 
                    scheduler.step(val_score)

                    logging.info('Validation Dice score: {}'.format(val_score))
                    try:
                        experiment.log({
                            'learning rate': optimizer.param_groups[0]['lr'],
                            'validation Dice': val_score,
                            'images': wandb.Image(images[0].cpu()),
                            'masks': {
                                'true': wandb.Image(true_masks[0].float().cpu()),
                                'pred': wandb.Image(masks_pred.argmax(dim=1)[0].float().cpu()),
                            },
                            'step': global_step,
                            'epoch': epoch,
                            **histograms
                        })
                    except:
                        pass

    if save_checkpoint:
        Path(dir_checkpoint).mkdir(parents=True, exist_ok=True)
        state_dict = model.state_dict()  # 提取model中的参数具体值
        state_dict['mask_values'] = dataset.mask_values
        torch.save(state_dict, str(dir_checkpoint / 'checkpoint_epoch{}.pth'.format(epoch)))  # 参数保存
        logging.info(f'Checkpoint {epoch} saved!')
```
***
***注释***
1. grad_scaler and autocast: 用混合精度来提升运算速度 https://zhuanlan.zhihu.com/p/165152789
2. model.train() vs model.eval(): 是否启用 Batch Normalization 和 Dropout. https://blog.csdn.net/weixin_44211968/article/details/123774649
3. assert： 条件性警告，如果条件为假，则报错后面内容
4. dice_loss: 比较两个tensor的相似性 https://blog.csdn.net/qq_54000767/article/details/129905320
5. 接前面的wandb，这里记录的参数就可以在网页中可视化












