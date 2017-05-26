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

[image1]: ./report/image1_undistort_output.PNG "Undistorted"
[image2]: ./report/image2_undistory_demo.PNG "Road Transformed"
[image3]: ./report/image3_threshold.PNG "Binary Example"
[image4]: ./report/image4_warped.PNG "Warp Example"
[image5]: ./report/image5_polynomial.PNG "Fit Visual"
[image6]: ./report/image6_lane.PNG.jpg "Output"
[video1]: ./report/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Code
All code referenced by this writeup is contained within the notebook, Project4-AdvancedLaneLines-Final.ipynb
located here
https://github.com/brightwellt/CarND-Advanced-Lane-Lines/blob/master/Project4-AdvancedLaneLines-Final.ipynb

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second code cell of the notebook 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding in the "apply_thresholds" function, and combining channels in "combine_channels" - both located in cell 1. Several cells also contain demonstrations of how these functions work.
Specifically I do
- a sobel threshold on the x gradient
- separate out H, L, S
- apply thresholds to L and S.
- combine the images into a binary image.
Here's an example of my output for this step. 

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in cell 6 of the notebook The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose to derive the source and destination points in the following manner:

```python
 src = np.float32(
        [[(img_size[0] / 2) - 50, img_size[1] / 2 + 90],
        [((img_size[0] / 6) - 20), img_size[1] -20],
        [(img_size[0] * 5 / 6) + 65, img_size[1] -20],
        [(img_size[0] / 2 + 50), img_size[1] / 2 + 90]])
    dst = np.float32(
        [[(img_size[0] / 4), 0],
        [(img_size[0] / 4), img_size[1]],
        [(img_size[0] * 3 / 4), img_size[1]],
        [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 590, 450      | 320, 0        | 
| 193, 700      | 320, 720      |
| 1131, 700     | 960, 720      |
| 690, 450      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

This cell also contains the vertices used to fit a region of interest to the warped image and remove extraneous data from the edges and middle. The region_of_interest function used to apply the vertices to the image is in cell 1.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Cell 7 contains the full_pipeline function used to take a raw image, and return an image with lane lines on it.

The second function in this Cell, "add_histo" does the following to fit lane lines with a 2nd order polynomial:
- takes a histogram of the lower half of the image
- finds the peaks of each half of the histogram to use as starting points for each line
- For each line, it creates a series of windows around the starting point, going bottom to top 
- 9 windows will be created vertically across the image for each line
- I'll search each window for nonzero pixels
- if we find enough pixels in the window, we'll recenter the next window on the mean of the non-zero pixels in this window
- I'll fit a 2nd order polynomial for each line based on the pixels we found.

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

White pixels show pixels excluded by the search. The red pixels are those identified for the left lane; the blue pixels for the right lane. The two polynomials are drawn on top in yellow. Note that when generating this image I briefly disabled the region of interest code - I wasn't getting left with any white pixels!

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The add_histo function scales the fitted polynomials and image dimensions from pixels to metres. The equation used and code this is based upon are taken from the "Measuring Curvature" section of this part of the course.

For vehicle position, we convert from pixels to m. I take the centre of the image as the midpoint of the vehicle. I determine how far this is from the edges of the lanes, and can work out how far from the centre of the lane we are. 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this at the end of the "add_histo" function.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](.report/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
