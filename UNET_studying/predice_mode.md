-  https://github.com/milesial/Pytorch-UNet/blob/master/predict.py

#### 载入包
```
import torch
from utils.data_loading import BasicDataset
from PIL import Image
from unet import UNet
import torch.nn.functional as F
import numpy as np
```
#### 载入Unet和训练好的参数
```
net = UNet(n_channels=3, n_classes=2, bilinear=False)  # 建立模型
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
net.to(device=device)
state_dict = torch.load('./checkpoint_epoch5.pth', map_location=device)  # 读取保存的model参数值
mask_values = state_dict.pop('mask_values', [0, 1])
net.load_state_dict(state_dict)  # 将参数读入到model中
```
## 开始预测
### 1. 载入图片并用BasicDataset转换成需要的矩阵 torch.Size([1, 3, 1280, 1918])
```
full_img = Image.open('data/imgs/00087a6bd4dc_01.jpg')
net.eval()
img = torch.from_numpy(BasicDataset.preprocess(None, full_img, 1, is_mask=False))    # torch.Size([3, 1280, 1918])
img = img.unsqueeze(0)    # torch.Size([1, 3, 1280, 1918])
img = img.to(device=device, dtype=torch.float32)
```
### 2. 模型预测
- 用net将图片矩阵转换成 torch.Size([1, 2, 1280, 1918])，（注意！！！！从3到2非常重要）
- 用argmax判定前面的2中哪个大，取大的index存为mask，矩阵变为 torch.Size([1, 1280, 1918])
- 最后把torch变为numpy
```
with torch.no_grad():    ## 计算梯度，预测时必须加入，不然内存报错
    output = net(img).cpu()  # torch.Size([1, 2, 1280, 1918])
    output = F.interpolate(output, (full_img.size[1], full_img.size[0]), mode='bilinear')   ## 似乎可以不要，待研究！！！！
    mask = output.argmax(dim=1)  # torch.Size([1, 1280, 1918])
mask=mask[0].long().squeeze().numpy()  # (1280, 1918)
# 似乎可以简化为 mask=mask[0].numpy()  ，待研究！！！！
```
### 3. 制造一个与mask大小一致的False矩阵  (1280, 1918)
```
out = np.zeros((mask.shape[-2], mask.shape[-1]), dtype=bool)
```
### 4. 将False矩阵根据mask添加TURE
```
# 方法一：官方
for i, v in enumerate(mask_values):
    out[mask == i] = v

# 方法二：乱来，效果一致
out[mask == 1]=1
```
### 5. 将矩阵逆转为图片
```
result=Image.fromarray(out)
result
```
