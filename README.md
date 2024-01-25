# Real-time Multi-Object Tracking for Rescue Operations

## Problem Statement

## Abstract

## Table of Contents
1. [Object Detection](#od)
2. [Object Tracking](#ot)
3. [The Hungarian Algorithm](#ha)
4. [SORT: Simple Online Realtime Tracking](#s)
5. [Deep SORT: Simple Online and Realtime Tracking with a Deep Association Metric](#ds)

------------
<a name="od"></a>
## 1. Object Detection

------------
<a name="ot"></a>
## 2. Object Tracking
In my other project - [**Optical Flow Obstacle Avoidance for UAV**](https://github.com/yudhisteer/Optical-Flow-Obstacle-Avoidance-for-UAV) - we saw how we can analyze the **apparent motion** of objects in a sequence of images using ```Optical Flow```. We tested with **sparse**, **dense**, and **deep optical flow** to eventually see how every point in the scene is moving from frame to frame in a video sequence. In object tracking, we want to track **objects** or **regions** from frame to frame. 

### 2.1 Change Detection
Change detection refers to a way to detect meaningful changes in a sequence of frames. Suppose we have a static camera filming the environment as shown in the video below, the meaningful changes will be the moving objects - me running. In other words, we will be doing a real-time **classification** (```foreground-background classification```) of each pixel as belonging to the **foreground** (```meaningful change```) or belonging to a **background** (```static```).


To build this classification, we will need to get rid of unmeaningful changes or uninterested changes which are:

- Background fluctuations such as leaves moving due to wind.
- Image noise due to low light intensity.
- Natural turbulence such as snow or rain.
- Shadows of people moving.
- Camera shake such as in the video below.

<div style="text-align: center;">
  <video src="https://github.com/yudhisteer/Real-time-Ego-Tracking-A-Tactical-Solution-for-Rescue-Operations/assets/59663734/38a4e14a-6e03-4ead-adee-7a9a13bb01a5" controls="controls" style="max-width: 730px;">
  </video>
</div>

There are a few ways to detect meaningful changes:
1. We calculate the difference between the current frame and the previous one. Wherever this difference is significantly higher than a set threshold, we consider it a change. However, we may experience a lot of noise and register uninterested changes such as background fluctuations.

2. A more robust method is to compute a background image, i.e., the **average** of the first ```K``` frames of the scene. We then compare our pixel value (subsequent frames) with the average background image. However, we are storing only one value for pixel as the model and we comparing all the future values with that value and may not handle the movement of leaves or change in lighting.

3. A better method than the average of the first ```K``` frames would be to instead calculate the **median** of the first K frames. It is more stable than the average, but a value in any given frame can vary substantially from the most popular value.

4. An even more robust model is to build an **adaptive** one such that the median is computed using the last few frames and not the first few frames. That is, we are going to recompute the median of the background model and it is going to adapt to changes. We can experience substantial improvement, especially for background fluctuations.


### 2.2 Template Matching
Now that we can spot important changes in videos, we choose one of these changes to represent an object or area we want to focus on. Our goal is to follow and track this object throughout the entire video sequence. We can do so by using a region of interest (ROI) as a template and applying template matching from frame to frame.

#### 2.2.1 Appearance-based Tracking
One approach involves using the entire image region to create **templates** and then applying **template matching**. This process involves creating a template from the initial image, using it to locate the object in the next frame, and then updating the template for further template matching in subsequent frames.

<p align="center">
  <img src="https://github.com/yudhisteer/Real-time-Ego-Tracking-A-Tactical-Solution-for-Rescue-Operations/assets/59663734/a4294f57-94ea-4d66-8863-3dcf4830b165" width="70%" />
</p>

In the example above, we take the grey car in frame ```t-1``` in the red window as a template.  We then apply that template within a search window (green) in the next frame, ```t```. Wherever we find a good match (blue), we declare it as the new position of the object. The condition is that the **change in appearance** of the object between time ```t-1``` and ```t``` is **very small**. However, this method does not handle well large changes in **scale**, **viewpoint**, or **occlusion**.


#### 2.2.2 Histogram-based Tracking

In histogram-based tracking, rather than using the entire image region, we compute a **histogram** - 1-dimensional (grayscale image) or high-dimensional histogram (RGB image). This histogram serves as a **template**, and the tracking process involves **matching** these histograms between images to effectively track the object.

<p align="center">
  <img src="https://github.com/yudhisteer/Real-time-Ego-Tracking-A-Tactical-Solution-for-Rescue-Operations/assets/59663734/d40b0246-bbe9-4191-adbf-16554e7adf93" width="90%" />
</p>

We want to track an object within a region of interest (ROI). However, the reliability of points in the ROI decreases towards the **edges** due to potential **background interference**. To address this, a **weighted histogram**, like the ```Epanechnikov kernel```, is used. This weights pixel contributions differently based on their distance from the center of the window. The weighted histograms are then employed for **matching** between frames, similar to **template matching**. This method, relying on histogram matching, proves more **resilient** to changes in **object pose**, **scale**, **illumination**, and **occlusion** compared to appearance-based template matching.

### 2.3 Tracking-by-Detection

#### 2.3.1 Matching Features using SIFT

1. Given a frame at time ```t-1```, we can either use an object detection algorithm to detect an object and draw a bounding box around it, or we can manually draw a bounding box around an object of interest. 
2. We then compute **SIFT** of similar features for the frame. Note that SIFT has the location and also the descriptor of the features.
3. We classify the features within our bounding box as an **object** and declare them to belong to set ```O```.
4. We then classify the remaining features (outside the bounding box) as **background** and declare them to belong to set ```B```.
5. For the next frame ```t```, we again calculate SIFT **features** and **descriptors**.
6. For each feature and descriptor in frame ```t```, we calculate the distance, ```d_O```, between the current feature and the best matching feature in the object model, ```O```.
7. For each feature and descriptor in frame ```t```, we calculate the distance, ```d_B```, between the current feature and the best matching feature in the background model, ```B```.
8. If ```d_O``` is much smaller than ```d_B``` then we give a confidence value of ```+1``` that the feature belongs to the object else we give a confidence value of ```-1``` that it does not belong to the object.
9. We then take the bounding box in the previous frame ```t-1``` and place it in the current frame ```t```.
10. We will **distort** this window that has changed its position and shape to grab as many object features as possible.
11. We want to find the window for which we have the **largest** number of **object features** inside and a small number of background features such that it becomes the **new position** of the object.
12. Recall that the object may have changed in appearance slightly so we're going to then take the features inside to update the object model, ```O```, and the features outside to update the background model, ```B```.
134. We repeat the process for all the remaining frames and track the object of interest.


#### 2.3.2 Similarity Learning using Siamese Network


<p align="center">
  <img src="https://github.com/yudhisteer/Real-time-Ego-Tracking-A-Tactical-Solution-for-Rescue-Operations/assets/59663734/cd85c8a9-1bf8-4a8f-a695-a1495eecfe63" width="90%" />
</p>




### 2.4 Cost Function

#### 2.4.1 Intersection over Union (IoU)




#### 2.4.2 Sanchez-Matilla



#### 2.4.3 Yu



### 2.4 Metric

#### 2.5.1 MOTA

#### 2.5.2 IDF1

---------------
<a name="ha"></a>
## 3. The Hungarian Algorithm
The Hungarian algorithm(Kuhn-Munkres algorithm) is a **combinatorial optimization algorithm** used for solving ```assignment problems```. In the context of object tracking, it is employed to find the optimal association between multiple **tracks** and **detections**, optimizing the **cost** of assignments based on metrics such as **Intersection over Union (IoU)**. But why do we need the Hungarian algorithm? Why don't we choose the highest IOU? Here's why:

- It can deal with cases where not all tracks are associated with detections or vice versa.
- It can handle scenarios with multiple tracks and detections, ensuring coherent and consistent assignments. Suppose for one object we have three IOUs: ```0.29```, ```0.30```, and ```0.31```. If we had to choose the highest IOU we would choose ```0.31``` but this also means that we selected this IOU over the lowest one (```0.29```) with only a difference of ```0.02```. Selecting the highest IOU would be a naive approach.
- It considers all possible associations simultaneously, optimizing the overall assignment of tracks to detections.

Now let's take an example of three bounding boxes as shown below. The **black** ones are the **tracks** at time ```t-1``` and the **red** ones are the **detections** at time ```t```. From the image, we can already see which track will associate with which detection. Note that this is a simple scenario where we have no two or more detections for one track.

<p align="center">
  <img src="https://github.com/yudhisteer/Multi-Object-Tracking-with-Deep-SORT/assets/59663734/00b43dbd-7929-4ec2-9023-09b6a4e47e45" width="70%" />
</p>

The next step will be to calculate the IOU for each combination of detection and track and put them in a matrix as shown below. Again, we can already see a pattern of association emerging for the detection and track from the value of IOU only.

<p align="center">
  <img src="https://github.com/yudhisteer/Multi-Object-Tracking-with-Deep-SORT/assets/59663734/95b5f22f-13bc-48b1-9fe7-ac7db1090f85" width="50%" />
</p>

Below is the step-by-step process of the Hungarian algorithm. We won't need to code it from scratch but use a function from ```scipy```.

<p align="center">
  <img src="https://github.com/yudhisteer/Multi-Object-Tracking-with-Deep-SORT/assets/59663734/38d83258-89c1-424a-ad84-8ec151d62090" width="50%" />
</p>

For our scenario since our metric is IOU, meaning the highest IOU equal to the highest overlap between two bounding boxes, it is a **maximization** problem. Hence, we introduce a **minus** sign in the IOU matrix when putting it as a parameter in the ```linear_sum_assignment``` function.

```python
row_ind, col_ind = linear_sum_assignment(-iou_matrix)
```
We then select an IOU **threshold** ```(0.4```), to filter matches and unmatched items for determining associations between detections and trackings. This threshold allows us to control the level of overlap required for a match. From the results, we may have three possible scenarios:

- **Matches**: Associations between detected objects at time ```t``` and existing tracks from time ```t-1```, indicating the continuity of tracking from one frame to the next.

- **Unmatched Detections**: Detected objects at time ```t``` that do not have corresponding matches with existing tracks from time ```t-1```. These represent newly detected objects or objects for which tracking continuity couldn't be established.

- **Unmatched Trackings**: Existing tracks from time ```t-1``` that do not find corresponding matches with detected objects at time ```t```. This may occur when a tracked object is not detected in the current frame or is incorrectly associated with other objects.

```python
matches = [(old[i], new[j]) for i, j in zip(row_ind, col_ind) if iou_matrix[i, j] >= 0.4]
unmatched_detections = [(new[j]) for i, j in zip(row_ind, col_ind) if iou_matrix[i, j] < 0.4]
unmatched_trackings = [(old[i]) for i, j in zip(row_ind, col_ind) if iou_matrix[i, j] < 0.4]
```
The output:

```python
Matches: [([100, 80, 150, 180], [110, 120, 150, 180]), ([250, 160, 300, 220], [250, 180, 300, 240])]
Unmatched Detections: [[350, 160, 400, 220]]
Unmatched Trackings: [[400, 80, 450, 140]]
```

------------
<a name="s"></a>
## 4. SORT: Simple Online Realtime Tracking
The authors of the SORT paper offer a lean approach for MOT for online and real-time applications. They argue that in the tracking-by-detection method, detection holds a key factor whereby the latter can increase the tracking accuracy by ```18.9%```. SORT focuses on a frame-to-frame prediction using the **Kalman Filter** and association using the **Hungarian algorithm**. Their method achieves speed and accuracy comparable to, at that time, SOTA online trackers. Below is my implementation of the SORT algorithm. Though it is not the same as the official SORT GitHub repo, my approach offers a simpler method with not-so-bad accuracy. Most of the explanations below have been extracted from the [SORT](https://arxiv.org/abs/1602.00763) paper itself and rewritten by me.

### 4.1 Detection
The authors of the SORT paper use a Faster Region CNN - FrRCNN as their object detector. In my implementation, I will use the [YOLOv8m](https://github.com/ultralytics/ultralytics) model. I have created a **yolo_detection** function which takes in as parameters an image, the YOLO model, and the label classes we want to detect.

```python
    # 1. Run YOLO Object Detection to get new detections
    _, new_detections_bbox = yolo_detection(image_copy, model, label_class={'car', 'truck', 'person'})
```

### 4.2 Estimation Model
In the SORT algorithm, a ```first-order four-dimensional (4D) Kalman filter``` is employed for object tracking. Each **tracked object** is represented by a ```4D state vector```, incorporating **position** coordinates and **velocities**. The Kalman filter is initialized upon detection, setting its initial state based on the bounding box. In each frame, the filter **predicts** the object's **next state**, updating the bounding box accordingly. When a detection aligns with a track, the Kalman filter is **updated** using the observed information. In my project [UAV Drone: Object Tracking using Kalman Filter](https://github.com/yudhisteer/UAV-Drone-Object-Tracking-using-Kalman-Filter), I explain more about the Kalman Filter in detail.

```python
def KalmanFilter4D(R_std: int = 10, Q_std: float = 0.01):

    # Create a Kalman filter with 8 state variables and 4 measurement variables
    kf = KalmanFilter(dim_x=8, dim_z=4)

    # State transition matrix F
    kf.F = np.array([[1, 1, 0, 0, 0, 0, 0, 0],
                     [0, 1, 0, 0, 0, 0, 0, 0],
                     [0, 0, 1, 1, 0, 0, 0, 0],
                     [0, 0, 0, 1, 0, 0, 0, 0],
                     [0, 0, 0, 0, 1, 1, 0, 0],
                     [0, 0, 0, 0, 0, 1, 0, 0],
                     [0, 0, 0, 0, 0, 0, 1, 1],
                     [0, 0, 0, 0, 0, 0, 0, 1]])

    # Initialize covariance matrix P
    kf.P *= 1000

    # Measurement noise covariance matrix R
    kf.R[2:, 2:] *= R_std

    # Process noise covariance matrix Q
    kf.Q[-1, -1] *= Q_std
    kf.Q[4:, 4:] *= Q_std

    return kf
```

```python
def state_transition_matrix(dt: float):
    # Define the state transition matrix F based on the time step (dt)
    return np.array([
        [1, dt, 0, 0, 0, 0, 0, 0],
        [0, 1, 0, 0, 0, 0, 0, 0],
        [0, 0, 1, dt, 0, 0, 0, 0],
        [0, 0, 0, 1, 0, 0, 0, 0],
        [0, 0, 0, 0, 1, dt, 0, 0],
        [0, 0, 0, 0, 0, 1, 0, 0],
        [0, 0, 0, 0, 0, 0, 1, dt],
        [0, 0, 0, 0, 0, 0, 0, 1]])
```

### 4.3 Data Association
When associating detections with existing targets, the algorithm estimates each target's bounding box by predicting its position in the current frame. The assignment cost matrix is computed using the IOU distance, measuring the overlap between detections and predicted bounding boxes. The **Hungarian** algorithm optimally solves the **assignment problem**, with a minimum IOU threshold rejecting assignments with insufficient overlap. Below I wrote an **association** function that computes the Hungarian and returns the indices of match detections, bounding boxes of unmatched detections, and bounding boxes of unmatched trackers as explained in the Hungarian section.

```python
    # 4. Associate new detections bbox (detections) and old obstacles bbox (tracks)
    match_indices, matches, unmatched_detections, unmatched_trackers = association(tracks=old_obstacles_bbox,
                                                                                   detections=new_detections_bbox,
                                                                                  metric_function=metric_total)
```

In this code snippet, for each pair of matched indices representing existing targets and new detections, the algorithm retrieves the ID, bounding box, and age of the old obstacle. It increments the age and creates a new obstacle instance with the corresponding information, including the current time. The Kalman filter predicts the next state. Subsequently, the Kalman filter of the obstacle is then updated with the measurement (bounding box) from the new detection. The obstacle's time is updated, and its bounding box is adjusted according to the Kalman filter's predicted values. Finally, the newly updated obstacle is appended to the list of new obstacles.

```python
    # 5. Matches: Creating new obstacles based on match indices
    for index in match_indices:
        # Get ID of old obstacles
        id = old_obstacles[index[0]].id
        # Get bounding box of new detections
        detection_bbox = new_detections_bbox[index[1]]
        # Get age of old obstacles and increment by 1
        age = old_obstacles[index[0]].age + 1
        # Create an obstacle based on id of old obstacle and bounding box of new detection
        obstacle = ObstacleSORT(id=id, bbox=detection_bbox, age=age, time=current_time)
        # PREDICTION
        F = state_transition_matrix(current_time - obstacle.time)
        obstacle.kf.F = F
        obstacle.kf.predict()
        obstacle.time = current_time
        obstacle.bbox = [int(obstacle.kf.x[0]), int(obstacle.kf.x[2]), int(obstacle.kf.x[4]), int(obstacle.kf.x[6])]
        # UPDATE
        measurement = new_detections_bbox[index[1]]
        obstacle.kf.update(np.array(measurement))
        # Append obstacle to new obstacles list
        new_obstacles.append(obstacle)
```


### 4.4 Creation and Deletion of Track Identities

In the code below, for each unmatched detection, a new obstacle is created with a unique ID, using the bounding box coordinates of the unmatched detection and the current time. This ensures that each new detection, not associated with any existing target, is assigned a distinct identifier. The newly created obstacle is then added to the list of new obstacles, and the ID counter is incremented to maintain uniqueness for the next unmatched detection.

```python
    # 6. New (Unmatched) Detections: Give the new detections an id and register their bounding box coordinates
    for unmatched_detection_bbox in unmatched_detections:
        # Create new obstacle with the unmatched detections bounding box
        obstacle = ObstacleSORT(id=id, bbox=unmatched_detection_bbox, time=current_time)
        # Append obstacle to new obstacles list
        new_obstacles.append(obstacle)
        # Update id
        id += 1
```

Here, the unmatched trackers, which represent existing targets that were not successfully matched with any detection in the current frame, are processed. For each unmatched tracker, the corresponding obstacle is retrieved from the list of old obstacles based on the bounding box. The Kalman filter associated with the obstacle is then updated by predicting its state using the state transition matrix and the time difference since the last update. The unmatch age of the obstacle is incremented, indicating how many frames it has remained unmatched. The obstacle's bounding box is also updated, and it is added to the list of new obstacles to continue tracking.

```python
    # 7. Unmatched tracking: Update unmatch age of tracks in unmatch trackers
    for tracks in unmatched_trackers:
        # Get index of bounding box tracks in old obstacles that match with unmatched trackers
        index = old_obstacles_bbox.index(tracks)
        # If we have a match
        if index is not None:
            # Based on index get the obstacle from old obstacles list
            obstacle = old_obstacles[index]
            # PREDICTION
            F = state_transition_matrix(current_time - obstacle.time)
            obstacle.kf.F = F
            obstacle.kf.predict()
            obstacle.time = current_time
            obstacle.bbox = [int(obstacle.kf.x[0]), int(obstacle.kf.x[2]), int(obstacle.kf.x[4]), int(obstacle.kf.x[6])]
            # Increment unmatch age of obstacle
            obstacle.unmatch_age += 1
            # Append obstacle to new obstacles list
            new_obstacles.append(obstacle)
```

Tracks are terminated after not being detected for a duration defined by "MAX_UNMATCHED_AGE". It avoids issues where predictions continue for a long time without being corrected by the detector. The author argues that "MAX_UNMATCHED_AGE" is set to ```1``` in experiments for two reasons: the constant velocity model is a poor model of the true dynamics, and secondly, the focus is on frame-to-frame tracking rather than **re-identification**. 

```python
    # Draw bounding boxes of new obstacles with their corresponding id
    for _, obstacle in enumerate(new_obstacles):
        # Remove false negative: Filter out obstacles that have not been detected for a long time ("MAX_UNMATCHED_AGE")
        if obstacle.unmatch_age > MAX_UNMATCHED_AGE:
            new_obstacles.remove(obstacle)

        # Remove false positive: Display detections only when appeared "MIN_HIT_STREAK" times
        if obstacle.age >= MIN_HIT_STREAK:
            x1, y1, x2, y2 = obstacle.bbox
            color = get_rgb_from_id(obstacle.id*20)
            cv2.rectangle(image_copy, (x1, y1), (x2, y2), color, thickness=cv2.FILLED)
```


--------------
<a name="ds"></a>
## 4. Deep SORT: Simple Online and Realtime Tracking with a Deep Association Metric

----------------

## References
1. https://arshren.medium.com/hungarian-algorithm-6cde8c4065a3
2. https://www.thinkautonomous.ai/blog/hungarian-algorithm/
3. https://medium.com/augmented-startups/deepsort-deep-learning-applied-to-object-tracking-924f59f99104
4. https://www.youtube.com/watch?v=QtAYgtBnhws&ab_channel=DynamicVisionandLearningGroup
5. https://www.youtube.com/watch?app=desktop&v=ezSx8OyBZVc&ab_channel=ShokoufehMirzaei
6. https://brilliant.org/wiki/hungarian-matching/
7. https://www.youtube.com/watch?v=BLRSIwal7Go&list=PL2zRqk16wsdoYzrWStffqBAoUY8XdvatV&index=12&ab_channel=FirstPrinciplesofComputerVision
