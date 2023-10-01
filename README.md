## 1. yes or no classification and sigmoid, gradients and optimizer
- https://nbviewer.org/github/fastai/fastbook/blob/master/04_mnist_basics.ipynb
#### loss.backward: calculate gradients
![image](https://github.com/171909771/deep-learning/assets/41554601/c9dff38e-eb6a-4391-bf63-845f39c5fcf9)
```
def mnist_loss(predictions, targets):
    return torch.where(targets==1, 1-predictions, predictions).mean()
```
-每一个项目的预测output只有一个value（x），传入sigmoid函数将值缩放到0-1（yes的概率），target为yes（loss=1-yes的概率），target为no（loss=yes的概率）

## 2. multiple classification and softmax + negative log likelihood = crossentropy
- https://nbviewer.org/github/fastai/fastbook/blob/master/05_pet_breeds.ipynb
![image](https://github.com/171909771/deep-learning/assets/41554601/bb9a2bd7-8aae-4746-acf0-3029525822e2)

#### 2.1. softmax
```
def softmax(x): return exp(x) / exp(x).sum(dim=1, keepdim=True)
```
- 每一个项目的预测output有n个value（n=分类数），将这些value传入softmax得到该项目每个分类的概率（总和为1）

#### 2.2. negative log
![image](https://github.com/171909771/deep-learning/assets/41554601/c1fe4eb7-3fc3-4dee-8e2e-5a840349a51a)

![image](https://github.com/171909771/deep-learning/assets/41554601/86a7b943-9bbd-46f3-87c5-f62befe06bbc)

- 用-log（result）来进一步扩大target差预测的值


## 2. 卷积kernel and padding:
1. kernel的大小（e.g., kernal_size = 3 is 3*3）会使图片的大小  *x * y* 变为 *x-(3-1) * y-(3-1)*
2. padding的大小（e.g., padding = 1) 会使图片的大小  *x * y* 变为 *x+(1+1) * y+(1+1)*
3. 如果要维持图片大小不变，**kernel -1 = padding * 2**
