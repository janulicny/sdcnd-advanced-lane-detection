## Writeup Template


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



[calib]: ./output_images/calib.png "Calibration"
[calib_real]: ./output_images/calib_real.png "Calibration real image"
[detected]: ./output_images/detected.png "Detected"
[trans1]: ./output_images/transform_1.png "Transform 1"
[trans2]: ./output_images/transform_2.png "Transform 2"
[birdview]: ./output_images/birdview.png "Birdview"
[fitted]: ./output_images/fitted.png "Fitted"
[final]: ./output_images/final.png "Final"
[eq]: ./output_images/eq.png "Radius equation"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second code cell of the IPython notebook located in "./P3-Advanced_lane_finding-submit.ipynb"

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![calib]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![calib_real]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a color threshold filter in HLS space to generate a binary image. The code for this step is contained in the fifth code cell of the IPython notebook located in "./P3-Advanced_lane_finding-submit.ipynb"

1. Convert the image from RGB to HLS colorspace.
2. Segment the lane markings.
  1. Usually the white and yellow are the brightest colors in the image and we can just threshold for high values in _lightness_ component.
  2. But I ran into trouble with yellow detection in dark or very light images. So another branch just for yellow is needed. We threshold in _saturation_ component, but we also assume that the lane is fairly bright, so we demand _lightness_ to be over certain value. (There are no yellow lanes on the example image, but you can see that the yellow road signs were selected.)
  3. Combine the two filters with `or` operator.
3. Morphological operation _Closing_ is performed to get rid of small holes in the detected lanes.
4. Keep only region of interest. The detection area is defined with a trapezoid with following vortices:
  * lower left corner
  * lower right corner
  * lower 40% of image height, 55% of image width
  * lower 40% of image height, 45% of image width

Here's an example of my output for this step. 

![detected]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

First, I needed to search for appropriate source and destination points for the transformation. I have used these two provided test images of straight lines to find these points:

![trans1]
![trans2]

I chose to implement the source and destination points independently on the image size in the following manner:

```python
top = int(0.6*img.shape[0])
src = np.float32(np.array([[int(0.16*img.shape[1]),img.shape[0]],
                [int(0.486*img.shape[1]), top], 
                [int(0.514*img.shape[1]), top], 
                [int(0.86*img.shape[1]),img.shape[0]]]))
dst = np.float32(np.array([[int(0.3*img.shape[1]),img.shape[0]],
                [int(0.3*img.shape[1]), 0], 
                [int(0.7*img.shape[1]), 0], 
                [int(0.7*img.shape[1]),img.shape[0]]]))
```

This resulted in the following source and destination points for 1280x720 images:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 622, 432      | 384, 0        | 
| 205, 720      | 384, 720      |
| 1101, 720     | 896, 720      |
| 658, 432      | 896, 0        |


The code for the birdview transform is contained in the code cell 7 of the IPython notebook located in "./P3-Advanced_lane_finding-submit.ipynb"

Here is na example of the birdview transfrom applied to a test image:

![birdview]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code in the code cell 10 of the IPython notebook located in "./P3-Advanced_lane_finding-submit.ipynb" is used to fit the lanes with parabolas and return fit coefficients and lane curvature and centering of the vehicle in between the lanes.

1. Column histogram of likely lane pixels is calculated on lower half of the image.
2. The maximum value for the histogram on the left or right side is considered to be starting point of the left or right lane respectively.
3. Utilize sliding window technique to select the pixels belonging to left or right lane. The search starts at the detected base positions of the lanes, a window is created at that position and every likely lane pixel in it is considered to be really part of the lane. Then mean value of the pixels in window is calculated and the this value is used as center point for the next window taht will be built on top of this one. This process is repeated until top of image is reached.
4. Now we have two sets of points, one for left and one for right lane. Each set is fitted with parabola using `numpy.polyfit` function.

The radius of the lines is calculated in following way:

1. Determine conversion between pixel and real world dimensions.
	* Usual width of lane on US highway is 3.7 m --> 1 px = 0.72 cm in horizontal direction
	* Usual distance between dashed lane markings is 40 feet, there are approx 7 lane markings --> 1 px = 11.7 cm in vertical direction
2. Parabolas are again fitted, but now through the points transformed into real world units.
3. Curvature at the base of the image is calculated using this equation: 
![eq], where A and B are coefficients of the fit.

The camera is assumed to be mounted at the center of the vehicle, so the calculation is pretty straight-forward, I just compare the difference of the base positions of the lanes with the middle of the image and return the difference.​​​​

Here is an example of birdview binary image fitted with parabolas and overlayed values for left and right curvatures and lane centering:

![fitted]


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in code cell 12 of the IPython notebook located in "./P3-Advanced_lane_finding-submit.ipynb".

Here is an example of my result on a test image:

![final]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### Shortcomings of my current pipeline and ideas for improvement

* varying illumination of the scene
  * adaptive thresholding
  * better tuning of thresholds
  * add gradient filter

* jerky behavior in videos
  * implement memory of past lanes
  * assume that new lane position shall not move much in respect to the old one

* lanes outside of the region of interest (e.g. crossing lanes, driving up/downhill, etc.)
  * develop another method for selecting ROI

* detection of really curved lanes
  * higher order polynomial regression for fitting the lanes

* computation time
  * try to get to real time execution capability 
