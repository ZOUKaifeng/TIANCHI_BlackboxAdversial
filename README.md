# [TIANCHI_BlackboxAdversial](https://tianchi.aliyun.com/forum/postDetail?spm=5176.12282027.0.0.6003379cgB3pPW&postId=79471)




## Environment & Run
- python3
- pytorch>=1.0
- pillow>=5.0
- dlib ver.19.17   only support python3.5    with [shape_predictor_68_face_landmarks.dat_百度云盘提取码：4qjg](https://pan.baidu.com/s/1LMhhW2tXa8a1m2dx8-mCzQ&shfl=shareset) or [shape_predictor_68_face_landmarks.dat_Google drive](https://drive.google.com/open?id=1iMXiyvu3nYcNumtUHifVauU3-P_I_ssV)
- scikit-image>=0.14
- [models_百度云盘提取码：u46u](https://pan.baidu.com/s/1USe0e12jyeVj49AELL7KLw&shfl=shareset) or [models_Google drive](https://drive.google.com/open?id=1KrBN9-vlpmcbX5N-vc0QtKVsXuxF0jXd)

Download and unzip models
```bash
$ python target_iteration.py
```
If you only add noise to the face area, you need to leverage dlib to crop the face, which will be elaborated later.

## Methods
### Ensemble models
To address the black-box face attack challenge, we integrate the common DNN model structure<sup>[1]</sup>, including IR50, IR101, IR152 (model depth is different). The code for model construction is in [model_irse.py](https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/blob/master/model_irse.py). The specific algorithm flow chart is shown in Figure 1. Considering that the online evaluation system may determine the category of the image by similarity, we employ the target attack. [Cal_likehood.py]() calculates the similarity between the faces through multi-model ensembling. We select the second similar image as the attack target. At the same time, our loss function is made up of three components, the classic distance loss such as L2, cos loss. TV loss is to maintain the smoothness of the image, which will be elaborated later. The resulting noise will be convolved by gauss kernel and finally superimposed on the original image. The above process is iterated until the current picture is terminated with its own matrix similarity of more than 0.25.

![image](https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/blob/master/assets/Algorithm%20flowchart%20n.png)

In addition, our model still adopts multi-process multi-graphics acceleration. We utilize two GTX 1080Ti, and it takes less than one hour to generate 712 samples.

### TV loss
In the process of noise cancelling, the artificial noises may have a very enormous visual impact on the result images. At this time, we need to add some regularizaiton to the optimization problem to restrain the image smooth. TV loss is A commonly used regularizaiton in the computer vision. The integration of the continuous domain becomes the summation in the discrete region of the pixel. The specific calculation process is as follows:
$$ tvloss= ∑_{i,j}((x_{i,j-1}-x_{i,j} )^2+(x_{i+1,j}-x_{i,j} )^2 )^{β/2} $$

### Gaussian filtering
Gaussian filtering combines image frequency domain processing with time domain processing under the image processing concept. As a low-pass filter, it can filter low-frequency energy (such as noise) to smooth the image.

Gaussian filtering is performed on the generated interference noise, so that the generated noise of each pixel has correlation with surrounding pixels, which reduces the difference between the interference noise generated by different models (because different models have similar classification boundaries), effectively improving fight against the success rate of sample attacks. At the same time, considering that the online test may have a defense mechanism such as Gaussian filtering, adding Gaussian filtering when the algorithm generates noise can also invalidate the defense mechanism to improve the sample attack rate. This can be done by convolution using a Gaussian kernel function. The Gaussian kernel is as follows:
$$G(x,y)=1/{(2πσ^2)} e^{{-(x^2+y^2)}/2σ^2} $$

### Noise regional attention<sup>[2]</sup>
The existing neural network model largely rely on critical regions(eyes, noses) to distingush from human faces. In the [Face Attention Maps Visualization.ipynb](https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/blob/master/Face%20Attention%20Maps%20Visualization.ipynb) code, we try to generate an attention map on the image, thus find colored face region is more prominent in face classification task.


 ![image](https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/blob/master/assets/attention%20map%20init.png)  ![image](https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/blob/master/assets/attention%20map%20final.png) 
 
 
 Therefore, we restrict the adversarial noises on significant facial areas. In the implementation, we use dlib to calibrate the 68 landmarks of the face, select 17 points to form a non-mask area, and finally we will save the generated image as attentional masks [mask1](https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/tree/master/mask1). For a few pictures that cannot be used to calibrate the mapmark with dlib , we manually frame the face range.
 
 
 <div align=center><img width="250" height="250" src="https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/blob/master/assets/dlib%2068%20face%20landmarks%20crop.png"/></div>
 
 The order of selecting 17 face landmarks is (48, 59-54, 26-17), reference code [crop_image.py](https://github.com/BruceQFWang/TIANCHI_BlackboxAdversial/blob/master/crop_image.py) In the experiment, it took about 10 minutes to generate 712 non-mask areas using dlib.
 

### momentum trick<sup>[1]</sup>
Integrating the momentum into the iterative process of the attack stabilizes the update direction and leaves the poor local maximum during the iteration, resulting in adversarial samples with strong generalization ability. In order to elevated the success rate of black box attacks, we integrate the momentum iteration algorithm into our pipeline. Experiments show that the black box attack is better after adding the momentum term. The formula for the calculation is as follows:
$$
g_{n+1}= μ*g_n+(∇_x L(X_n^{adv},y^{true};θ))/{||∇_x L(X_n^{adv},y^{true};θ)||_1 }
$$


$$X_{n+1}^{adv}=Clip_X^ϵ (X_n^{adv}+α*sign(g_{n+1}) )  $$

### input diversity<sup>[3]</sup>
When training the lfw dataset, in addition to directly cropping the face portion of 112*112, we also employ a random padding similar to data augmentation, random resizing operation, to promote the diversity of the input mode.
The algorithm computation process is as follows:

$$X_{n+1}^{adv}=Clip_X^ϵ ( X_n^{adv}+α*sign(∇_x L(T(X_n^{adv};p),y^{true};θ)) )$$


## Reference
1. [Boosting Adversarial Attacks with Momentum](https://arxiv.org/pdf/1710.06081.pdf)
2. [Paying More Attention to Attention: Improving the Performance of Convolutional Neural Networks via Attention Transfer](https://arxiv.org/abs/1612.03928) | [code](https://github.com/szagoruyko/attention-transfer)
3. [Improving Transferability of Adversarial Examples with Input Diversity](https://arxiv.org/abs/1803.06978)
