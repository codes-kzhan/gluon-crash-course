# Predict with a pre-trained model

A saved model can be used in multiple places, such as continue training, fine-tuning and prediction. This tutorial we will discuss how to predict new examples with the model trained before.

```{.python .input  n=61}
from mxnet import nd
from mxnet import gluon, cpu
from mxnet.gluon import nn
import matplotlib.pyplot as plt
```

We copy the model definition here. (Note, there is an advanced way to save the network definition and load it back without redefining the network, refer [xx] for more details).

```{.python .input  n=6}
net = nn.Sequential()
with net.name_scope():
    net.add(
        nn.Conv2D(channels=6, kernel_size=5, activation='relu'),
        nn.MaxPool2D(pool_size=2, strides=2),
        nn.Conv2D(channels=16, kernel_size=3, activation='relu'),
        nn.MaxPool2D(pool_size=2, strides=2),
        nn.Flatten(),        
        nn.Dense(120, activation="relu"),
        nn.Dense(84, activation="relu"),
        nn.Dense(10)
    )
```

In the last section, we saved all parameters into a file, now let load it back

```{.python .input  n=7}
net.load_params('net.params', ctx=cpu()) 
```

## Predict

sdf 
asU
Remember the data transformation we did for training, now we need the same transformation for predicting, except that we assume the data is a single image instead of a batch of images.

```{.python .input  n=13}
def transform(data):
    # data: (height, weight, channel) ndarray
    return data.transpose((2,0,1)).expand_dims(axis=0).astype('float32')/255
```

Now let's try to predict the first 10 images in the validation dataset and saves the predictions into `preds`.

```{.python .input  n=29}
mnist_valid = gluon.data.vision.FashionMNIST(train=False)
X, y = mnist_valid[:10]

preds = []
for x in X:
    pred = net(transform(x)).argmax(axis=1)
    preds.append(pred.astype('int32').asscalar())
```

Finally, we visualize the images and compare the prediction with the ground truth.

```{.python .input  n=30}
_, figs = plt.subplots(1, 10, figsize=(15, 15))
for f,x in zip(figs, X):
    f.imshow(x.reshape((28,28)).asnumpy())
    f.axes.get_xaxis().set_visible(False)
    f.axes.get_yaxis().set_visible(False)
plt.show()

text_labels = [
    't-shirt', 'trouser', 'pullover', 'dress,', 'coat',
    'sandal', 'shirt', 'sneaker', 'bag', 'ankle boot'
]
print('Pred:', [text_labels[i] for i in preds])
print('True:', [text_labels[i] for i in y])
```

## Predict with models from Gluon model zoo


The LeNet trained on FashionMNIST is a good example to start with, but too simple to predict real-life pictures.  Instead of training large-scale model from scratch, [Gluon model zoo](https://mxnet.incubator.apache.org/api/python/gluon/model_zoo.html) provides multiple pre-trained powerful models. For example, we download and load a pre-trained ResNet-50 V2 model on the ImageNet dataset.

```{.python .input  n=47}
from mxnet.gluon.model_zoo import vision as models
from mxnet.gluon.utils import download
from mxnet import image
 
net = models.resnet50_v2(pretrained=True)
```

We also download and load the text labels for each class.

```{.python .input}
url = 'http://data.mxnet.io/models/imagenet/synset.txt'
fname = download(url)
with open(fname, 'r') as f:    
    text_labels = [' '.join(l.split()[1:]) for l in f]
```

We randomly pick a dog image from Wikipedia as a test image, download and read it.

```{.python .input  n=67}
url = 'http://writm.com/wp-content/uploads/2016/08/Cat-hd-wallpapers.jpg'
url = 'https://upload.wikimedia.org/wikipedia/commons/thumb/b/b5/Golden_Retriever_medium-to-light-coat.jpg/365px-Golden_Retriever_medium-to-light-coat.jpg'
fname = download(url)
x = image.imread(fname)

```

Following the conventional way of preprocessing ImageNet data, we first resize the short edge into 256 pixes and then perform a center crop to obtain a 224-by-224 image. We used the image processing functions provided in the [image module](https://mxnet.incubator.apache.org/api/python/image/image.html).

```{.python .input  n=68}
x = image.resize_short(x, 256)
x, _ = image.center_crop(x, (224,224))
plt.imshow(x.asnumpy())
plt.show()
```

Now you may know it is a golden retriever (You can also infer it from the image URL). 

The futher data transformation is similar to FashionMNIST except that we subtract the RGB means and divide by the corresponding variances to normalize each color channel.

```{.python .input  n=66}
def transform(data):
    data = data.transpose((2,0,1)).expand_dims(axis=0)    
    rgb_mean = nd.array([0.485, 0.456, 0.406]).reshape((1,3,1,1))
    rgb_std = nd.array([0.229, 0.224, 0.225]).reshape((1,3,1,1))
    return (data.astype('float32') / 255 - rgb_mean) / rgb_std
```

Now we can recognize the object in the image now. We perform an additional softmax to the output to obtain probability scores. And then print the top-5 recognized objects.

```{.python .input  n=79}
prob = net(transform(x)).softmax()
idx = prob.topk(k=5)[0]
for i in idx:
    i = int(i.asscalar())
    print('With prob = %.5f, it contains %s' % (
        prob[0,i].asscalar(), text_labels[i]))
```

As can be seen, the model is fairly confident the image contains a goden retriever.
