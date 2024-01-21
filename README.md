# Real-time Multi-Object Tracking: A Tactical Solution for Rescue Operations

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
Change detection refers to a way to detect meaningful changes in a sequence of frames. Suppose we have a static camera filming the environment as shown in the video below, the meaningful changes will be the moving objects - me running. So the the real problem, In other words, we will be doing a real-time **classification** (```foreground-background classification```) of each pixel as belonging to the **foreground** (```meaningful change```) or belonging to a **background** (```static```).


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

#### 2.3.2 Similarity Learning using Siamese Network


<p align="center">
  <img src="https://github.com/yudhisteer/Real-time-Ego-Tracking-A-Tactical-Solution-for-Rescue-Operations/assets/59663734/cd85c8a9-1bf8-4a8f-a695-a1495eecfe63" width="90%" />
</p>




### 2.4 Cost Function

#### 2.4.1 Intersection over Union (IoU)

```python
def metric_IOU(bbox1: Tuple[int, int, int, int], bbox2: Tuple[int, int, int, int]) -> float:
    # Get max and min coordinates
    x1 = max(int(bbox1[0]), int(bbox2[0]))
    y1 = max(bbox1[1], bbox2[1])
    x2 = min(bbox1[2], bbox2[2])
    y2 = min(bbox1[3], bbox2[3])

    # Calculate the distance between coordinates
    width = x2 - x1
    height = y2 - y1

    # Condition to see if there is overlap
    if width > 0 and height > 0:
        # Calculate overlap area
        overlap = width * height

        # Calculate union area
        area_bbox1 = (bbox1[2] - bbox1[0]) * (bbox1[3] - bbox1[1])
        area_bbox2 = (bbox2[2] - bbox2[0]) * (bbox2[3] - bbox2[1])
        union_area = area_bbox1 + area_bbox2 - overlap

        # Calculate IOU
        iou = round(overlap / union_area, 5)
        return iou

    # No overlap
    iou = 0
    return iou
```


#### 2.4.2 Sanchez-Matilla

```python
def metric_sanchez_matilla(bbox1: Tuple[int, int, int, int], bbox2: Tuple[int, int, int, int]) -> float:
    
    # Get width and height of image
    width = 1920
    height = 1080

    # Area of image
    Q_shp = width * height

    # Diagonal of image
    Q_dist = np.sqrt(width ** 2 + height ** 2)
    # Get W,H and (X,Y)
    X_A = (bbox1[2] + bbox1[0]) / 2
    X_B = (bbox2[2] + bbox2[0]) / 2
    Y_A = (bbox1[3] + bbox1[1]) / 2
    Y_B = (bbox2[3] + bbox2[1]) / 2
    H_A = bbox1[3] - bbox1[1]
    H_B = bbox2[3] - bbox2[1]
    W_A = bbox1[2] - bbox1[0]
    W_B = bbox2[2] - bbox2[0]

    # Distance term
    c_dist = np.sqrt((X_A - X_B) ** 2 + (Y_A - Y_B) ** 2) / Q_dist

    # Shape term
    c_shp = np.sqrt((H_A - H_B) ** 2 + (W_A - W_B) ** 2) / Q_shp

    # Cost
    sm_cost = c_dist * c_shp

    return sm_cost

```

#### 2.4.3 Yu

```python
def metric_yu(bbox1: Tuple[int, int, int, int],
              bbox2: Tuple[int, int, int, int],
              old_features: np.ndarray,
              new_features: np.ndarray) -> float:

    # Get W,H and (X,Y)
    X_A = (bbox1[2] + bbox1[0]) / 2
    X_B = (bbox2[2] + bbox2[0]) / 2
    Y_A = (bbox1[3] + bbox1[1]) / 2
    Y_B = (bbox2[3] + bbox2[1]) / 2
    H_A = bbox1[3] - bbox1[1]
    H_B = bbox2[3] - bbox2[1]
    W_A = bbox1[2] - bbox1[0]
    W_B = bbox2[2] - bbox2[0]

    # Weights
    w1 = 0.5
    w2 = 1.5

    # Distance term
    c_dist = np.exp(-w1 * (((X_A - X_B) / W_A) ** 2 + ((Y_A - Y_B) / H_A) ** 2))

    # Shape term
    c_shp = np.exp(-w2 * ((np.abs(H_A - H_B)) / (H_A + H_B) + (np.abs(W_A - W_B)) / (W_A + W_B)))

    # Feature cost
    c_AB = cosine_similarity(old_features, new_features)[0][0]

    # Total cost
    yu_cost = c_AB * c_dist * c_shp

    return yu_cost
```

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
