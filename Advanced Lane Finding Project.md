## Advanced Lane Finding Project


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

[image1]: ./output_images/Calibration.png "Calibration"
[image2]: ./output_images/Undistort.png "Undistorted"
[image3]: ./output_images/Warped.png "Warp Example"
[image4]: ./output_images/Binaries.png "Binary Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./output_images/Polyfit.png "Polynomial fitting"
[image7]: ./output_images/result_0.png "Output image"
[video1]: project_video_out.mp4 "Video"

### Camera Calibration

The code for this step is contained in the "Camera Calibration" section of the python notebook, in the class Camera.

Camera was calibrated using a set of calibration images provided. This is a set of images of chessboards photographed under various angles and perspectives. 

For each of the calibration images, we used OpenCV function `cv2.findChessboardCorners` that identifies all the intersection grid points on the chessboard on the actual calibration image. We call this set of points `imgpoints`. Then, we identify a set `objpoints` that contains coordinates of those same chessboards corner but in the transformed image. The goal of calibration is to provide a matrix that can map `imgpoints` into `objpoints`.

Luckily, OpenCV provides the function for doing just that, called `cv2.calibrateCamera` that returns distortion matrix and distortion coefficients.

Now, we can use the matrix and cofficients to undistort any image taken with this camera by simply calling OpenCV function `cv2.undistort` on the image.

Here is an example of one of the chessboard pictures before and after we undistort it.

![alt text][image1]

### Pipeline (single images)

Here are all the processing steps for a single input image to achieve lane detection.

#### 1. Undistort

First, we calibrate the camera and then undistort the input image using the method described in Camera Calibration section. Here is an example of input image before and after undistortion. (is undistortion a word?)

![alt text][image2]

#### 2. Perspective transform

I reversed the order of processing steps in my project from the one suggested. I first did the perspective transform, to achieve bird-eye view of the lanes, and then followed with color and gradient thresholding. This helped me to focus less on all the noise around the road and make sure that my thresholds worked well for the lanes themselves.

Perspective transform works similar to camera calibration, we find points on the image that we want to map to points in the transformed image. I used GIMP to select four points on an image with **straight lanes**, two on the left lane and two on the right lane and hardcoded them in an array `src`. Then, I mapped them into a rectangle in the transformed image and stored those points in array `dst`:

```python
src = np.float32([
	[194, img.shape[0]], 
	[1112, img.shape[0]], 
	[573, 463], 
	[708, 463]])
dst = np.float32([
	[200, undistorted.shape[0]], 
	[1000, undistorted.shape[0]], 
	[200, 0], 
	[1000, 0]])
```

I then applied OpenCV `cv2.getPerspectiveTransform` to src and dst array and obtained perspective matrix that can be used to warp images using `cv2.warpPerpective` into bird-eye perspective of the lanes on the road.

Here is this transformation look like on the same, undistorted input image from above:

![alt text][image3]

#### 3. Color and Gradient Thresholding

After I had the bird-view images of my lanes, I spent quite a bit of time trying out different gradient and color thresholding tehniques. Most of the thresholding variations was described in the lectures, but it took a while to find a set of thresholds that seemed to work well for my test images. The final combo chosen for the pipeline is described in the function `thresholding_pipeline`, in the `Color and Gradient Thresholding` section of the notebook.

The combo is comprised of filtering on the Red channel of the RGB image, and the S channel of the HLS image. I tried including gradient, magnitude and directional thresholding as well, but I simply couldn't find a combination that looked better on these images.

It's quite possible that a set of better thresholds exist, but manual exploration takes time, and the only evaluation that I've done is visual inspection of the test images.

Here is how the lane images looked like after the thresholding was applied. These images are all binary, where 1s (whites) represent lanes, and 0s (blacks) represent the rest of the road.

![alt text][image4]

#### 4. Identifying lane lines

The next task was to identify which pixels of the image belong to the left lane and which to the right lane, given a binary image like the ones showed above.

First part of this task was to identify indeces of the binary image that correspond to left lane and right lane, and separate them out. The function that accomplishes that is defined in the class `ImageLaneFinder` and it's called `sliding_window_find_lane_indices`. 

This method was described in the udacity lectures. It first determines where the lanes start at the bottom of the image by taking a histogram of the bottom section of the image. There should be two peaks in the histogram, the first one belonging to the left lane, the second one belong to the right. We then use the sliding window that moves from the bottom to *follow* the lanes up, sectioning pixels into left and right side.

Once we have the indeces of left lane and right lane, we want to fit a quadratic polynomial to left lane indeces, and a separate line to right lane indeces. The function that does this is `ImageLaneFinder.fit_from_indices`. This returns two lines, that correspond to the two lanes. The idea is pictured here (this image is taken from the lectures, and was not generated by me):

![alt text][image5]

To enhance the performance, we try to minimize the number of calls to the sliding window method. We do this by leveraging the information about the previous known fit. If we have previous fit, we reuse those fits to obtain the indices faster.

Here are some images that show how pixels are split between left lane (RED) and right lane (BLUE) with the yellow line fitted through each of them (I did generate these!):

![alt text][image6]


#### 5. Lane curvature

Lane curvature was calculated using the radius of curvature equation given in the lecture notes. The code for it is in `ImageLaneFinder.measure_lane_curvature`. Since there are two curvatures, one for each lane, we display the average value of the two and calculate it in `ImageLaneFinder.measure_lanes_curvatures`.

#### 6. Position

Position of the car relative to the lanes is calculated by seeing how *off* the car is from the center. We assume that if the car is driving in the center of the road with equal distance from both lanes, each lane will be equally removed from the center of the camera image.

Therefore we calculate car's position by determining where the left and right lanes are at the bottom of the image, and seeing how far those positions are from the center of the image at 640 pixels. The code for this is inside the `unwarp` function in ImageLaneFinder.

#### 6. Final Output

In the final output image, we unwarp the image and color the area between the lanes green. Additionally we print Curvature and Position calculated at the top of the image. Here is how the final output looks:

![alt text][image7]

---

### Pipeline (video)

Here's a [link to my video result](project_video_out.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The first challenge of the project was manual tweaking of the gradient and color threshold and the exploration of different channel combinations, as well as gradient directions and magnitudes. 

The first problem is that the space is huge and trying out different things by hand takes forever, so a way to automate and parametrize this would be helpful. The obstacle for automatization is that we need a way to *evaluate* performance of each guess, and be able to compare and rank different performances. This could be achieved by having training images that were correctly labeled by hand, and instructing the algo to find a set of parameters that matches the lanes found the closest for the training images. 

The second issue, is that these parameters were chosen to work on a small set of images, and of course fail to generalize properly to images that don't match those, which is demonstrated by much poorer performance on the challenge video and potentially videos with different lighting conditions as well.


The second challenge was to decide when to discard a lane and how to combine previous lines fitted to calculate the current one. My approach was simple: I discarded lanes if the combined lane will result in either a curvature that was too great (I randomly chose 10000 as my threshold) or if the newly calculated lane would result in a line much different from the previously fitted line (again I chose some thresholds for each of the quadratic coefficients). I combined the lines by simply having the new line be the combination of 0.8 of the previous line and 0.2 of the newly calculated line. That seemed to work ok for the project_video.

These discarding conditions helped to improve the performance on the challenge video, but the challenge video still resulted in lanes curving outwards, so some additional constrains on the quadratic coefficients and making sure that these point in the same direction would be needed to improve the performance.

Finally, the discard conditions have helped me stay on track once the proper lanes were found, but I have not resolved an issue where right at the beginning the camera can't find lanes. It would probably be helpful to have some starting state that's different from what we do once the lanes have been found.