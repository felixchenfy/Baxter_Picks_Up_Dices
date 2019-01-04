

This is a README for the **computer vision section** of the Final project of ME495-ROS: **Baxter Plays Yahtzee**  

Key ideas:
* Get relative pose between table and Baxter by detecting a chessboard on table surface and then call PnP.
* Detect Dices based on using two graph-based image segmentation algorithms, and other image processing techniques.
* Wrote a ROS node to provide the above services for the main program.


Links:   
1. [Main repo of the project](https://github.com/mschlafly/baxterplaysyahtzee).   
2. [Video demo](https://www.youtube.com/watch?v=vOceYSICtQc).
As seen in the video, the [Baxter robot](https://en.wikipedia.org/wiki/Baxter_(robot)) detected the dices on table, picked them up, and put them into a cup.

Group members of the project: Vikas, Millicent, Zhicheng, Joel, Feiyu(me)


## 1. My work

I wrote this separate README to describe only the work I've done, which is mainly about computer vision as summarized at the beginning of this README.  

## 1.1 A simple GUI and test of PyKDL's IK
See this [github folder](https://github.com/mschlafly/baxterplaysyahtzee/tree/master/src/BaxterGUI).

I wrote a simple GUI for (a) viewing Baxter's current state and (b) for easier testing IK. The code might be helpful if you want to write a GUI using Python tkinter.

I used [PyKDL](http://sdk.rethinkrobotics.com/wiki/Baxter_PyKDL) for solving IK. The result was not good: When the desired end-effector pose is far from current pose, the IK solver might return "no solution". Jarvis guessed that it might due to the local minima caused by self-collison and joint limits during numerical solving.  

I didn't look deep into why this IK failures, but I heared several solutions: 
* Try different initial seeds. 
* Use [Moveit!](https://moveit.ros.org/). 
* Build Baxter's model and solve by yourself, either Numerical method (Newton-Raphson), or analytical method (see [this](https://www.cds.caltech.edu/~murray/books/MLS/pdf/mls94-complete.pdf), p99, suggested by my frined Pengfei).

IK was not my role for the project. I did this only because I was quite free at that time, which was two weeks before deadline. I didn't expect that at last we almost couldn't finish the project in time.

## 1.2 Computer Vision (CV)
See this [github folder](https://github.com/mschlafly/baxterplaysyahtzee/tree/master/src/cv).

I wrote a node ([src/cv/nodeCV.py](https://github.com/mschlafly/baxterplaysyahtzee/blob/master/src/cv/nodeCV.py)) which provides 5 services:

1. **/mycvGetObjectInImage**  
Detect the dice in the middle of the image (from Baxter's left hand camera), return 2d-pos in image. (By OpenCV's GrabCut.)

2. **/mycvGetAllObjectsInImage**  
Detect all dices in image, return their 2d-pos in image. (By another [Graph-based segmentation algorithm](https://github.com/luisgabriel/image-segmentation), then finding squares). 

1. **/mycvCalibChessboardPose**  
Calibrate the pose of table's surface: If there is a chessboard in image, calculate its pose wrt camera, and then obtain its pose wrt baxter's base frame.  
(By OpenCV's **findChessboardCorners**  and **solvePnPRansac**.)

1. **/mycvGetObjectInBaxter**  
Call **/mycvGetObjectInImage**, and then transform pixel pos to world pos.  
This is done by finding (solving) the intersection between **table surface** and **the ray from camera center to object pixel point**.

5. **/mycvGetAllObjectsInBaxter**  
Call **/mycvGetAllObjectsInImage**, and then transform all objects' pixel pos to world pos. Use method same as above.

Functions used in the above (1)(2) are defined in [src/cv/lib_cv_detection.py](https://github.com/mschlafly/baxterplaysyahtzee/blob/master/src/cv/lib_cv_detection.py). The code for graph-based segmentation is copied from [this work](https://github.com/luisgabriel/image-segmentation) and put in [/src/cv/lib_image_seg](https://github.com/mschlafly/baxterplaysyahtzee/tree/master/src/cv/lib_image_seg/) after some small modifications..

Functions for (3)(4)(5) are in [src/cv/lib_cv_calib.py](https://github.com/mschlafly/baxterplaysyahtzee/blob/master/src/cv/lib_cv_calib.py).

# 2. Details about CV in this project

## 2.1 Camera Calibration

We only used Baxter's left hand camera, whose camera info is already in its topic. 

However, the Baxter's head camera_info inside the topic was not valid, whose distortion coefs are all zero. Because I at first thought we needed head camera, I went to calibrate it using Python. See [here](https://github.com/mschlafly/baxterplaysyahtzee/tree/master/src/cv/camera_calibration). 


### 2.2 Get the Pose of Table Surface

Before running code: Put a chessboard on table surface. Manually measure chessboard corners' real pos in chessboard frame. 

Inside the program, use OpenCV funcs to find corners' pos in image, and solve **PnP** to get the transformation from **camera frame** to **chessboard frame**. By left multiplying another matrix, we get the transformation from **Baxter base frame** to **chessboard frame**.

### 2.3 Locate Object Pos in Baxter Frame
Suppose we get the object pos in image. We know the object is on the **table surface (left side of equations)**, and it's also on a **beam (right side of equatiosn)** shooted from camera's focal point. We solve the equations (X1=X2, Y1=Y2, 0=Z2) to get X1, Y1. These are the object's pos in Chessboard frame. Then transform it to the Baxter's frame.

### 2.4 Detect All Dices in Image
Since thresholding method is trivial and troublesome, I searched for some automatic image segmenation algorithms, and then found this one, a Graph Based Image Segmentation ([paper](http://cs.brown.edu/people/pfelzens/segment/), [code](https://github.com/luisgabriel/image-segmentation)). 

After segmenting and labelling, regions with square shape are the dices. I tried these two: cv2.fitEllipse, cv2.minAreaRect. The first one is more robust, but might return a wrong value of square's direction. Vise versa.

The camera image is resized to half size, 320x200, before feeding into the algorithm. It takes about 4s for segmentation.

If the dice's color and the table's color are enough distinguishable, this method works well.


### 2.5 Detect One Dice in Image

If we know there is a dice in the middle of the image, we first define a potential region it might be in, and then use Grabcut to segment it out.

### 2.6 Number of Dots on Dice (not implemented)
I tried OpenCV's Blob Detection to detect dots, but it works bad. The dots on dice are so small and unclear.

### 2.7 Problems

The current performance of our computer vision code doesn't perform as robust as expected.

For detecting dice, sometimes only some of the dices are detected. In an ideal condition, with sufficient lighting and uniform table color, the algorithm should detect all dices. However, images from Baxter are dark, and there are also shadows, which make things bad.

For locating dice, there are about 2cm error when the Baxter's hand is 20 cm above the table. It comes from two folds:  
    1. The detected square region in image is not the accurate countour of the real dice.  
    2. The pose of Baxter's camera frame might not be so accurate.  
To solve it, after first locating, we move the camera lower and above the dice, took a second image, and locate the dice again. This improved the performance. The error was within 1cm.

Besides, I forgot two CV things in this project:  
1. I should have increased the camera exposure time, so that the image can be brighter. This can greatly improve the detection accuracy. Besides, We team have dicussed about adding lights, but forgot it at last. 
2. I forgot to undistort the dice's 2d pos in image before tranferring it to 3d. This will increase a little bit error.
