## <a name="top"></a> Vehicle Detection Project [![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](https://www.udacity.com/course/self-driving-car-engineer-nanodegree--nd013)

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector. 
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[image1]: ./output_images/car_not_car.png
[image2]: ./output_images/HOG_example.png
[image21]: ./output_images/car_normilized_hist.png
[image22]: ./output_images/3d_color_hist.png
[image3]: ./output_images/sliding_windows.png
[image4]: ./output_images/sliding_window.png
[image6]: ./output_images/labels_map.png
[image7]: ./output_images/output_bboxes.png
[video1]: ./project_video_output.mp4

## [Rubric Points](https://review.udacity.com/#!/rubrics/513/view)
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.

The code for this step is contained in the first code cell of the IPython notebook `def get_hog_features`.  

I started by reading in all the `vehicle` and `non-vehicle` images.  Here is an example of one of each of the `vehicle` and `non-vehicle` classes:
```python
def import_images():
    cars = glob.glob('./data/vehicles/**/*.png')
    notcars = glob.glob('./data/non-vehicles/**/*.png')
    num_of_car_images = len(cars)
    num_of_notcar_images = len(cars)

    print('Number of Car images = {}'.format(num_of_car_images))
    print('Number of Not-Car images = {}'.format(num_of_notcar_images))
    return cars, notcars
```
The result is 
```
Number of Car images = 8792
Number of Not-Car images = 8792
```
![alt text][image1]

I then explored different color spaces and different `skimage.hog()` parameters (`orientations`, `pixels_per_cell`, and `cells_per_block`).  I grabbed random images from each of the two classes and displayed them to get a feel for what the `skimage.hog()` output looks like.

Here is an example using the `YCrCb` color space and HOG parameters of `orientations=9`, `pixels_per_cell=(8, 8)` and `cells_per_block=(2, 2)`:
```python
def get_hog_features(img, orient, pix_per_cell, cell_per_block, vis=False, feature_vec=True):
    if vis == True:
        features, hog_image = hog(img, orientations=orient, pixels_per_cell=(pix_per_cell, pix_per_cell),
                                  cells_per_block=(cell_per_block, cell_per_block), transform_sqrt=False, 
                                  visualise=True, feature_vector=False)
        return features, hog_image
    else:      
        features = hog(img, orientations=orient, pixels_per_cell=(pix_per_cell, pix_per_cell),
                       cells_per_block=(cell_per_block, cell_per_block), transform_sqrt=False, 
                       visualise=False, feature_vector=feature_vec)
        return features
```
![alt text][image2]

To reduce a number of features I lookes at features generated by HOG and by extracting `color_hist` by lookiing at different color channels. In order to figure out how it works I used 3D representation of color on the images.
```python
#This function calls bin_spatial() and color_hist()
def extract_features(imgs, cspace='RGB', spatial_size=(32, 32),
                        hist_bins=32, hist_range=(0, 256)):
    #Create a list to append feature vectors to
    features = []
    #Iterate through the list of images
    for file in imgs:
        # Read in each one by one
        image = mpimg.imread(file)
        #apply color conversion if other than 'RGB'
        if cspace != 'RGB':
            if cspace == 'HSV':
                feature_image = cv2.cvtColor(image, cv2.COLOR_RGB2HSV)
            elif cspace == 'LUV':
                feature_image = cv2.cvtColor(image, cv2.COLOR_RGB2LUV)
            elif cspace == 'HLS':
                feature_image = cv2.cvtColor(image, cv2.COLOR_RGB2HLS)
            elif cspace == 'YUV':
                feature_image = cv2.cvtColor(image, cv2.COLOR_RGB2YUV)
        else: feature_image = np.copy(image)      
         #Apply bin_spatial() to get spatial color features
        spatial_features = bin_spatial(feature_image, size=spatial_size)
        #Apply color_hist() also with a color space option now
        hist_features = color_hist(feature_image, nbins=hist_bins, bins_range=hist_range)
        #Append the new feature vector to the features list
        features.append(np.concatenate((spatial_features, hist_features)))
    #Return list of feature vectors
    return features
```
![alt text][image21]

For 3D representation of an image I used this function:
```python
from mpl_toolkits.mplot3d import Axes3D

def plot3d(pixels, colors_rgb, axis_labels=list("RGB"), axis_limits=[(0, 255), (0, 255), (0, 255)]):
    """Plot pixels in 3D."""

    #Create figure and 3D axes
    fig = plt.figure(figsize=(8, 8))
    ax = Axes3D(fig)

    #Set axis limits
    ax.set_xlim(*axis_limits[0])
    ax.set_ylim(*axis_limits[1])
    ax.set_zlim(*axis_limits[2])

    #Set axis labels and sizes
    ax.tick_params(axis='both', which='major', labelsize=14, pad=8)
    ax.set_xlabel(axis_labels[0], fontsize=16, labelpad=16)
    ax.set_ylabel(axis_labels[1], fontsize=16, labelpad=16)
    ax.set_zlabel(axis_labels[2], fontsize=16, labelpad=16)

    #Plot pixel values with colors given in colors_rgb
    ax.scatter(
        pixels[:, :, 0].ravel(),
        pixels[:, :, 1].ravel(),
        pixels[:, :, 2].ravel(),
        c=colors_rgb.reshape((-1, 3)), edgecolors='none')

    return ax  # return Axes3D object for further manipulation
```
![alt text][image22]

#### 2. Explain how you settled on your final choice of HOG parameters.

I was playing with different combinations. From the Lesson I've learned to use `orientations=8`, `pixels_per_cell=(8, 8)` and `cells_per_block=(2, 2)`. After doint testing on project I have ended up using `orientations=9`, `pixels_per_cell=(8, 8)` and `cells_per_block=(2, 2)`.

#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

I trained a linear SVM using `LinearSVC()`.
```python
    #Split up data into randomized training and test sets
    rand_state = np.random.randint(0, 100)
    X_train, X_test, y_train, y_test = train_test_split(
        scaled_X, y, test_size=0.2, random_state=rand_state)

    print('Using:',orient,'orientations',pix_per_cell,
        'pixels per cell and', cell_per_block,'cells per block')
    print('Feature vector length:', len(X_train[0]))
    #Use a linear SVC
    svc = LinearSVC()
```
As input params to train the model I used 
```python
        color_space='YCrCb', orient=9, pix_per_cell=8, cell_per_block=2,
        hog_channel='ALL', spatial_size=(16, 16), hist_bins=32, spatial_feat=True,
        hist_feat=True, hog_feat=True
```
The output of training of the model looks like this:
```python
Extracting Features...
120.99 Seconds to extract features
('Using:', 9, 'orientations', 8, 'pixels per cell and', 2, 'cells per block')
('Feature vector length:', 6156)
(7.0, 'Seconds to train SVC...')
Test Accuracy of SVC = 0.9921
('SVC predicts: ', array([ 0.,  0.,  0.,  1.,  0.,  1.,  1.,  1.,  0.,  0.]))
('For these', 10, 'labels: ', array([ 0.,  0.,  0.,  1.,  0.,  1.,  1.,  1.,  0.,  0.]))
(0.0017, 'Seconds to predict', 10, 'labels with SVC')
```
### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

To search for windows I used `slide_window` and `search_windows` function.
```python
    x_start_stop = [None, None]
    xy_overlap = (0.75, 0.75)

    y_start_stops = [[400, 645], [400, 600], [400, 550]]
    xy_windows = [(128, 128), (96, 96), (64, 64)]

    windows = []

    for y_start_stop, xy_window in zip(y_start_stops, xy_windows):
        windows.extend(slide_window(image, x_start_stop, y_start_stop, xy_window, xy_overlap))

    hot_windows = search_windows(image, windows, svc, X_scaler,
                        color_space='YCrCb', spatial_size=spatial_size, hist_bins=hist_bins,
                        orient=orient, pix_per_cell=pix_per_cell,
                        cell_per_block=cell_per_block, hog_channel=hog_channel,
                        spatial_feat=spatial_feat, hist_feat=hist_feat, hog_feat=hog_feat)
```

![alt text][image3]

#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

Ultimately I searched on two scales using YCrCb 3-channel HOG features plus spatially binned color and histograms of color in the feature vector, which provided a nice result.  Here are some example images:

![alt text][image4]
---

### Video Implementation

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)
Here's a [link to my video result](https://youtu.be/Queh_WHV6bo) 

#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.

I recorded the positions of positive detections in each frame of the video.  From the positive detections I created a heatmap and then thresholded that map to identify vehicle positions.  I then used `scipy.ndimage.measurements.label()` to identify individual blobs in the heatmap.  I then assumed each blob corresponded to a vehicle.  I constructed bounding boxes to cover the area of each blob detected.  

Here's an example result showing the heatmap from a series of frames of video, the result of `scipy.ndimage.measurements.label()` and the bounding boxes then overlaid on the last frame of video and link to my test video [link to my test video](https://youtu.be/43Doj__RnR8):

### Here is the output of `scipy.ndimage.measurements.label()` on the integrated heatmap from all six frames:
![alt text][image6]

### Here the resulting bounding boxes are drawn onto the last frame in the series:
![alt text][image7]

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?
It takes a lot of time to detect vehicle prosecc project video locally. I have read about [YOLO](https://pjreddie.com/darknet/yolo/), it was too close to the end of the project soft deadline so I descided t try it later.

Probelms I have faced were mainly with detection accuracy. I was important to balance clisifier accuracy and execution speed. Some vehicles can change possion faster than another, this aspect brings more inacccuracy and misclassification. In the frame were black car was passing in fron of the white car that algorythm detected overlapping tow vehicles as one object and draw one wide rectangular, even there were two cars in the frame.

Producing a very high accuracy classifier and maximizing window overlap might improve the per-frame accuracy to the point that integrating detections from previous frames is unnecessary.

The pipeline is probably most likely to fail in cases where vehicles or the HOG features as they don't resemble those in the training dataset, but lighting and environmental conditions might also play a role (like a white car against a white background, gray car on the asphalt road).

I believe that the best approach, given plenty of time to pursue it, would be to combine a very high accuracy classifier with high overlap in the search windows. The execution cost could be offset with more intelligent tracking strategies, such as:

* determine vehicle location and speed to predict its location in subsequent frames
* begin with expected vehicle locations and largest scale search areas, and include overlap and redundant detections from smaller scale search areas to speed up execution
* use a convolutional neural network, to do the sliding window search 

