- https://github.com/milesial/Pytorch-UNet/blob/master/unet/unet_model.py
- https://github.com/milesial/Pytorch-UNet/blob/master/unet/unet_parts.py
![image](https://github.com/171909771/deep-learning/assets/41554601/46eca9c9-f76d-4704-995f-04b725e5ab2e)

## 1. 主程序
```
from .unet_parts import *

class UNet(nn.Module):
    def __init__(self, n_channels, n_classes, bilinear=False):
        super(UNet, self).__init__()
        self.n_channels = n_channels  # 3：因为使RGS
        self.n_classes = n_classes  # 2: 因为是黑白图预测，双值
        self.bilinear = bilinear
        # 看下面的调用函数
        self.inc = (DoubleConv(n_channels, 64)) 
        self.down1 = (Down(64, 128))
        self.down2 = (Down(128, 256))
        self.down3 = (Down(256, 512))
        factor = 2 if bilinear else 1
        self.down4 = (Down(512, 1024 // factor))
        self.up1 = (Up(1024, 512 // factor, bilinear))
        self.up2 = (Up(512, 256 // factor, bilinear))
        self.up3 = (Up(256, 128 // factor, bilinear))
        self.up4 = (Up(128, 64, bilinear))
        self.outc = (OutConv(64, n_classes))

    def forward(self, x):
        x1 = self.inc(x)
        x2 = self.down1(x1)
        x3 = self.down2(x2)
        x4 = self.down3(x3)
        x5 = self.down4(x4)
        x = self.up1(x5, x4)
        x = self.up2(x, x3)
        x = self.up3(x, x2)
        x = self.up4(x, x1)
        logits = self.outc(x)
        return logits

    # use_checkpointing：在训练的前向传播中不保留中间激活值，从而节省下内存，并在反向传播中重新计算相关值，以此来执行一个高效的内存管理。
    def use_checkpointing(self):
        self.inc = torch.utils.checkpoint(self.inc)
        self.down1 = torch.utils.checkpoint(self.down1)
        self.down2 = torch.utils.checkpoint(self.down2)
        self.down3 = torch.utils.checkpoint(self.down3)
        self.down4 = torch.utils.checkpoint(self.down4)
        self.up1 = torch.utils.checkpoint(self.up1)
        self.up2 = torch.utils.checkpoint(self.up2)
        self.up3 = torch.utils.checkpoint(self.up3)
        self.up4 = torch.utils.checkpoint(self.up4)
        self.outc = torch.utils.checkpoint(self.outc)
```

## 2. 调用函数
```
import torch
import torch.nn as nn
import torch.nn.functional as F


class DoubleConv(nn.Module):
    """(convolution => [BN] => ReLU) * 2"""

    def __init__(self, in_channels, out_channels, mid_channels=None):
        super().__init__()
        if not mid_channels:
            mid_channels = out_channels
        self.double_conv = nn.Sequential(
            # 卷积，in_channels 到 mid_channels，
            # 卷积图片大小公式：image_size = ((raw_h/w - kernel + 2*padding)/stride)+1
            nn.Conv2d(in_channels, mid_channels, kernel_size=3, padding=1, bias=False),
            # 归一化
            nn.BatchNorm2d(mid_channels),
            # 激活函数，使函数非线性
            nn.ReLU(inplace=True),
            # 卷积，使channel变成out_channel
            nn.Conv2d(mid_channels, out_channels, kernel_size=3, padding=1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True)
        )

    def forward(self, x):
        return self.double_conv(x)


class Down(nn.Module):
    """Downscaling with maxpool then double conv"""

    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.maxpool_conv = nn.Sequential(
            # 最大池化，使image_size变为原来一半，如果使奇数则-1
            nn.MaxPool2d(2),
            # 卷积，加倍channel
            DoubleConv(in_channels, out_channels)
        )

    def forward(self, x):
        return self.maxpool_conv(x)


class Up(nn.Module):
    """Upscaling then double conv"""

    def __init__(self, in_channels, out_channels, bilinear=True):
        super().__init__()

        # if bilinear, use the normal convolutions to reduce the number of channels
        if bilinear: # 待研究
            self.up = nn.Upsample(scale_factor=2, mode='bilinear', align_corners=True)
            self.conv = DoubleConv(in_channels, out_channels, in_channels // 2)
        else: # 用这个
            # 去卷积， 减半channel，但是，下面的forward又用下采样的图层补齐了channel)
            # 此外，通过调整 kernel_size=2和stride=2 加倍image_size
            # 去卷积图片大小公式：image_size = stride *(raw_h/w - 1) + kernel - 2*padding, 卷积公式逆推
            self.up = nn.ConvTranspose2d(in_channels, in_channels // 2, kernel_size=2, stride=2)
            # 真正减半channel
            self.conv = DoubleConv(in_channels, out_channels)

    def forward(self, x1, x2):
        # 1.去卷积，形成上采样的data
        x1 = self.up(x1)
        # 2.将该层上采样的image_size与该层对应的下采样的image_size对比
        diffY = x2.size()[2] - x1.size()[2]
        diffX = x2.size()[3] - x1.size()[3]
        # 3.通过F.pad函数用0填补上采样的image_size差缺的位置，使之与该层对应的下采样的image_size相同
        x1 = F.pad(x1, [diffX // 2, diffX - diffX // 2,
                        diffY // 2, diffY - diffY // 2])
        # 4.合并同层的上采样及下采样的data，同时也补齐了channel
        x = torch.cat([x2, x1], dim=1)
        # 5.用卷积减半channel
        return self.conv(x)


class OutConv(nn.Module):
    def __init__(self, in_channels, out_channels):
        super(OutConv, self).__init__()
        # 用卷积将channel减到目标数
        self.conv = nn.Conv2d(in_channels, out_channels, kernel_size=1)

    def forward(self, x):
        return self.conv(x)
```
