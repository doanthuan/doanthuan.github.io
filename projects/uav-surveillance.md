---
layout: page
title: 
permalink: /projects/uav-surveillance/
---
<h3></h3>

### Crack Segmentation in Transport Infrastructure

![Image](/assets/crack_seg.jpg)



#### Overview
The project focuses on developing an automated system for crack detection and segmentation in transport infrastructure using advanced deep learning models. The aim is to enhance the inspection process of transport infrastructure such as roads, bridges, and tunnels, where structural integrity is paramount. Timely and accurate crack detection helps prevent costly repairs and accidents, ensuring public safety and longevity of the infrastructure.

#### Background
Traditional crack detection methods in transport infrastructure are often manual, time-consuming, and prone to human error. Current practices rely on visual inspection, which is both labor-intensive and subjective. The advent of deep learning techniques, especially in the field of image segmentation, presents an opportunity to automate this process with high precision and reliability. Recent advancements in Convolutional Neural Networks (CNNs), particularly models designed for pixel-level image segmentation such as U-Net, DeepLab, and Mask R-CNN, have demonstrated remarkable performance in various computer vision tasks, including crack detection.

#### Objectives
1. **Develop a robust crack segmentation model:** Use deep learning techniques to create a model that accurately identifies cracks in images of transport infrastructure.
2. **Improve segmentation accuracy:** Implement advanced architectures such as U-Net, DeepLab, or other state-of-the-art segmentation models, leveraging pre-trained weights on relevant datasets (e.g., ImageNet) and fine-tuning them for crack detection.
3. **Data Augmentation and Preprocessing:** Utilize data augmentation techniques to enhance the generalization capabilities of the model. Develop preprocessing pipelines to handle diverse image qualities, different lighting conditions, and various types of cracks (hairline, medium, large).
4. **Model Evaluation and Metrics:** Evaluate the model using appropriate metrics like Intersection over Union (IoU), F1-score, and precision/recall to ensure high accuracy and low false positives/negatives.
5. **Deployment on Edge Devices:** Explore the possibility of deploying the model on mobile devices or drones equipped with cameras for real-time crack detection in field environments.

#### Methodology
1. **Data Collection & Annotation:** Assemble a diverse dataset of transport infrastructure images, including various crack types and sizes. Manual annotations will be used to create ground truth for training and evaluation.
The dataset contains around 11.200 images that are merged from 12 available crack segmentation datasets. The name prefix of each image is assigned to the corresponding dataset name that the image belong to. There’re also images with no crack pixel, which could be filtered out by the file name pattern “noncrack*” All the images are resized to the size of (448, 448). The two folders images and masks contain all the images. The two folders train and test contain training and testing images splitted from the two above folder. The splitting is stratified so that the proportion of each dataset in the train and test folder are similar
![Image](/assets/crack_sample.png)
2. **Model Selection:** Compare different deep learning models for segmentation (U-Net, DeepLab, Mask R-CNN) and select the best performing one based on validation results.
3. **Training & Optimization:** Train the selected model on the annotated dataset, employing techniques like transfer learning, data augmentation, and hyperparameter tuning to achieve optimal performance.
4. **Post-Processing:** Apply post-processing techniques like morphological operations or connected component analysis to refine the segmented crack regions.
5. **Evaluation:** Test the model on unseen images and assess its generalizability across different environments (urban roads, rural highways, bridges, etc.).
![Image](/assets/crack_result.png)
6. **Deployment:** Develop a lightweight version of the model to be deployed on edge devices or drones for real-time crack detection in the field.

#### Expected Outcomes
- **High-accuracy crack segmentation model** capable of detecting and segmenting cracks with minimal false positives.
- **Automated inspection tool** that significantly reduces the time and labor costs associated with manual inspection.
- **Edge-based solution** for real-time crack detection using drones or mobile devices.
- **Improved safety and maintenance efficiency** in transport infrastructure through timely crack detection and repair recommendations.

#### Potential Impact
The implementation of deep learning-based crack segmentation in transport infrastructure will revolutionize maintenance practices by reducing human dependency, increasing accuracy, and enabling real-time, scalable inspections. This project will contribute to safer and more resilient infrastructure by facilitating proactive maintenance and preventing catastrophic failures caused by unnoticed or untreated cracks.

#### Tools & Technologies
- **Deep Learning Frameworks:** TensorFlow
- **Segmentation Models:** U-Net, DeepLab, Mask R-CNN
- **Data Augmentation:** Keras Augmentation, Albumentations
- **Deployment:** TensorFlow Lite, NVIDIA Jetson
- **Evaluation Metrics:** IoU, F1-Score, Precision/Recall

<br/>
<br/>


<h3></h3>

### Wild fire classification and segmentation using Unmanned Aerial Vehicle (UAV)

#### Title
![Image](/assets/flame_compressed.gif)


#### Dataset
* The dataset is uploaded on IEEE dataport. You can find the dataset here at [IEEE Dataport](https://ieee-dataport.org/open-access/flame-dataset-aerial-imagery-pile-burn-detection-using-drones-uavs) or [DOI](https://dx.doi.org/10.21227/qad6-r683). IEEE account is free, so you can create an account and access the dataset files without any payment or subscription. 

* This table below shows all available data for the dataset.
* This project uses items 7, 8, 9, and 10 from the dataset. Items 7 and 8 are being used for the "Fire_vs_NoFire" image classification. Items 9 and 10 are for the fire segmentation. 
* If you clone this repository on your local drive, please download item [7](https://ieee-dataport.org/open-access/aerial-images-pile-fire-detection-using-drones-uavs) from the dataset and unzip in directory /frames/Training/... for the Training phase of the "Fire_vs_NoFire" image classification. The direcotry looks like this:
```bash
Repository/frames/Training
                    ├── Fire/*.jpg
                    ├── No_Fire/*.jpg
```
* For testing your trained model, please use item [8](https://ieee-dataport.org/open-access/aerial-images-pile-fire-detection-using-drones-uavs) and unzip it in direcotry /frame/Test/... . The direcotry looks like this:
```bash
Repository/frames/Test
                    ├── Fire/*.jpg
                    ├── No_Fire/*.jpg
```
* Items [9](https://ieee-dataport.org/open-access/aerial-images-pile-fire-detection-using-drones-uavs) and [10](https://ieee-dataport.org/open-access/aerial-images-pile-fire-detection-using-drones-uavs) should be unzipped in these directories frames/Segmentation/Data/Image/... and frames/Segmentation/Data/Masks/... accordingly. The direcotry looks like this:
```bash
Repository/frames/Segmentation/Data
                                ├── Images/*.jpg
                                ├── Masks/*.png
```

* Please remove other README files from those directories and make sure that only images are there. 


![Image](/assets/wildfire_table.png)


#### Model
* The binary fire classifcation model of this project is based on the Xception Network:

![Image](/assets/small_Xception_model.PNG)
<br/>
<br/>

* The fire segmentation model of this project is based on the U-NET:

![Image](/assets/u-net-segmentation.PNG)

#### Sample
* A short sample video of the dataset is available on YouTube:
[![Alt text](frames/sample_video.PNG)](https://youtu.be/bHK6g37_KyA "Sample video")

#### Requirements
* os
* re
* cv2
* copy
* tqdm
* scipy
* pickle
* numpy
* random
* itertools
* Keras 2.4.0
* scikit-image
* Tensorflow 2.3.0
* matplotlib.pyplot

#### Code
This code is run and tested on Python 3.6 on linux (Ubuntu 18.04) machine with no issues. There is a config.py file in this directoy which shows all the configuration parameters such as **Mode**, **image target size**, **Epochs**, **batch size**, **train_validation ratio**, etc. All dependency files are available in the root directory of this repository.
* To run the training phase for the "Fire_vs_NoFire" image classification, change the **mode** value to 'Training' in the config.py file. 
```
Mode = 'Training'
```
Make sure that you have copied and unzipped the data in correct direcotry.

* To run the test phase for the "Fire_vs_NoFire" image classification, change the **mode** value to 'Classification' in the config.py file. 
```
Mode = 'Classification'
```
Make sure that you have copied and unzipped the data in correct direcotry.

* To run the test phase for the Fire segmentation, change the **mode** value to 'Classification' in the config.py file. 
```
Mode = 'Segmentation'
```
Make sure that you have copied and unzipped the data in correct direcotry.

Then after setting your parameters, just run the main.py file.
```
python main.py
```

#### Results
* Fire classification accuracy:

![Image](/assets/classification.PNG)

* Fire classification Confusion Matrix:

<img src="/assets/confusion.PNG" width="500" height="500"/>

* Fire segmentation metrics and evaluation:

![Image](/assets/segmentation.PNG)

* Comparison between generated masks and grount truth mask:

![Image](/assets/segmentation_sample.PNG)

* Federated Learning sample <br/>
To consider future challenges, we defined a new sample of federated learning on a local node (NVidia Jetson Nano, 4GB RAM). Jetson Nano is available in two versions: 1) 4GB RAM developer kit, and 2) 2GB RAM developer kit. In this Implementation, the 4GB version is used with the technical specifications of a 128-core Maxwell GPU, a Quad-core ARM A57 @ 1.43 GHz CPU, 4GB LPDDR4 RAM, and a 32GB microSD storage. To test Jetson Nano for the federated learning, items (9) and (10) from Dataset are used for the fire segmentation. Since Jetson Nano has limited RAM, we assumed that each drone has access to a portion of the FLAME dataset. Only 500 fire images and masks are considered for the training and validation phase on the drone. As we aimed at learning a model on a smaller subset of the FLAME dataset and inferring that model, the default Tensorflow version is used here. Also, the image and mask dimension for each input is reduced to 128 x 128 x 3 rather than 512 x 512 x 3. To save more memory on the RAM, all peripherals were turned off and only WiFi was working at that time for the Secure Shell (SSH) connection. The setup of this node is:

    <img src="/assets/federated_node_cropped.jpg" width="500" height="500"/>
