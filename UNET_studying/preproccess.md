- https://github.com/milesial/Pytorch-UNet/blob/master/utils/data_loading.py
#### 载入库
```
import logging
import numpy as np
import torch
from PIL import Image
from functools import lru_cache
from functools import partial
from itertools import repeat
from multiprocessing import Pool
from os import listdir
from os.path import splitext, isfile, join
from pathlib import Path
from torch.utils.data import Dataset
from tqdm import tqdm
```

## 主程序
### 1.读取文件目录下的文件名（不要后缀名）
```
images_dir = Path('./data/imgs/train/')
mask_dir = Path('./data/masks/train_masks/')
images_dir = Path(images_dir)
mask_dir = Path(mask_dir)
ids = [splitext(file)[0] for file in listdir(images_dir) if isfile(join(images_dir, file)) and not file.startswith('.')]
```
- splitext：以"."将str划分
- listdir：读取读取文件目录下的文件名
- file.startswith('.')：不要以"."开头的文件，文件夹里面有一个".keep"文件，这个不要

### 2.遍历所有mask文件看是否都是0,1值
#### 2.1. 读取mask图像，并且转换为数值，最后将数值唯一化(e.g., 图片是由大量的0，1组成，最后得到0，1)
```
def unique_mask_values(idx, mask_dir, mask_suffix):
    mask_file = list(mask_dir.glob(idx + mask_suffix + '.*'))[0]   # 提取mask文件名
    mask = np.asarray(load_image(mask_file))   # 读取图片，并且转换为array
    if mask.ndim == 2:    #  mask文件是二维数据
        return np.unique(mask)     #  将mask的数据唯一化
    elif mask.ndim == 3:
        mask = mask.reshape(-1, mask.shape[-1])
        return np.unique(mask, axis=0)
    else:
        raise ValueError(f'Loaded masks should have 2 or 3 dimensions, found {mask.ndim}')
```
***
***注释***
- 分割图层文件名为 "00087a6bd4dc_01_mask.gif"
- .glob: 查找附后后面条件的文件
- idx：代表去除后缀名和原始图像相同的文件名
- 由于mask_dir.glob为内存地址，需要用list来提取内容，再用[0]来提取list中的内容
#### 2.2. 用unique_mask_values遍历所有mask文件
```
with Pool() as p:
            unique = list(tqdm(
                p.imap(partial(unique_mask_values, mask_dir=mask_dir, mask_suffix=mask_suffix), ids),
                total=len(ids)
            ))
```
***
***注释***
- 注意tqdm的用法
- P.imap：并行处理迭代like
```
unique = []
for n in tqdm(ids,total=len(ids),desc="Processing"):
    result = partial(unique_mask_values, mask_dir=mask_dir, mask_suffix=mask_suffix)(n),
    unique.append(result)
```
- P.imap vs P.map: 前者是每次迭代都给出结果，后者全部计算完成后给出最终结果（浪费内存，不要用）
- partial：给unique_mask_values的mask_dir和mask_suffix赋予了默认值。

#### 2.3. 对遍历文件后得到的list取唯一值
```
mask_values = list(sorted(np.unique(np.concatenate(unique), axis=0).tolist()))
```


