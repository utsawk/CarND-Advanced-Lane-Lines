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

[image1]: ./output_images/camera_cal.png "camera calibration"
[image2]: ./output_images/flow_chart.png "Flow chart"
[image3]: ./test_images/test5.jpg "test image"
[image4]: ./output_images/undistorted.png "Undistorted image"
[image5]: ./output_images/threshold.png "threshold image"
[image6]: ./output_images/mask.png "mask"
[image7]: ./output_images/warp.png "warp"
[image8]: ./output_images/lane_lines.png "lane lines"
[image9]: ./output_images/final_output.png "final image"

### Files

My project includes the following files:
* P4.ipynb (Jupyter notebook) containing all the code for running the file
* project_video_output.mp4 is the project video output with lane lines superimposed
* output_images folder includes output images from different functions in the pipeline
* README.md/writeup.md summarizing the results
* test_images folder includes some images for testing the pipeline


### Camera Calibration

The code for this step is contained in the second code cell of the IPython notebook "P4.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function. Both the test image and undistorted image are shown below.

![alt text][image1]

### Image Processing Pipeline 
The overall pipeline is shown in figure below.
![alt text][image2]
I will next describe each step of the pipeline.

#### Distortion correction.

Both the test image and the distortion corrected image are shown below.
![alt text][image3]
![alt text][image4]


#### Combining R, S and sobel-x operator

I tried a combination of all thresholds: S and sobel (x, y, magnitude and direction). I found a combination (bitwise OR) of S and sobel-x to be most useful. However, these were also picking up the shadows causing a lot of bad unity pixels in the binary image. I observed that R channel does a good job in filtering out shadows and decided to use it. I combined the former with the later using an AND operation. The code for this is in function `thresholding()` in the 4th cell of the code. The output of each of these is shown in figure below.
![alt text][image5]

However, with this, I had to play with numerous number of sanity check functions to remove outliers. None of the sanity checks were good for all the frames in the video. This is because the perspective transform (described later) itself is a flawed operation leading to situations where either the radius of curvature was significantly different for the left and right lines or the lines were not approximately parallel throughout the binary image. After a week of experimenting with the above mentioned thresholds, I added yellow and white mask and immediately found a solution that works on the project video.

#### Combining yellow and white mask

I had used yellow and white masking in project 1. When I used that in this project, I immediately saw improvement in the video output (the code for this is in function `filter_color()` in the 4th cell). Carefully filtering out the yellow and white pixels removed the noise from the binary images. I combined the output of the thresholding described in previous section with yellow and white mask and the output is a clean binary image. An image showing the output of yellow & white masking function is shown below.
![alt text][image6]

#### Perspective transform

The code for my perspective transform includes a function called `warp()`, which appears on the 4th cell of my notebook.  The `warp()` function takes as inputs an image (`image`), as well as source (`src`) and destination (`dst`) points.  I hardcoded the source and destination points in the following manner:

```python
# source points
sx1, sy1, sx2, sy2, sx3, sy3, sx4, sy4 = 170, 720, 595, 450, 730, 450, 1150, 720
src = np.float32([[sx1, sy1],[sx2, sy2],[sx3, sy3],[sx4, sy4]]) 

# destination points
dx1, dy1, dx2, dy2, dx3, dy3, dx4, dy4 = 320, 720, 320, 0, 960, 0, 960, 720 
dst = np.float32([[dx1, dy1],[dx2, dy2],[dx3, dy3],[dx4, dy4]]) 
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 170, 720      | 320, 720        | 
| 595, 450      | 320, 0      |
| 730, 450     | 960, 0      |
| 1150, 720      | 960, 720        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image7]

#### Identifying lane pixels & fitting line

The function `find_lane_pixels()` is used to identify the lane pixels for the start of the video. It follows the steps outlined in the lectures. Once the lane lines are identified, the location of the line from the current frame can be used to identify the lane lines in the next frame. The implementation of this is in the function `polyfit_using_prev()`. If `polyfit_using_prev()` function cannot detect lanes for 20 consecutive frames, the search is reset and brute force search implemented in `find_lane_pixels()` is used to detect lane lines. Based on the non-zero pixels detected by these functions, a polynomial is fit using the function `fit_polynomial()`. An example of fitted lane lines on a binary image is shown below.

![alt text][image8]

This function returns the averaged polynomial coeffecients from the last 3 successfully detected lane line frames. 


#### Radius of curvature and offset from center

The radius of curvature is calculated in function `measure_curvature_real()` in the 20th cell of the notebook. The fitted polynomial coefficients from the pixel space are scaled appropriately to transform the radius into meters. The offset is calculated based on the center of the lane lines in the image and with the assumption that the camera is mounted at the center of the car. The offset calculation is done in `fit_polynomial()` function. 

#### Final output

I implemented this step in the function `mark_lane_lines()`.  The output on a test image is below.

![alt text][image9]

---

### Video Output

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

This project was much more difficult than I had anticipated. Whether it was the color channel/gradient thresholds or sanity checks adjustments, it never worked for all frames. Trying to kill the noise using thresholds in some frames also killed the lane lines in other frames. I tried atleast 20 different sanity checks (combination of radius of curvature or/and checking if lane lanes are parallel) thinking it will help in certain frames of the video, but the logic was flawed in some other frames of the video. In my opinion, this is because of the perspective transform. Irrespective of what source and destination points one chooses for the transform, the lane lines are never really parallel in the transformed image. Additionally, the thresholds here are optimized for the project video and need to be tweaked to make it work on the challenge videos. I tried this for a few days but gave up eventually because I want to move ahead in the course. The CNN approach seems much easier to generalize than the color spaces/gradient approach that require loads of experimentation to generalize. It would be good if the reviewer can throw some light on what is used in the industry (current working implementations) for this purpose -  CNN or traditional Computer Vision.
