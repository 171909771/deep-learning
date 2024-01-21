Json文件就是下面代码的示例文件

### onekey notice
```
# 当出现
WARNING	findfont: Generic family 'sans-serif' not found because none of the following families were found: Arial Unicode MS
# 在适当的地方加入该行代码即可
plt.rcParams['font.sans-serif'] = ['DejaVu Sans', 'Arial', 'Verdana']
```

