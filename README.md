### yes or no classification and sigmoid, gradients and optimizer
- https://nbviewer.org/github/fastai/fastbook/blob/master/04_mnist_basics.ipynb
###### loss.backward: calculate gradients
![image](https://github.com/171909771/deep-learning/assets/41554601/c9dff38e-eb6a-4391-bf63-845f39c5fcf9)
```
def mnist_loss(predictions, targets):
    return torch.where(targets==1, 1-predictions, predictions).mean()
```


### multiple classification and softmax + negative log likelihood = crossentropy
- https://nbviewer.org/github/fastai/fastbook/blob/master/05_pet_breeds.ipynb
![image](https://github.com/171909771/deep-learning/assets/41554601/bb9a2bd7-8aae-4746-acf0-3029525822e2)

###### 1. softmax
```
def softmax(x): return exp(x) / exp(x).sum(dim=1, keepdim=True)
```

###### 2. negative log
![image](https://github.com/171909771/deep-learning/assets/41554601/c1fe4eb7-3fc3-4dee-8e2e-5a840349a51a)

- loss: log(-(result))

![image](https://github.com/171909771/deep-learning/assets/41554601/86a7b943-9bbd-46f3-87c5-f62befe06bbc)
