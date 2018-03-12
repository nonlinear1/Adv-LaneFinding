## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./camera_cal/calibration1.jpg "Distorted"
[image2]: ./camera_cal_op_images/calibration1.jpg "Undistorted"
[image3]: ./test_images/test1.jpg "Road Distorted"
[image4]: ./undist_images/test1.jpg "Road Undistorted"
[image5]: ./undist_images/test5.jpg "Road Undistorted Thresholding input"
[image6]: ./thresh_images/test5.jpg "Road Thresholded Thresholding output"
[image7]: ./roi_images/test5.jpg "Road ROIed output"
[image8]: ./persp_images/test5.jpg "Road Perpestive Transformed"
[image9]: ./window_images/test5.jpg "Lane Lines Detected"
[image10]: ./frame_op/frame_1.jpg "Final Lane"
[video1]: ./test_videos_output/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the function CalibCamera present in AdvLaneFinding.ipynb.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

The total number of corners to be ideally detected for the set of chessboard images provided for calibration was 9 in the Y-direction and 6 in the X-direction. However on executing the Camera Calibration's function findChessboardCorners on the entire set of images, it failed to return the ideal set of corners for a few of the images, which were distorted so badly that only a subset of the total 54 corners were actually visible in the image.

In order to be able to calibrate the camera and generate an accurate value of distortion coefficients as well as Camera matrix, I adapted the code logic to be dynamic in terms of the number of corners it would try to detect in each chessboard input image. The code would start looking for 9 corners in the Y-direction and 6 in the X-direction ideally and incase it would fail, the next step would be try looking for 9 corners in the Y-direction and 5 in the X-direction. This detection approach would continue till atleast 3 corners in the X-direction can be detected, after which the number of corners in the Y-direction would be decreased to 8 and the algo would search for 8 corners in the Y-direction and 6 in the X-direction. This decremental search allows me to ensure that all the provided camera calibration images are utilized in generating the distortion coefficients as well as Camera Calibration matrix.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![Distorted Image][image1]
![Undistorted Image][image2]

All the other undistorted images output for the chessboard images are present in camera_cal_op_images folder.

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![Distorted][image3]
![Undistorted][image4]

All the other undistoted versions of the test-images are present in the undist_images folder.


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I played around with all the thresholding methods like Gradient in Xdirection, Ydirection, Magnitude of Gradient based thresholding, Direction of gradient thresholding as well as Colour-thresholding methods.
The colour-thresholding methods that I tried out were:
1) Saturation channel based thresgolding of the HSV colour space image
2) B channel based thresholding of the LAB colour space image
3) Hue channel based thresholding of the HSV colour space image for deriving the yellow and white colour masks.
4) V channel based thresholding of the HSV colour space image.
5) L channel based thresholding of the HSL colour space image.

Finally I only retained the thresholding methods:
a)  Graident-X & Graident-Y Orred with
b)  Saturation channel & V channel

The Gradient-X thresholding method is contained in the function abs_sobel_thresh with oreint='x' input
The Graident-Y thresholding method is contained in the function abs_sobel_thresh with oreint='y' input
The Saturation channel thresholding method is contained in the function col_thresh and col_thresh_assist with channel='s' input
The V-Channel thresholding method is contained in the function col_thresh and col_thresh_assist with channel='v' input

The combination of these thresholding methods are covered in the function Thresholding().
Here's an example of my output for this step.

![Thresholding input][image5]
![Thresholding output][image6]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is present seperately for operating on the sample test-images as well as in the Pipeline for processing the video frames.
For both the Pipeline video processing as well as the Sample test-images, the thresholded image is ROI extracted and then fed into the Perspective Transform.

For ROI masking, I took the following region, which is the same as that for Project-1 for Lane finding:
(40,720); (640,405); (675,425); (1280,720)

Further the ROI image was subjected to Perpective Transform using the folowing src and destination points:

```python
 w,h = 1280,720
    x,y = 0.5*w, 0.8*h
    src = np.float32([[200./1280*w,720./720*h],
                      [453./1280*w,547./720*h],
                      [835./1280*w,547./720*h],
                      [1100./1280*w,720./720*h]])
    dst = np.float32([[(w-x)/2.,h],
                  [(w-x)/2.,0.82*h],
                  [(w+x)/2.,0.82*h],
                  [(w+x)/2.,h]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 200,720       | 320,720       | 
| 453,547       | 320,590       |
| 835,547       | 960,590       |
| 1100,720      | 960, 720      |

An example of a thresholded image, its corresponding ROI image and its perspective Transformed image is as follows:

![Thresholding output][image6]
![ROI output][image7]
![Perspective Transform output][image8]

All the other image ouputs for the ROI images as well as the Perspective Transformed images are present in the folders roi_images as well as persp_images respectively.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

In order to identify the Lane pixels for the Left and Right Lanes, I utilized the Sliding Window approach explained in the lessons where for one of the frames, sliding windows would be formed for both the lanes and all the X and Y coordinates lying in all these detected windows would be utilized for finding the Polynomial coefficients. Once the polynomial coefficients were determined, the corresponding X coordinates of the points lying on the fitted polynomial are determined to result is us obtaining:
a) Poly Coeffs for the polynomial fit for the Left and Right Lanes
b) X and Y coordinates of the points lying on the Poly fitted line for both the lanes.
This is implemented in the function LanefitFirst and is referred to as Blind Window search throughout the notebook.

Also once the Sliding Window approach has been carried out for the first frame, the subsequent frame does not perform a sliding window detection, but rather assumes that the windows in the current frame would be an offset around the windows found in the first frame. This has been used from the code present in the Lessons and is implemented in the function ExtrapolatedLaneFit.

Also a function SanityCheck has been defined which checks if the X-coordinates determined for the Left and the Right lanes(Averge calculated lane width) are within a threshold of 0.3m from the actual lane width of 3.7m.

The algorithm for determining the lane-line pixels and fitting their positions is structured as follows:
a) Perform a Blind Window Search to find the Poly coeffs as well as the Xfits for the Left and Right Lanes.
b) Append and save these detected poly coeffs to a class instance's member by calling append_fits and set the detected member to True.
c) For the case of a frame that follows a frame for which Blind Window search was carried out, find the poly coeffs as well as the Xfits for the left and right lanes by extrapolation.
d) Perform a Sanity check on such frames for which X-coords and poly coeffs were obtained by extrapolation.
e) Incase Sanity check passes for them, Append and save these detected poly coeffs to a class instance's member by calling append_fits.
f) If Sanity check fails for them, utilize the Averaged X-coordinates as well as poly coeffs(Data of last two Sanity Check passed or Blind window search frames is stored in a weighted manner) from class storage and increment the frame-reuse count.
g) If the Frame-reuse count exceeds a threshold of 5, i.e. last 5 frames have utilized averaged poly coeffs as well as Xfits, then reset the is_detected flags to ensure that a blind window search is carried out for the next Image frame of the video.

An example of the Sliding Window output on one of the test-images is as follows:

![Lane Line detected][image9]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The Radius of curvature is determined for both the the Left and Right lanes as well as the position of the vehicle from the center is calculated in the function Curv_Dist_Cent.
It utilizes the apprach presented in the Lessons for converting the Xfit coeffs and the Y coordinates to Real-world space of meteres from pixels, finding the polyfitted coeffs for the transformed coordinates and then using the radius of curvature formula for finding the radius at the y coordinate of 720(i.e. bottom of the image).

The Distance of the vehicle with respect to the center is done by finding the X-coodinate of the Left lane at Y=720 and the X-coorinate of the Right LAne at Y=720. Finding the midpoint of their X-coorinates and then finding the difference of the midpoint from the x-coordinate of the center of the image.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The overlaying of the Radius of Curvature and Distance is performed in the function Curv_Rad_Text.
The overlaying of the Lane segments encompassed between the detected left and right lanes is implemented in the function Pipline()
Here is an example of my result on a test image:

![Final lane output][image10]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./test_videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The main issue that I faced during the implementation of this project was deciding on the Thresholding methods to utilize as well as the min and max thresholds for each of them. This required significant experimentation as well as fine tuning of thresholds to arrive at the final choice that performs reasonably well.
Also another choice was regarding the max number of X-fits as well as poly coeffs that should be stored in the class Line()'s member to re-use for future frames if needed. I eventually selected retaining data pertaining to 2 frames only in a weighted manner.
Yet another choice was regarding the number of frames that could utilize averaged X-fit and averaged poly coeffs from storage, before reverting back to the Blind Window search for the next frame. I eventually selected a frame re-use factor of 5 as it looked to be performing well based on dumped statistics for the video.
The pipeline fails on highly curvy roads as well as roads where the colour is not uniform as seen in the challenge videos and the Thresholding technique has to be more robust for the entire pipeline to perform well.
As a further improvement, Convolution based approach for Lane pixel detection could be implemented as well as other image thresholding methods could be tried out.
Also another approach can be implemented as a potential improvement where the Undistored image is ROI filtered, Perspective transformed and then subjected to Thresholding to reduce the dependency on the exact choice and order of Gradient Thresholding that needs to be incorporated.
