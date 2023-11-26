```
import json
from PIL import Image, ImageDraw
import numpy as np
json_file_path = '../input/testhaha/10044.json'
# Load JSON data
with open(json_file_path, 'r') as json_file:
    data = json.load(json_file)
width = 3000
height = 3000
pixel_values = data
mask = Image.new('1', (width, height), 0)
draw = ImageDraw.Draw(mask)
for pixel1 in pixel_values:
    for pixel2 in pixel1:
        draw.point((pixel2[0], pixel2[1]), fill=1)

     
# Convert the binary mask to a NumPy array
mask_array = np.array(mask).astype(int)

# 图片很有可能看不见，如果图片太大，运行下面的命令导出到本地查看
# Convert the array inversely and save the the array to .png file 
plt.imsave('/kaggle/working/binary_mask.png', mask_array^1, cmap='gray', vmin=0, vmax=1)

# Plot the binary mask
plt.imshow(mask_array, cmap='gray', vmin=0, vmax=1)
plt.title('Binary Mask')
plt.show()
```
