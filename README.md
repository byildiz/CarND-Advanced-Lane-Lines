# Advanced Lane Finding Project

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

[image1]: ./output_images/calibration1_undist.png "Undistorted"
[image2]: ./output_images/straight_lines2_undist.png "Example distortion correction"
[image3]: ./output_images/test4_thresholded.png "Example combined binary"
[image4]: ./output_images/straight_lines2_warped.png "Example perspective transform"
[image5]: ./output_images/test2_pixel_searching.png "Visualization of found lines"
[image6]: ./output_images/video_0005_warpedback.png "Example of warping back onto the original image"
[image7]: ./output_images/separation_diff.svg "Example of warping back onto the original image"
[video1]: ./project_video_output.mp4 "Result for project_video.mp4"
[video2]: ./challange_video_output.mp4 "Result for challenge_video.mp4"
[video3]: ./harder_challenge_video_output.mp4 "Result for harder_challenge_video.mp4"

## [Rubric](https://review.udacity.com/#!/rubrics/1966/view) Points

### Here I consider each rubric points individually and describe how I addressed each point in my implementation.

---

### Writeup / README

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project. Where will your pipeline likely fail? What could you do to make it more robust?

I wrote this write up and discussed the problems and issues I faced at the end of the writeup.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the IPython notebook located in "./Camera-Calibration.ipynb" file.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function. I saved the calculated camera calibration parameters in a pickle file for later usage. I applied this distortion correction to one of the calibration images that was not used to calculating parameter due to the missing corners to verify correction using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

The result shows that the correction works.

### Pipeline (test images)

The code belonging to the rest of this writeup is contained in the IPython notebook located in "__./Advanced-Lane-Finding.ipynb__" file and sections mentioned above are in this file as well.

#### 1. Provide an example of a distortion-corrected image.

I applied the distortion correction to "./test_images/straight_lines2.jpg" and here is the result:

![alt text][image2]

The corresponding code is in _Section 1_ in the IPython file.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image. Provide an example of a binary image result.

I wrote two different thresholding functions (`threshold()` and `threshold2()`) in _Section 2_ in the IPython notebook. I used the second one in the project since it is more robust than the first one. It combines color and gradient thresholds. I used the V channel of LUV color space and the B channel of LAB color space to threshold white and yellow lines respectively. When calculating the gradients, I used an averaged gray scale of V and B channel to decrease false positives. I applied the thresholding function to all of the given test images and the following image is one of the resulted image.

![alt text][image3]

The rest of the outputs of thresholding of the test images were saved to `.output_images/*_thresholded.png`.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I hardcoded source and destination points used to calculate transform matrix. These are my source and destination points:

```python
src = np.float32([[576, 460],
                  [708, 460],
                  [1120, 720],
                  [207, 720]])
dst = np.float32([[320, 0],
                  [960, 0],
                  [960, 720],
                  [320, 720]])
```

In the _Section 3_, I wrote two functions. The first one is `calculateTransformMatrix()` and this function takes source and destination points and calculates transform and inverse transform matrices. The second one is `warp()` function. This function takes and image and transform matrix and warps the image using given matrix.

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for identifying lane-line pixels and fitting a polynomial to identified pixels was written in _Section 4_ in the IPython file.

First of all, I defined a `Line()` class to track some useful information like found pixel positions, fitted polynomial function and radius of curvature at the bottom of image, etc.

Second, I defined a function named `fit_polynomial()` to fit a second order polynomial for given x and y positions. The function actually takes a line of which lane-line pixels was found and positions of them are stored in `allx` and `ally`. It fits a second order polynomial by using `np.polyfit()` and calculates x positions corresponding to y positions to not calculate each time if they are needed. I think that it is more practical to use `np.poly1d()` to calculate points and I stored polynomial version of fitted line instead of storing just coefficients.

Third, I defined a function to find lane-line pixels in a given warped binary image. The function name is `find_line_pixel()` and it only finds left or right line each time is called because sometimes it is not required to find both left and right lines at the same time. For example, left line could be found in previous video frame and it should be searched around previously found fit.

The function takes a binary image, a line to store found pixel positions and a flag named `left` which is `True` if we want to find left line. The function calculates the histogram of the bottom half of binary image and identifies a base point to start to search line pixels. Then, it finds all pixels in the window centered on the base point and iteratively goes up of the image by repositioning the window and updates base positions if the number of found pixels is more than the predefined hyperparameter named `minpix`.

Lastly, I defined a function named `draw_lines()` in `Section 5`. The function draws the found line pixels, searched windows and fitted polynomials of left and right lines. It was also designed for future usage like visualizing best fit and search area based on previously found fit.

Here is an example output of the `draw_lines()` for the test image `test2.jpg`:

![alt text][image5]

For more examples, please see the images named `*pixel_searching.png` under the `./output_images` folder.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In `Section 6`, I defined four functions.

The first one is `calculate_curvature_rad()`. The function takes two inputs, `y_eval` and `c` respectively. `y_eval` is the y position in pixels of which we want to calculate the curvature of polynomial and `c` is the polynomial coefficients of the fitted polynomial. Since the coefficients are calculated based on in pixels instead of meters we must calculate the first and second order derivatives a little bit different but the rest of the radius calculation is same.

The second function is `calculate_dist_to_center()`. The function takes `y_eval`, `car_center` and `x_values` all of them are in pixels and calculates the absolute distance of line at the `y_eval` to the car center in meters.

The third is `calculate_line_props()`. This function calculates all will-use properties of a given line by calling 2 functions explained above. Further more, it also calculates line slopes in meters at all y positions.

The final function is a printing function named `print_lines()` which prints calculated properties of lines. An example output for the given test images is as follows:

```
test_images/straight_lines1.jpg
Radius:   2923   5197 Base pos:   1.84   1.79 Slope:  -0.01   0.01

test_images/straight_lines2.jpg
Radius:   2650  85219 Base pos:   1.80   1.79 Slope:  -0.00   0.00

test_images/test1.jpg
Radius:    310    720 Base pos:   1.69   1.94 Slope:   0.00  -0.01

test_images/test2.jpg
Radius:    284    394 Base pos:   1.53   2.16 Slope:  -0.01   0.01

test_images/test4.jpg
Radius:    612    130 Base pos:   1.59   2.37 Slope:  -0.00   0.07

test_images/test3.jpg
Radius:    536    409 Base pos:   1.74   1.94 Slope:  -0.03  -0.00

test_images/test5.jpg
Radius:    222   1159 Base pos:   1.99   1.90 Slope:   0.00  -0.01

test_images/test6.jpg
Radius:    512    337 Base pos:   1.66   2.16 Slope:  -0.03   0.01
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I combined this step with putting lane information onto the image and implemented this step in _Section 9_ and _Section 10_ with two functions.

The first function is `draw_lane()` and it visualize the found lane on to the given image. It first creates an empty image and draws the lane onto it and then warps back it using inverse transform matrix and combines it with the original image.

The second one is `put_text()`. This function calculates the radiuses of curvatures of both left and right lines and averages them to find lane radius of curvature and puts it as a text onto the given image. In addition this, it also calculates car position with respect to the lane center and creates a text describing the position of the car and puts it onto the image as well. The car position equals to half of the difference of line base positions.

The following image is the output of `draw_lane()` and `put_text()`:

![alt text][image6]

Again for more examples, please see the images named `*warpedback.png` under the `./output_images` folder.

---

### Pipeline (video)

#### 1. Provide a link to your final video output. Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!)

The code for this step is written in _Section 8_ and _Section 11_.

In _Section 8_, There are a number of functions related to the sanity checking and tracking lane. Let's have a focus on them first.

`sanity_check_prev()` function controls the newly found line by comparing its properties with the last successfully found line in terms of base position change, maximum slope change, maximum x distance (maximum separation) and relative radius of curvature change and returns `True` if it is a valid line.

`sanity_check_other()` function also controls the newly found line but this time by comparing its properties with the last successfully found line of other side in terms of lane width (should be around 3.7m), maximum slope change, maximum x distance (maximum separation) and relative radius of curvature change. In addition to those comparisons, this function also checks whether all of x positions of the polynomial of the left line must greater than the x positions of right line. It returns `True` if the line passes all check points.

To find the correct values for the hyper parameters for sanity checking, first I run the pipeline on `challenge_video.mp4` without sanity checking and printed out all properties of found left and right lines. Then, I imported the output text to excel and draw the graph of hyper parameters and lastly decided the values of hyper parameters. An example graph for maximum separation between newly detected lines and last successfully detected one for the first 5 seconds of `challenge_video.mp4` is as follows:

![alt text][image7]

I chose maximum separation 0.5m since I know that the car went through underpass between the time span around from 4s to 5s and the separation values for that time span is not valid.

`update_best()` function updates best fit by adding some fraction of last best fit and newly found line fit.

`update_last()` was written to update last successfully found line and if newly found line is passes sanity checking returns newly found otherwise first increases number of lost frames by one and returns last successfully detected line.

In `detect_line()` function, a new line detection pipeline was defined. The function takes the inputs which are binary warped image, last successfully detected left and right line, left flag to select left or right side and a debug flag to print useful information while debugging and output of the function would be the newly detected line. It first creates a new line and checks the last detected line and decides whether line pixels should search around the last detected or scratch detection is required. After finding line pixel, a polynomial is fitted to the found pixels, line properties are calculated and sanity checking is done and finally if line passes the sanity checking, the best fit is going to be updated.

`process_image()` function is the main pipeline for video frame processing. It takes a video frame, find lane lines in the frame and draws lane and puts texts describing the radius of curvature and car position onto the original frame and lastly returns the modified frame.

The following links are the resulted new videos for each of test videos:

- [Result for `project_video.mp4`](video1)
- [Result for `challenge_video.mp4`](video2)
- [Result for `harder_challenge_video.mp4`](video3)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project. Where will your pipeline likely fail? What could you do to make it more robust?

The result for `project_video.mp4` was reasonably good. On the other hand the result for `challenge_video.mp4` was not as good as the result for `project_video.mp4` since false positive found pixels is slightly changes the left line frame by frame and sanity check detects the anomaly late. This problem can be solved by using another curve fitting method, finding a better way to threshold or finding a way to filter out pixels not belonging the lane lines. In addition, lane line pixels were not detected when the car went through underpass. To solve this problem, the thresholding should be improved.

The result for `harder_challenge_video.mp4` is, however, terrible. I must improve pixel finding function `find_line_pixel()` because the line sometimes goes out left or right sides after warping and the searching should stop at that points instead of continuing to search upwards. It causes to catching false positive pixels. I also must improve the thresholding in such a way that reduces the illumination and reflection problems. I don't know any method for know to reduce the illumination and reflection problems. There may be also need to adjust the sanity checking hyper parameters.

After a number of frames that the pipeline could not detect the lines (etc. 10 frames without detection), the pipeline decides to search from scratch by starting to create histogram and goes on. In such a case as described, pipeline accepts the found lines without sanity checking and it may cause the fatal detection. I tried sanity checking but this time the newly found never passed the checking and the pipeline lost the line constantly. I also didn't find a better solution than skipping sanity checking to this problem.