---
layout: post
title:  "Deep Learning with MXNetR"
date:   2015-11-03 13:17:00 -0800
author: Tong He
categories: rstats
comments: true
---

Deep learning has been an active field of research for some years, there are breakthroughs in image and language understanding etc.
However, there has not yet been a good deep learning package in R that offers state-of-art deep learning models 
and the real GPU support to do fast training on these models.

In this post, we introduce [MXNetR](https://github.com/dmlc/mxnet/tree/master/R-package), an R package that brings fast GPU computation and state-of-art deep learning
to the R community. MXNet allows you to flexibly configure state-of-art deep learning models backed by the fast CPU and GPU back-end.
This post will cover the following topics:

- Train your first neural network in five minutes
- Use MXNet for Handwritten Digits Classification Competition
- Classify ***real world*** images using state-of-art deep learning models.


## Train your first neural network in five minutes

### A Classfication Task

Let's use a demo data to demonstrate the basic grammar and parameters of `mxnet`. Firstly we load in the data:

```r
require(mlbench)
```

```
## Loading required package: mlbench
```

```r
require(mxnet)
```

```
## Loading required package: mxnet
## Loading required package: methods
```

```r
data(Sonar, package="mlbench")

Sonar[,61] = as.numeric(Sonar[,61])-1
train.ind = c(1:50, 100:150)
train.x = data.matrix(Sonar[train.ind, 1:60])
train.y = Sonar[train.ind, 61]
test.x = data.matrix(Sonar[-train.ind, 1:60])
test.y = Sonar[-train.ind, 61]
```

Next we are going to use a multi-layer perceptron as our classifier. In `mxnet`, we offer a function called `mx.mlp` so that users can build a general multi-layer neural network to do classification or regression.

There are several parameters we have to feed to `mx.mlp`:

- Training data and label.
- Number of hidden nodes in each hidden layers.
- Number of nodes in the output layer.
- Type of the activation.
- Type of the output loss.
- The device to train (GPU or CPU).
- Other parameters for `mx.model.FeedForward.create`.

The following code piece is showing a possible way of using `mx.mlp` (the output is modified to be shorter):


```r
mx.set.seed(0)
model <- mx.mlp(train.x, train.y, hidden_node=10, out_node=2, out_activation="softmax",
                num.round=20, array.batch.size=15, learning.rate=0.07, momentum=0.9, 
                eval.metric=mx.metric.accuracy)
```

```
## Auto detect layout of input matrix, use rowmajor..
## Start training with 1 devices
## [1] Train-accuracy=0.488888888888889
## [2] Train-accuracy=0.514285714285714
## [3] Train-accuracy=0.514285714285714

...

## [18] Train-accuracy=0.838095238095238
## [19] Train-accuracy=0.838095238095238
## [20] Train-accuracy=0.838095238095238
```

Note that `mx.set.seed` is the correct function to control the random process in `mxnet`. 
The reason is that most of MXNet random number generator can run on different devices, such as GPU.
We need to use massively parallel PRNG on GPU to get fast random number generations. It can also be quite costly to seed these PRNGs.
So we introduced `mx.set.seed` to control random numbers in MXNet.
You can see the accuracy from the training process. It is also easy to make prediction and evaluate it.


```r
preds = predict(model, test.x)
```

```
## Auto detect layout of input matrix, use rowmajor..
```

```r
pred.label = max.col(t(preds))-1
table(pred.label, test.y)
```

```
##           test.y
## pred.label  0  1
##          0 24 14
##          1 36 33
```

Note for multi-class prediction, mxnet outputs `nclass` x `nexamples`, and each row corresponds to probability of that class.

### A Regression Task with Structure Configuration

Now let's learn something new. We use the following code to load and process the data:

```r
data(BostonHousing, package="mlbench")

train.ind = seq(1, 506, 3)
train.x = data.matrix(BostonHousing[train.ind, -14])
train.y = BostonHousing[train.ind, 14]
test.x = data.matrix(BostonHousing[-train.ind, -14])
test.y = BostonHousing[-train.ind, 14]
```

Although we can use `mx.mlp` again to do regression by changing the `out_activation`, this time we are going to introduce a flexible way to configure neural networks in `mxnet`. The configuration is done by the "Symbol" system in `mxnet`, which takes care of the links among nodes, the activation, dropout ratio, etc. To configure a multi-layer neural network, we can do it in the following way:

```{r}
# Define the input data
data <- mx.symbol.Variable("data")
# A fully connected hidden layer
# data: input source
# num_hidden: number of neurons in this layer
fc1 <- mx.symbol.FullyConnected(data, num_hidden=1)

# Use linear regression for the output layer
lro <- mx.symbol.LinearRegressionOutput(fc1)
```

What matters for a regression task is mainly the last function. It enables the new network to optimize for squared loss. We can now train on this simple data set. In this configuration, we dropped the hidden layer so the input layer is directly connected to the output layer. For more information about the symbolic operation in `mxnet`, please check [our tutorial](http://mxnet.readthedocs.org/en/latest/R-package/ndarrayAndSymbolTutorial.html) on this topic.

Next we can make prediction with this structure and other parameters with `mx.model.FeedForward.create`:



```r
mx.set.seed(0)
model <- mx.model.FeedForward.create(lro, X=train.x, y=train.y,
                                     ctx=mx.cpu(), num.round=50, array.batch.size=20,
                                     learning.rate=2e-6, momentum=0.9, eval.metric=mx.metric.rmse)
```

```
## Auto detect layout of input matrix, use rowmajor..
## Start training with 1 devices
## [1] Train-rmse=16.063282524034
## [2] Train-rmse=12.2792375712573
## [3] Train-rmse=11.1984634005885

...

## [48] Train-rmse=8.26890902770415
## [49] Train-rmse=8.25728089053853
## [50] Train-rmse=8.24580511500735
```

It is also easy to make prediction.


```r
preds = predict(model, test.x)
```

```
## Auto detect layout of input matrix, use rowmajor..
```

```r
sqrt(mean((preds-test.y)^2))
```

```
## [1] 7.800502
```

Here we also changed the `eval.metric` for regression. Currently we have four pre-defined metrics "accuracy", "rmse", "mae" and "rmsle". One might wonder how to customize the evaluation metric. `mxnet` provides the interface for users to define their own metric of interests:


```r
demo.metric.mae <- mx.metric.custom("mae", function(label, pred) {
  res <- mean(abs(label-pred))
  return(res)
})
```

This is an example for mean absolute error. We can simply plug it in the training function:


```r
mx.set.seed(0)
model <- mx.model.FeedForward.create(lro, X=train.x, y=train.y,
                                     ctx=mx.cpu(), num.round=50, array.batch.size=20,
                                     learning.rate=2e-6, momentum=0.9, eval.metric=demo.metric.mae)
```

```
## Auto detect layout of input matrix, use rowmajor..
## Start training with 1 devices
## [1] Train-mae=13.1889538083225
## [2] Train-mae=9.81431959337658
## [3] Train-mae=9.21576419870059

...

## [48] Train-mae=6.41731406417158
## [49] Train-mae=6.41011292926139
## [50] Train-mae=6.40312503493494
```


Congratulations! Now you have learnt the basic for using `mxnet`. We can go further to tackle some real world problems!

## Handwritten Digits Classification Competition

[MNIST](http://yann.lecun.com/exdb/mnist/) is a handwritten digits image data set created by *Yann LeCun*. Every digit is represented by a 28x28 image. It has become a standard data set to test classifiers on simple image input. Neural network is no doubt a strong model for image classification. There's a [long-term hosted competition](https://www.kaggle.com/c/digit-recognizer) on Kaggle using this data set.

We will present the basic usage of [mxnet](https://github.com/dmlc/mxnet/tree/master/R-package) to compete in this challenge.


### Data Loading

First, let us download the data from [here](https://www.kaggle.com/c/digit-recognizer/data), and put them under the `data/` folder in your working directory.

Then we can read them in R and convert to matrices.

```r
require(mxnet)
```

```
## Loading required package: mxnet
## Loading required package: methods
```

```r
train <- read.csv('data/train.csv', header=TRUE)
test <- read.csv('data/test.csv', header=TRUE)
train <- data.matrix(train)
test <- data.matrix(test)

train.x <- train[,-1]
train.y <- train[,1]
```

Here every image is represented as a single row in train/test. The greyscale of each image falls in the range [0, 255], we can linearly transform it into [0,1] by


```r
train.x <- t(train.x/255)
test <- t(test/255)
```
We also transpose the input matrix to npixel x nexamples, which is the column major format accepted by mxnet (and the convention of R).

In the label part, we see the number of each digit is fairly even:


```r
table(train.y)
```

```
## train.y
##    0    1    2    3    4    5    6    7    8    9
## 4132 4684 4177 4351 4072 3795 4137 4401 4063 4188
```

### Network Configuration

Now we have the data. The next step is to configure the structure of our network.


```r
data <- mx.symbol.Variable("data")
fc1 <- mx.symbol.FullyConnected(data, name="fc1", num_hidden=128)
act1 <- mx.symbol.Activation(fc1, name="relu1", act_type="relu")
fc2 <- mx.symbol.FullyConnected(act1, name="fc2", num_hidden=64)
act2 <- mx.symbol.Activation(fc2, name="relu2", act_type="relu")
fc3 <- mx.symbol.FullyConnected(act2, name="fc3", num_hidden=10)
softmax <- mx.symbol.SoftmaxOutput(fc3, name="sm")
```

1. In `mxnet`, we use its own data type `symbol` to configure the network. `data <- mx.symbol.Variable("data")` use `data` to represent the input data, i.e. the input layer.
2. Then we set the first hidden layer by `fc1 <- mx.symbol.FullyConnected(data, name="fc1", num_hidden=128)`. This layer has `data` as the input, its name and the number of hidden neurons.
3. The activation is set by `act1 <- mx.symbol.Activation(fc1, name="relu1", act_type="relu")`. The activation function takes the output from the first hidden layer `fc1`.
4. The second hidden layer takes the result from `act1` as the input, with its name as "fc2" and the number of hidden neurons as 64.
5. the second activation is almost the same as `act1`, except we have a different input source and name.
6. Here comes the output layer. Since there's only 10 digits, we set the number of neurons to 10.
7. Finally we set the activation to softmax to get a probabilistic prediction.

### Training

We are almost ready for the training process. Before we start the computation, let's decide what device should we use.


```r
devices <- mx.cpu()
```

Here we assign CPU to `mxnet`. After all these preparation, you can run the following command to train the neural network! Note again that `mx.set.seed` is the correct function to control the random process in `mxnet`.


```r
mx.set.seed(0)
model <- mx.model.FeedForward.create(softmax, X=train.x, y=train.y,
                                     ctx=devices, num.round=10, array.batch.size=100,
                                     learning.rate=0.07, momentum=0.9,  eval.metric=mx.metric.accuracy,
                                     initializer=mx.init.uniform(0.07),
                                     epoch.end.callback=mx.callback.log.train.metric(100))
```

```
## Start training with 1 devices
## Batch [100] Train-accuracy=0.6563
## Batch [200] Train-accuracy=0.777999999999999
## Batch [300] Train-accuracy=0.827466666666665
## Batch [400] Train-accuracy=0.855499999999999
## [1] Train-accuracy=0.859832935560859
## Batch [100] Train-accuracy=0.9529
## Batch [200] Train-accuracy=0.953049999999999
## Batch [300] Train-accuracy=0.955866666666666
## Batch [400] Train-accuracy=0.957525000000001
## [2] Train-accuracy=0.958309523809525

...

## Batch [100] Train-accuracy=0.9937
## Batch [200] Train-accuracy=0.99235
## Batch [300] Train-accuracy=0.991966666666668
## Batch [400] Train-accuracy=0.991425000000003
## [9] Train-accuracy=0.991500000000003
## Batch [100] Train-accuracy=0.9942
## Batch [200] Train-accuracy=0.99245
## Batch [300] Train-accuracy=0.992433333333334
## Batch [400] Train-accuracy=0.992275000000002
## [10] Train-accuracy=0.992380952380955
```

### Prediction and Submission

To make prediction, we can simply write


```r
preds <- predict(model, test)
dim(preds)
```

```
## [1]    10 28000
```

It is a matrix with 28000 rows and 10 cols, containing the desired classification probabilities from the output layer. To extract the maximum label for each row, we can use the `max.col` in R:


```r
pred.label <- max.col(t(preds)) - 1
table(pred.label)
```

```
## pred.label
##    0    1    2    3    4    5    6    7    8    9
## 2818 3195 2744 2767 2683 2596 2798 2790 2784 2825
```

With a little extra effort in writting to a csv file, we can have our submission to the competition!


```r
submission <- data.frame(ImageId=1:ncol(test), Label=pred.label)
write.csv(submission, file='submission.csv', row.names=FALSE, quote=FALSE)
```

### LeNet

Next we are going to introduce a new network structure: [LeNet](http://yann.lecun.com/exdb/lenet/). It is proposed by Yann LeCun to recognize handwritten digits. Now we are going to demonstrate how to construct and train an LeNet in `mxnet`.

First we construct the network:


```r
# input
data <- mx.symbol.Variable('data')
# first conv
conv1 <- mx.symbol.Convolution(data=data, kernel=c(5,5), num_filter=20)
tanh1 <- mx.symbol.Activation(data=conv1, act_type="tanh")
pool1 <- mx.symbol.Pooling(data=tanh1, pool_type="max",
                          kernel=c(2,2), stride=c(2,2))
# second conv
conv2 <- mx.symbol.Convolution(data=pool1, kernel=c(5,5), num_filter=50)
tanh2 <- mx.symbol.Activation(data=conv2, act_type="tanh")
pool2 <- mx.symbol.Pooling(data=tanh2, pool_type="max",
                          kernel=c(2,2), stride=c(2,2))
# first fullc
flatten <- mx.symbol.Flatten(data=pool2)
fc1 <- mx.symbol.FullyConnected(data=flatten, num_hidden=500)
tanh3 <- mx.symbol.Activation(data=fc1, act_type="tanh")
# second fullc
fc2 <- mx.symbol.FullyConnected(data=tanh3, num_hidden=10)
# loss
lenet <- mx.symbol.SoftmaxOutput(data=fc2)
```

Then let us reshape the matrices into arrays:


```r
train.array <- train.x
dim(train.array) <- c(28, 28, 1, ncol(train.x))
test.array <- test
dim(test.array) <- c(28, 28, 1, ncol(test))
```

Next we are going to compare the training speed on different devices, so the definition of the devices goes first:


```r
n.gpu <- 1
device.cpu <- mx.cpu()
device.gpu <- lapply(0:(n.gpu-1), function(i) {
  mx.gpu(i)
})
```

As you can see, we can pass a list of devices, to ask mxnet to train on multiple GPUs (you can do similar thing for cpu,
but since internal computation of cpu is already multi-threaded, there is less gain than using GPUs).

We start by training on CPU first. Because it takes a bit time to do so, we will only run it for one iteration.


```r
mx.set.seed(0)
tic <- proc.time()
model <- mx.model.FeedForward.create(lenet, X=train.array, y=train.y,
                                     ctx=device.cpu, num.round=1, array.batch.size=100,
                                     learning.rate=0.05, momentum=0.9, wd=0.00001,
                                     eval.metric=mx.metric.accuracy,
                                     epoch.end.callback=mx.callback.log.train.metric(100))
```

```
## Start training with 1 devices
## Batch [100] Train-accuracy=0.1066
## Batch [200] Train-accuracy=0.16495
## Batch [300] Train-accuracy=0.401766666666667
## Batch [400] Train-accuracy=0.537675
## [1] Train-accuracy=0.557136038186157
```

```r
print(proc.time() - tic)
```

```
##    user  system elapsed
## 130.030 204.976  83.821
```

Training on GPU:


```r
mx.set.seed(0)
tic <- proc.time()
model <- mx.model.FeedForward.create(lenet, X=train.array, y=train.y,
                                     ctx=device.gpu, num.round=5, array.batch.size=100,
                                     learning.rate=0.05, momentum=0.9, wd=0.00001,
                                     eval.metric=mx.metric.accuracy,
                                     epoch.end.callback=mx.callback.log.train.metric(100))
```

```
## Start training with 1 devices
## Batch [100] Train-accuracy=0.1066
## Batch [200] Train-accuracy=0.1596
## Batch [300] Train-accuracy=0.3983
## Batch [400] Train-accuracy=0.533975
## [1] Train-accuracy=0.553532219570405
## Batch [100] Train-accuracy=0.958
## Batch [200] Train-accuracy=0.96155
## Batch [300] Train-accuracy=0.966100000000001
## Batch [400] Train-accuracy=0.968550000000003
## [2] Train-accuracy=0.969071428571432
## Batch [100] Train-accuracy=0.977
## Batch [200] Train-accuracy=0.97715
## Batch [300] Train-accuracy=0.979566666666668
## Batch [400] Train-accuracy=0.980900000000003
## [3] Train-accuracy=0.981309523809527
## Batch [100] Train-accuracy=0.9853
## Batch [200] Train-accuracy=0.985899999999999
## Batch [300] Train-accuracy=0.986966666666668
## Batch [400] Train-accuracy=0.988150000000002
## [4] Train-accuracy=0.988452380952384
## Batch [100] Train-accuracy=0.990199999999999
## Batch [200] Train-accuracy=0.98995
## Batch [300] Train-accuracy=0.990600000000001
## Batch [400] Train-accuracy=0.991325000000002
## [5] Train-accuracy=0.991523809523812
```

```r
print(proc.time() - tic)
```

```
##    user  system elapsed
##   9.288   1.680   6.889
```

As you can see by using GPU, we can get a much faster speedup in training!
Finally we can submit the result to Kaggle again to see the improvement of our ranking!


```r
preds <- predict(model, test.array)
pred.label <- max.col(t(preds)) - 1
submission <- data.frame(ImageId=1:ncol(test), Label=pred.label)
write.csv(submission, file='submission.csv', row.names=FALSE, quote=FALSE)
```

## Classify Real-World Images with Pre-trained Model

After the MNIST examples, are you ready to take one step further? One of the cool thing that a deep learning
algorithm can do is to classify real world images.

In this example we will show how to use a pre-trained Inception-BatchNorm Network to predict the class of
real world image. The network architecture is decribed in [1].

The pre-trained Inception-BatchNorm network can be downloaded from [this link](http://webdocs.cs.ualberta.ca/~bx3/data/Inception.zip)
This model gives the recent state-of-art prediction accuracy on image net dataset.

### Pacakge Loading

To get started, we load the mxnet package by

```r
require(mxnet)
```

In this example, we also need the package `imager` to load and preprocess the images in R.


```r
require(imager)
```

### Load the Pretrained Model

Make sure you unzip the pre-trained model in current folder. And we can use the model
loading function to load the model into R.


```r
model = mx.model.load("Inception/Inception_BN", iteration=39)
```

We also need to load in the mean image, which is used for preprocessing using ```mx.nd.load```.


```r
mean.img = as.array(mx.nd.load("Inception/mean_224.nd")[["mean_img"]])
```

### Load and Preprocess the Image

Now we are ready to classify a real image. In this example, we simply take the parrots image
from `imager`. But you can always change it to other images. Firstly we will test it on a photo of Mt. Baker in north WA.

Load and plot the image:


```r
im <- load.image("Pics/MtBaker.jpg")
plot(im)
```

![plot of chunk unnamed-chunk-5](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/Blog_mxnet_R/Blog_RealWorld_MtBaker.png) 

Before feeding the image to the deep net, we need to do some preprocessing
to make the image fit in the input requirement of deepnet. The preprocessing
includes cropping, and substraction of the mean.
Because mxnet is deeply integerated with R, we can do all the processing in R function.

The preprocessing function:


```r
preproc.image <-function(im, mean.image) {
  # crop the image
  shape <- dim(im)
  short.edge <- min(shape[1:2])
  yy <- floor((shape[1] - short.edge) / 2) + 1
  yend <- yy + short.edge - 1
  xx <- floor((shape[2] - short.edge) / 2) + 1
  xend <- xx + short.edge - 1
  croped <- im[yy:yend, xx:xend,,]
  # resize to 224 x 224, needed by input of the model.
  resized <- resize(croped, 224, 224)
  # convert to array (x, y, channel)
  arr <- as.array(resized)
  dim(arr) = c(224, 224, 3)
  # substract the mean
  normed <- arr - mean.img
  # Reshape to format needed by mxnet (width, height, channel, num)
  dim(normed) <- c(224, 224, 3, 1)
  return(normed)
}
```

We use the defined preprocessing function to get the normalized image.


```r
normed <- preproc.image(im, mean.img)
```

### Classify the Image

Now we are ready to classify the image! We can use the predict function
to get the probability over classes.


```r
prob <- predict(model, X=normed)
dim(prob)
```

```
## [1] 1000    1
```

As you can see ```prob``` is a 1000 times 1 array, which gives the probability
over the 1000 image classes of the input.

We can extract the top-5 class index.

```r
max.idx <- order(prob[,1], decreasing = TRUE)[1:5]
max.idx
```

```
## [1] 981 971 980 673 975
```

These indices do not make too much sense. So let us see what it really represents.
We can read the names of the classes from the following file.


```r
synsets <- readLines("Inception/synset.txt")
```

And let us print the corresponding lines:


```r
print(paste0("Predicted Top-classes: ", synsets[max.idx]))
```

```
## [1] "Predicted Top-classes: n09472597 volcano"      
## [2] "Predicted Top-classes: n09193705 alp"          
## [3] "Predicted Top-classes: n09468604 valley, vale" 
## [4] "Predicted Top-classes: n03792972 mountain tent"
## [5] "Predicted Top-classes: n09288635 geyser"
```

Mt. Baker is indeed a vocalno. We can also see the second most possible guess "alp" is also correct.

Let's see if it still does a good job on some other images. The following photo is taken in Vancouver downtown.


```r
im <- load.image("Pics/Vancouver.jpg")
plot(im)
```

![plot of chunk unnamed-chunk-12](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/Blog_mxnet_R/Blog_RealWorld_Vancouver.png) 

```r
normed <- preproc.image(im, mean.img)
prob <- predict(model, X=normed)
max.idx <- order(prob[,1], decreasing = TRUE)[1:5]
print(paste0("Predicted Top-classes: ", synsets[max.idx]))
```

```
## [1] "Predicted Top-classes: n09332890 lakeside, lakeshore"    
## [2] "Predicted Top-classes: n03983396 pop bottle, soda bottle"
## [3] "Predicted Top-classes: n13133613 ear, spike, capitulum"  
## [4] "Predicted Top-classes: n12144580 corn"                   
## [5] "Predicted Top-classes: n02980441 castle"
```

This photo is indeed taken at lakeside. One interesting guess is the fifth guess "castle". The outline of the building in the city is recognized as the battlements on a castle. We might need more pictures containing "battlements with glass windows" to teach the model about modern city.

How about this photo taken on Titlis:


```r
im <- load.image("Pics/Switzerland.jpg")
plot(im)
```

![plot of chunk unnamed-chunk-13](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/Blog_mxnet_R/Blog_RealWorld_Switzerland.png) 

```r
normed <- preproc.image(im, mean.img)
prob <- predict(model, X=normed)
max.idx <- order(prob[,1], decreasing = TRUE)[1:5]
print(paste0("Predicted Top-classes: ", synsets[max.idx]))
```

```
## [1] "Predicted Top-classes: n04371774 swing"                         
## [2] "Predicted Top-classes: n04275548 spider web, spider's web"      
## [3] "Predicted Top-classes: n01773549 barn spider, Araneus cavaticus"
## [4] "Predicted Top-classes: n03000684 chain saw, chainsaw"           
## [5] "Predicted Top-classes: n03888257 parachute, chute"
```

This time the main element is small and cannot stand out from the "noisy" background. This time the result is not perfect, but we can still find similarity between "swing" and "gondola". 

Now, why don't you take a photo around and ask `mxnet` to tell you what is included? Have some fun!

## Try it out and Contribute
You can find MXNet on [github](https://github.com/dmlc/mxnet/tree/master/R-package). Besides ```R```, MXNet also support ```python``` and ```Julia```,
and allows interpolations of models and analysis results between different language bindings.
MXNet is built by a active community of users. 
Please fork the project on github and contribute your wisdom to make the project even better :)

## Acknowledgement
We would like to thank the [RcppCore Team](https://github.com/RcppCore) for their great helps to make MXNetR happen.

[1] Ioffe, Sergey, and Christian Szegedy. "Batch normalization: Accelerating deep network training by reducing internal covariate shift." arXiv preprint arXiv:1502.03167 (2015).


