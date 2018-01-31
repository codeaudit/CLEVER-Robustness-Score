CLEVER: A Robustness Metric For Deep Neural Networks
=====================================

CLEVER (**C**ross-**L**ipschitz **E**xtreme **V**alue for n**E**twork **R**obustness) is a metric for
measuring the robustness of deep neural networks.  It estimates the robustness
lower bound by sampling the norm of gradients and fitting a limit distribution
using extreme value theory. CLEVER score is attack-agnostic; a higher score
number indicates that the network is likely to be less venerable to adversarial
examples.  CLEVER can be efficiently computed even for large state-of-the-art
ImageNet models like ResNet-50 and Inception-v3.

For more details, please see our paper:

[Evaluating the Robustness of Neural Networks: An Extreme Value Theory Approach](https://openreview.net/pdf?id=BkUHlMZ0b)
by Tsui-Wei Weng\*, Huan Zhang\*, Pin-Yu Chen, Dong Su, Yupeng Gao, Jinfeng Yi, Cho-Jui Hsieh and Luca Daniel

\* Equal contribution


Setup and train models
-------------------------------------

The code is tested with python3 and TensorFlow v1.3, v1.4 and v1.5. The following
packages are required:

```
sudo apt-get install python3-pip python3-dev
sudo pip3 install --upgrade pip
sudo pip3 install six pillow scipy numpy pandas matplotlib h5py tensorflow-gpu
```

Then clone this repository:

```
git clone https://github.com/huanzhang12/CLEVER.git
cd CLEVER
```

Prepare the MNIST and CIFAR-10 data and models:

```
python3 train_models.py
python3 train_2layer.py
```

To download the ImageNet models:

```
python3 setup_imagenet.py
```

To prepare the ImageNet dataset, download and unzip the following archive:

[ImageNet Test Set](http://jaina.cs.ucdavis.edu/datasets/adv/imagenet/img.tar.gz)


and put the `imgs` folder in `../imagenetdata`, relative to the CLEVER repository. 
This path can be changed in `setup_imagenet.py`.

```
cd ..
mkdir imagenetdata && cd imagenetdata
wget http://jaina.cs.ucdavis.edu/datasets/adv/imagenet/img.tar.gz
tar zxf img.tar.gz
cd ../CLEVER
```

How to run
--------------------------------------

### Step 1: Collect gradients

The first step for computing CLEVER score is to collect gradient samples.
The following command collects gradient samples for 10 images in MNIST dataset;
for each image, 3 target attack classes are chosen (random, top-2 and least likely).
Images that are classified incorrectly will be skipped, so you might get less than
10 images.
The default network used has a 7-layer AlexNet-like CNN structure.

```
python3 collect_gradients.py --dataset mnist --numimg 10
```

Results will be saved into folder `lipschitz_mat/mnist_normal` by default (which can be
changed by specifying the `--saved <folder name>` parameter), as
a few `.mat` files.

Run `python3 collect_gradients.py -h` for additional help information.

### Step 2: Compute the CLEVER score

To compute CLEVER score using the collected gradients, 
run `clever.py` with data saving folder as a parameter:

```
python3 clever.py lipschitz_mat/mnist_normal
```

Run `python3 clever.py -h` for additional help information.


### Step 3: How to intepret the score?

At the end of the output of `clever.py`, you will see three `[STATS][L0]` lines similar to the following:

```
[STATS][L0] info = least, least_clever_L1 = 2.7518, least_clever_L2 = 1.1374, least_clever_Li = 0.080179
[STATS][L0] info = random, random_clever_L1 = 2.9561, random_clever_L2 = 1.1213, random_clever_Li = 0.075569
[STATS][L0] info = top2, top2_clever_L1 = 1.6683, top2_clever_L2 = 0.70122, top2_clever_Li = 0.050181
```

The scores shown are the average scores for all (in the example above, 10)
images, with three different target attack classes: least likely, random and
top-2 (the class with second largest probability).  Three scores are provided:
CLEVER\_L2, CLEVER\_Linf and CLEVER\_L1, representing the robustness for L2, L1
and L infinity norm perturbations. CLEVER score for Lp norm roughly reflects
the minimum Lp norm of adversarial perturbations. A higher CLEVER score
indicates better network robustness, as the minimum adversarial perturbation is
likely to have a larger Lp norm. As CLEVER uses a sampling based method, the 
scores may vary slightly for different runs.

More Examples
---------------------------------

For example, the following command will evaluate the CLEVER scores on 1
ImageNet image, for a 50-layer ResNet model. We set the number of gradient
samples per iterations to 512, and run 100 iterations:

```
python3 collect_gradients.py --dataset imagenet --model_name resnet_v2_50 -N 512 -i 100
python3 clever.py lipschitz_mat/imagenet_resnet_v2_50/
```
<p align="center">
  <img src="http://www.huan-zhang.com/images/upload/clever/139.00029510.jpg" alt="Bustard"/>
</p>

For this image (`139.00029510.jpg`, which is the first image given the default
random seed) in dataset, the original class is 139 (bustard), least likely
class is 20 (chickadee), top-2 class is 83 (ruffed grouse), random class target
is 708 (pay-phone).  (These can be observed in `[DATAGEN][L1]` lines of the
output of `collect_gradients.py`). We get the following CLEVER scores:

```
[STATS][L0] info = least, least_clever_L1 = 4.8764, least_clever_L2 = 0.47412, least_clever_Li = 0.0018967
[STATS][L0] info = random, random_clever_L1 = 4.983, random_clever_L2 = 0.4034, random_clever_Li = 0.0019119
[STATS][L0] info = top2, top2_clever_L1 = 0.07491, top2_clever_L2 = 0.0099927, top2_clever_Li = 5.2723e-05
```

The L2 CLEVER score for the three classes are 0.47412, 0.0099927 and 0.4034,
respectively.  It indicates that it is very easy to attack this image from
class 139 to 83.  We then run the CW attack, which is the strongest L2 attack
to date, on this image with the same three target classes. The distortion of
adversarial images are 0.82929, 0.016434, 0.78028 for the three targets.
Indeed, to misclassify the image to class 83, only a very small distortion
(0.016434) is needed. Also, the CLEVER scores are (usually) less than the L2
distortions observed on adversarial examples, but are not too small to be
useless, reflecting the nature that CLEVER is an estimated robustness lower
bound.

CLEVER also has an untargeted version, which is essentially the smallest CLEVER
score over all possible target classes. The following examples shows how to
compute untargeted CLEVER score for 10 images from MNIST dataset, on the
2-layer MLP model:

```
python3 collect_gradients.py --data mnist --model_name 2-layer --target_type 16 --numimg 10
python3 clever.py --untargeted ./lipschitz_mat/mnist_2-layer/
```

Target type 16 (bit 4 set to 1) indicates that we are collecting gradients for
untargeted CLEVER score (see `python3 collect_gradients.py -h` for more details).
The results will look like the following:

```
[STATS][L0] info = untargeted, untargeted_clever_L1 = 3.4482, untargeted_clever_L2 = 0.69393, untargeted_clever_Li = 0.035387
```

For datasets which have many classes, it is very expensive to evaluate the
untargeted CLEVER scores.  However, usually the robustness of the top-2
targeted class can roughly reflect the untargeted robustness.  For example, in
the bustard image example above, if we run an untargeted CW attack, the class
of the adversarial example found is indeed class 83 (ruffed grouse).

Built-in Models 
-------------------------------- 

In the examples shown above we have used several different models.
The code on this repository has a large number of built-in models for
robustness evaluation.  Model can be selected by changing the `--model_name`
parameter to `collect_gradiets.py`.  For MNIST and CIFAR dataset, the following
models are available: "2-layer" (MLP), "normal" (7-layer CNN), "distilled"
(7-layer CNN with defensive distillation), "brelu" (7-layer CNN with Bounded
ReLU).  For ImageNet, available options are: "resnet_v2_50", "resnet_v2_101",
"resnet_v2_152", "inception_v1", "inception_v2", "inception_v3",
"inception_v4", "inception_resnet_v2", "vgg_16", "vgg_19", "mobilenet_v1_025",
"mobilenet_v1_050", "mobilenet_v1_100".


How to evaluate my own model?
--------------------------------

Models for MNIST, CIFAR and ImageNet datasets are defined in `setup_mnist.py`,
`setup_cifar.py` and `setup_imagenet.py`. For MNIST and CIFAR, you can modify
the model definition in `setup_mnist.py` and `setup_cifar.py` directly.  For
ImageNet, a protobuf (.pb) model with frozen network parameters is expected,
and new ImageNet models can be added into `setup_imagenet.py` by adding a new
`AddModel()` entry, similar to other ImageNet models. Please read the comments 
on `AddModel()` in `setup_imagenet.py` for more details.

