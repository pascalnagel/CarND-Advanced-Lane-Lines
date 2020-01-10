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
* Apply this pipeline to a video
* Use error metrics to determine whether to use the fit of a given frame or fall back to the last frame's fit
* Primarily use previous fits as starting values for the current fit, if the line is lost, reinitialize with histogram and sliding windows

[//]: # (Image References)

[image1]: ./output_images/undist.png "Undistorted"
[image2]: ./output_images/undist_road.png "Road Transformed"
[image3]: ./output_images/binary.png "Binary Example"
[image4]: ./output_images/trapezoid.png "Warp 1"
[image5]: ./output_images/warped.png "Warp 2"
[image6]: ./output_images/warped_binary.png "Warp 3"
[image7]: ./output_images/windows.png "Windows & Fit"
[image8]: ./output_images/final_image.png "Output"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the code cells 1-2 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

After importing the required packages, I define the object points which correspond to the real world position of each of the 9x6 chessboard corners in a given calibration image. They are chosen to be equidistantly spaced in the x-y-plane (steps of 1). I then loop through all calibration images in the `camera_cal` folder, load the image, convert it to grayscale, use `cv2.findChessboardCorners` to identify the coordinates of the 9x6 corners and, if successful, append the corners and the corresponding object points to respective lists.

Using `cv2.calibrateCamera` I then, from the lists of object and corner coordinates, calculate the camera matrix and distortion coefficients. 

In cell 2 we apply `cv2.undistort` to one of the distorted calibration images, using the obtained distortion coefficients and camera matrix, to check whether the image is now free of distortion:

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The code for this step is contained in code cell 3 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

The undistortion using the previously obtained distortion coefficients and camera matrix is carried out on one of the test images:

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for this step is contained in the code cells 4-9 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

I created a thresholded binary image for lane candidates using a combination of gradient and color thresholding. 

For color thresholding, I first transformed the undistorted image to HLS color space and extracted the s channel (cell 4). In a next step I created a binary image `s_binary` from the s-channel image, which is only true for pixels with a s-channel value between 170 and 255 (cell 8).

For gradient thresholding, I converted the undistorted image to grayscale (cell 5) and applied a horizontal sobel filter to the result (cell 6), to favorably extract vertical edges in the image. After scaling the resulting gradients to the range [0,255], I then created a binary image `sxbinary` from the scaled gradients, with only gradient values between 20 and 100 being true (cell 7).

Finally I combined both binary images with a logic and (cell 9):

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for this step is contained in the code cells 10-14 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

To carry out the perspective transformation to birds-eye view, I defined source and destination coordinates as follows:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 570, 465      | 300, 300      | 
| 712, 465      | 1164, 300     |
| 1084, 700     | 1164, 1164    |
| 220, 700      | 300, 1164     |

To not loose any accuracy to downsampling I chose the spacing between the left and right lane lines at the bottom of source and destination images to be the same (864px). The margin of 300px to either side of the lines was a good trade-off between loosing the lines too early in corners and getting noise from other lanes. To fit the above source coordinates to the lane lines, I used `cv2.polylines` for visualization (cell 11):

![alt text][image4]

Having obtained source and destination coordinates, I then evaluated the perspective transformation matrix `M` and its inverse `Minv`, using `cv2.getPerspectiveTransform` (cell 12). In the next step I warped the undistorted image using `cv2.warpPerspective` to obtain a birds eye view image (cell 13):

![alt text][image5]

and readjusted the source coordinates to make sure the lines are as vertical as possible. In the final step, I step applied the perspective transformation to the previously obtained binary map (cell 14):

![alt text][image6]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for this step is contained in code cell 15 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

To identify lane-line pixels in the image, I used a sliding window approach starting at the bottom of the image. The position of the lines at the bottom of the image can be identified somewhat robustly by generating a histogram of the number of true pixels in the final row of the image. The maximum histogram value in the left half of the image is taken to be the left line base position and vice versa for the right line. I then positioned the first two windows of width 100px with their center on the two base positions. The height of the windows is chosen to make 9 windows fit on top of each other exactly inside the image. For each of the 9 windows and both left and right line, I now carry out the following steps:

 - Identify which binary true pixels are inside the window and append their coordinates to a list
 - If more than 50 true pixels are inside the window, change its x-coordinate to the mean x coordinate of the true pixels
 - Stack the next window on top of the current one and repeat

After having looped through all 9 windows, we obtain a list of pixel coordinates, which we estimate to belong to the lines. These we can now fit with a 2nd order polynomial using `np.polyfit`.

Finally, we can visualize the lane pixel candidates, along with the windows and fitted curves:

![alt text][image7]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code for this step is contained in the code cells 16-19 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

To calculate the radius of curvature and the car offset in meters, I first needed to define how many pixels in the birds-eye image correspond to 1m (cell 16). The choice of 50/1164 m per pixel in the vertical and 3.7/864 m per pixel in the horizontal were chosen by estimating the width and length of the source trapezoid. The width of 3.7m was taken from the lane-width specifications and the 50m were approximated by taking into account the length and spacing of dashed lane lines.

The radius is then directly calculated in meters (cell 17). For evaluation the left and right curve radii are then calculated for the straight lines (cell 18), yielding 43km and 77km respectively, a perfect straight line corresponding to an infinite curve radius.

To calculate the offset (cell 18), we can use the fit polynomials of both lines to determine their bases and consecutively determine the x-coordinate of the middle between them, given by `lane_center`. Now that we know at which x-coordinate the lane center is, we need to evaluate the x-coordinate of the camera `camera_pos`, which is taken to be in the center of the image. Finally the difference between both is the offset in pixel. The offset in meters can then be evaluated simply by multiplication with the pixel-to-meter conversion factor above.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The code for this step is contained in the code cells 20-21 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

Using both polynomials fits, we calculate all line pixel coordinates and use `cv2.fillPoly` to draw a filled polygon between them, filling out the area between the lane lines. Finally we invert the perspective transformation on the polygon mask and add it to the undistorted input image. Lastly, I added a text overlay for the radius and vehicle offset, negative values being an offset to the left of center and positive values to the right.

![alt text][image8]

---

### Pipeline (video)

The code for this step is contained in the code cells 22-27 of the IPython notebook located in "./AdvancedLaneLines.ipynb".

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

[Link to annotated video](./test_videos_output/project_video_annotated.mp4)

#### 2. Describe how you discovered the lines in the first frames of the video.

I started by creating a class `Line()` and instantiating it once for the left and once for the right line, to track information across frames (cells 22-23).

The pipeline (cell 24) mainly follows the steps for single images above, with a few modifications. If the line has not been correctly detected for 5 or more frames in a row and at the beginning of the video, we use the above histogram and sliding window approach to obtain line candidates and then fit the polynomial to them.

Furthermore, the fit coefficients and radii are averaged over 3 frames, making the lines noticably smoother.

#### 3. Describe how you tracked the lines in frames following a detected line.

If the line has been correctly detected in the previous frame, all true pixels in the binary image, which are within a margin of 100px around the previous fit, are used to fit the polynomial for the current frame. If the line has not been detected, but for less than 5 frames, the frame is simply skipped and the fit of the last frame is kept.

Finally, if a line was misdetected is determined by the normalized RSS error of the fit. If the fit is worse than a RSS of 10000, the fit is determined to be bad.

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Additional robustness could be achieved by implementing more sanity checks for line misdetection. If all we are interested in is the radius of curvature close to the car, reducing the detection length from 50m to something shorter would make the fit more robust.

This entire approach relies heavily on having no significant incline/decline in the road, which would lead to changes in radius, since we have not properly taken it into account.

My implementation yields poor results on the challenge videos. Some of the issues, like gradients in the middle of the road, could be avoided by additional hand-crafted rules for selecting the line bases, etc.