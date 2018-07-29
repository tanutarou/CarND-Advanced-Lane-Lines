## Writeup Report

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/distort.png "Undistorted"
[image2]: ./output_images/distort_road.png "Road Transformed"
[image3]: ./output_images/pipeline.png "Binary Example"
[image4]: ./output_images/warp_rgb.png "Warp Example"
[image5]: ./output_images/warp_bin.png "Warp bin Example"
[image6]: ./output_images/fitting.png "Fit Visual"
[image7]: ./output_images/visualize.png "Output"
[video1]: ./test_videos_output/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "solution.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

`cv2::findChessboardCorners` return zero if it failed corner detection. So I used this flag and saved only corners which is detected successfully.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one.
![alt text][image2]

* I got objpoints and imgpoints in before section. So, I can do distortion correction easily by using `cv2.undistort`(We just call `cal_undistort()` in the first code cell of the "solution.ipynb").


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at L14, L18 and L22 in the 3rd code cell of "solution.ipynb").

I used L channel in HLS colorspace, HSV color space and Sobel filter to create a binary image.
I thought HSV color space is useful to extract white segments and Sobel filter can extract solid line easily. Using L channel doesn't need to detect lanes for 'project_video.mp4'. But, I thought it need to detect lanes for other videos like 'challenge_video.mp4'.


* I selelcted the range [0, 0, 170] - [255, 25, 255] to extract white segments in HSV colorspace
* I selected the threshold [240, 255] for L channel
* I selected the threshold [20, 100] for Sobelx filter

Here's an example of my output for this step.  

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp_transform()`, which appears in lines 1 through 15 in the 4th code cell of "solution.ipynb".  The `warp_transform()` function takes as inputs an image (`img`). I chose the hardcode the source and destination points in the following manner(This is same to the value of writeup_template.md. Because, this code adopt to image size. It's good to use for videos with different size):

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the src and dst points onto a test image and its warped counterpart to verify that the lines appear roughly parallel in the warped image.

I think selecting the hardcode the source and destination points are not robust for any road scene. Perspective transformation can be done by using intrinsic and extrinsic camera. It may improve robustness of the transformation.


![alt text][image4]
![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To identify lane-line pixels, I used searching from previous polynomial approach(L59-L110 at 5th code cell of "solution.ipynb"). It decides search area is around previous polynomial curve. Because, lane curves doesn't change drastically in real. Surrounding range was decided by "margin" (L62 at 5th code cell of "solution.ipynb").

I used the points of this area(green area in the below image) for lane fitting.
I fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image6]

I averaged polynomial parameters in 10 frames. It make robust lane detection.

I made 'mirror lane' program too. It suppose left lane and right lane are exactlly parallel. I considered many points for fitting is higher confidence. So if the confidence of one side lane is high, mirroring it for other side make good results. But it was not effective for `project_video.mp4`. So, I commented out it.


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculated the radius of curvature of the lane at L54-L55 in 5th code cell of "solution.ipynb" and the position of the vehicle with respect to center at L40-L41 in the same cell.

I used the equation in the Lesson18.7(Measuring Curvature 1) for caluculating the radius of curvature of the lane. And I calculated the position of the vehicle with respect to center by using the midpoint at the bottom of the two detected lanes. Then, I converted from 'pixcel' to 'meter' by using the meter per pixel of x-axis is 3.7/700 and the meter per pixel of y-axis is 30/720.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step at L1-L24 in 6th code cell of "solution.ipynb"(`visualize_lane()`). 
I used inverse matrix of perspective transformation matrix for inverse transformation. It can be calculated easily by 'numpy.linalg.inv()'.
And, I displayed curvature and position on the image by using `cv2.putText()`.

Here is an example of my result on a test image:

![alt text][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./test_videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

* Problems I faced in this project
  * I faced the problem in the camera calibration phase. In `camera_cal` directory, corners in some pictures cannot be detected by 'cv2.findChessboardCorners()'. So, I need to judge the return value 'cv2.findChessboardCorners' whether it detected successfully.
  * I faced the problem in the color transformation phase. Firstly, I used S channel only. It works well for 'project_video.mp4'. But it doesn't work for 'challenge.mp4'. Because, S channel couldn't extract the information of right lane. So, I added L channel information which can extract the information of right lane easily.

* My pipeline might fail when the point of one side lane cannot be detected. It means there is no points for fitting. It may be throw errors. To prevent this problem, I need to use the previous polynomial for this case. My program is weak for no lane detection frame.
* My program uses previous information(previous polynomial for deciding searching area), but not sufficient. If I use famous tracking algorithm like Kalman Filter or Particle Filter, my lane detector may become more robust.
* My program select the hardcode the source and destination points for perspective transformation. Perspective transformation can be done by using intrinsic and extrinsic camera. It may improve robustness of the transformation.
