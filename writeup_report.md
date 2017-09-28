## Writeup Template

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


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

Camera calibration is performed in the third cell of the IPython notebook. Multiple calibration images are used, each contains a distorted view of a 9x6 chessboard. 
The algorithm correlates the actual positions of the corners of the chessboard boxes to expected positions in a perfect (undistorted) chessboard it generates of size 9x6.
It then produces a transformation matrix and some coefficiens which describe the transformation required to correct the distortion. 

In cell 4 I visualize the result by undistorting one of the calibration images. 

![Example calibration result][output_images/calibration_result.png]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

I defined a function called 'undistort' which uses the previously computed coefficients (cell 3) to undistort any input image (assuming it came from the same camera). 

I visualize the result in cell 4:

![Example calibration result][output_images/calibration_result.png]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

This is performed in a cell under the title "Extract features, threshold and combine".  
I process the RGB image in two color spaces. 
First, in grayscale I am performing:
1. Sobel (edge detection, in my case vertical edges)
2. Gradient magnitude threshold (magnitude of sobel edge detected in both horizontal and vertical planes)
3. Gradient direction threshold - I'm mostly interested in vertical or near vertical edges to find lane lines

Second, I convert the image to the HLS space (after experimenting with some other spaces, this have good results).
There I apply a threshold to the saturation (S) channel.

The pixels found in all the above steps are then combined (OR logic, i.e. any rule match is ok).

I then visualize the results in the same cell on sample images:

![Feature extraction][output_images/feature_extraction.png]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Perspective transform appears in my code before feature extraction. It is configured by a cell under the title "Configuration for perspective transform" and performed by a cell under the title "Calculate trapezoid and matrices for perspective transform". The following cell visualizes the results.

I selected a trapeze that describes the area of interest in the road in the perspective view. From that I compute (using cv2.getPerspectiveTransform) a transformation matrix that describes how to transform the same area to a top down view. I also compute the inverse transform to later use in video overlaying. 

The next cell visualizes the selected trapeze and transform on two sample images:

![Perspective Transform][output_images/perspective_transform.png]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I do a search for the base of the polyline at by searching for histogram peaks (`find_starting_position_unhinted`).
From there I do a sliding windows search for where the polyline most likely proceeds (direction is father away from vehicle) (`search_lane_pixels`).
Pixels found within the sliding windows are then used to fit a 2nd degree polynomial (`fit_polynomial`). 

I then plot the results:

![Fitting a polynomial results][output_images/polynomial_fitting.png]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature we're interested in is in meters, not in pixel space. To compute it we need to transform pixel space to world space. This is done by assuming something about the field of view of the camera. 
The assumption is that our field of view is 30 meters vertical (depends on the trapeze - from camera to top of perspective transform trapeze), and 3.7 meters horizontal.
Radius is then computed from the newly fitted polyline (in world (i.e. meter) space). See the `calculate_curvature` function for implementation. 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.


![Example of overlay][output_images/overlay_example.png]]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_images/project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The time it takes to process a frame is over 20ms, which means it's not realtime. The main problem (detected by profiling with %prun) is the undistort function. There are strategies to avoid full undistort and replace it with faster methods, especially since the impact on the end result is quite small. During development at some point I just disabled undistort so I can process videos faster. 

Python type system is horrible, the errors are unhelpful when array shapes or tuples are involved.

I am not taking into account multiple factors:
1. Lane color is relatively constant once detected
2. Lanes have expected geometry - either it's a continuous line, or equally spaced equal sized lines. Also, lane has an expected width, that I can use to filter out false positive pixels. 
3. Lanes have a relatively constant distance between them.
4. Lanes are in the same position on the road, so if I detect travel in relation to lane center, I can expect the lane lines to be shifted accordingly in the image. 

It's easy to break the pipeline, and I didn't have time to address them:
1. Presense of shadows on the lane markers
2. Light and dirt artifacts on the car window before the camera
3. Artifacts on the road which can show us as gradients in the feature extractor (for example old markings, asphalt fixes, etc.)
4. Broken lane lines and/or cars crossing lane lines
5. One of the two lane lines not showing due to sharp turn (present in the harder challenge video) 

Adaptive approach to shadows can:
1. Divide the image into strips
2. Try to adjust feature extraction thresholds for maximum detection in each strip, searately.
3. Above adjustment can be made by trying different values and choosing the one with maximum detection, or using something like a histogram to detect road colors (common) vs lane colors (peak of rare color)
4. The search area can be limited to around previous detections of lanes, this would eliminate false positives for bases, especially in the harder challenge video
5. If lane pixel search doesn't find enough pixels, try another peak

Simple technique I started to implement (but stopped due to time constraint, and I assume that it's not required according to the quality of output on the project video) is averaging the lines (i.e. averaging the base for searching between frames, and averaging the curvature between frames). This would have made the pipeline more robust to small errors in specific frames. 
