---
title: "Tensorflow on RP4"
date: 2021-04-06T17:09:24Z
categories: ["DeepLearning", "IoT"]
tags: ["Raspberry Pi"]
draft: true
---
As part of the Coursera Specialization course on Tensorflow deployment, RP4 is used to demonstrate TF-lite. It it based on a somewhat older version of TF and other libraries, hence installing it on a fresh device took some effort. The object detection example asumed a RP-camera, which I did not have. I used an Intel Realsense D435 camera which required manual steps to install on the RP.
![Object detection with TF-lite on RP4](/uploads/object_detection_on_rp4.jpg)
There are many resources on the internet about installing TF2 on the PR, therefore I will not go into all details here, and focus on the tricky parts I bumped into.
## Preparing the RP
Because the object detection example requires video, I installed the "Raspberry Pi Desktop" image. This version will create a desktop environment. This desktop will be used fron a VNC client on the host laptop. In order to set things up properly, following yt can help (details at the end of the video) 
{{< youtube kxtT8PjPBik>}}
Upgrade apt to be up-to-date

## Installing Tensorflow
There are several ways to install TF on the RP. If only inference is relevant (which is mostly the case), installing the TF-interperter is sufficient. If more interaction with TF is required, full TF needs to be installed.

### Cross-compiling TF 
The prefered way to compile TF is to crosscompile it on a faster system. The resulting .whl file can then be copied on the RP where it can be installed with pip. I xcompiled it on my DeskMini pc (Core i5 16 GB) which took about 4 hours.
Although the resulting wheel could be transfered to the RP in order to install it there, I choose next approach:
### Install comunity maintained wheel package
Follow the instructions in :
[community maintained weel package](https://www.bitsy.ai/3-ways-to-install-tensorflow-on-raspberry-pi/)
in order to install the package. Instead of using 2.4RC2, use 2.4, it is available.

## Install RealSense D435 camera
[Install RealSense D435 camera](https://github.com/acrobotic/Ai_Demos_RPi/wiki/Raspberry-Pi-4-and-Intel-RealSense-D435)

## Install opencv
The object detection example in the Coursera course used the RP-camera API. Therefore another application that uses opencv in order to attach to the D435 was used:

[TFLite_detection_webcam.py](https://github.com/EdjeElectronics/TensorFlow-Lite-Object-Detection-on-Android-and-Raspberry-Pi/blob/master/TFLite_detection_webcam.py)
Change the index of the /dev/videoXX device in order to select the camera
```python
       self.stream = cv2.VideoCapture(4)
```
After the installation of the camera, they appeared as /dev/video[10-14]. None of them did work, the device could not be connected.
After unplugging the camera, they appeared as /dev/video[0-4]. Selecting index 4 selected the correct device.

Here a small helper program to check the camera:
```python
import cv2
cap = cv2.VideoCapture(4)
while True:

    ret, frame = cap.read()
    cv2.imshow('frame',frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```


### Links
[TF2 on Buster](https://itnext.io/installing-tensorflow-2-3-0-for-raspberry-pi3-4-debian-buster-11447cb31fc4)
 : Interesting blog, looks complete. Worthwile trying this for a next RP installation of TF

[TF-lite using usb-camera](https://github.com/EdjeElectronics/TensorFlow-Lite-Object-Detection-on-Android-and-Raspberry-Pi)
 : Interesting to see how the author uses MSYS in order to build TF from source

