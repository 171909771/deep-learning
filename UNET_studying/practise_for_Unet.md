## 预处理
```
# clone github
!git clone https://github.com/milesial/Pytorch-UNet.git
%cd /kaggle/working/Pytorch-UNet

# copy input file to output file
import os
import shutil
source_dir = "/kaggle/input/testforunet/test"
target_dir = "/kaggle/working/Pytorch-UNet/data"
## Create target subdirectories if they don't exist
os.makedirs(os.path.join(target_dir, "imgs"), exist_ok=True)
os.makedirs(os.path.join(target_dir, "masks"), exist_ok=True)
## Copy files
for filename in os.listdir(source_dir):
    file_path = os.path.join(source_dir, filename)
    if filename.endswith('.jpg'):
        shutil.copy(file_path, os.path.join(target_dir, "imgs"))
    elif filename.endswith('.png'):
        shutil.copy(file_path, os.path.join(target_dir, "masks"))
```

## 训练
```
!python train.py --amp --classes 3 --epochs 10 -l 1e-4
```
## 预测
```
!python predict.py --classes 3 -i /kaggle/input/testforunet/test/1.jpg -o output.png --model /kaggle/working/Pytorch-UNet/checkpoints/checkpoint_epoch5.pth
```

## 显示图片
```
import matplotlib.pyplot as plt
import matplotlib.image as mpimg

# Replace with your image file path
image_path = '/kaggle/working/Pytorch-UNet/output.png'  # or .png

# Read and display the image
img = mpimg.imread(image_path)
plt.imshow(img)
plt.axis('off')  # Turn off axis numbers
plt.show()
```


