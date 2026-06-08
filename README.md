## Aerial Vehicle Detection
The task is divided into 2 main parts:
1. Auto labeling data
2. Training student's model to Detect Vehicles

All tasks are performed in a single Python notebook: `aerial_vehicle_detection.ipynb`.

The notebook can be opened and reviewed together with my comments.

The trained model weights are stored at:
`./runs/detect/runs/vehicle_sahi640/weights/best.pt`

The resulting visualizations are saved in:
`out_train_viz`. Due to large files, all videos uploaded here : (See here)[!https://drive.google.com/drive/folders/1PAHuhBpzpNyz41mdzvgXJzJA1VL7DVzP?usp=sharing)]

## Auto Labeling Data
For Auto Labeling, I decided to use YOLO 26X model and use [2, 3, 5, 7] classes that represents:
car, motorcycle, bus, and truck that used as single class vehicle in our application.
As high-quality labeled data is a main key to efficient and precise model,
I use the biggest version of the YOLO model (x - means Extra).

I also tried to use previous version of YOLO model 11X, but from my investigation YOLO 26X do better labellig (less flickers).
Additionnaly, I tried to use both YOLO 26X-OBB and YOLO 11X-OBB models, that is optimized for detecting rotated objects in aerial and satellite imagery.
But my investigation shows that regular YOLO 26X do better work for smaller object. Possibly, that I need to play with OBB more.

As original YOLO models trained for image size 640 (while OBB for 1024 pixels) pixels, 
but our videos are high qualities, we cannot directly pass the frame into the model, cause it will be resized to smaller sizes,
that makes images loose improtant detailes. As our focus on detecting small objects, we cannot use it.

During investigation I found that on provided videos sometimes big buildings classified as vehicles. To excldue them I added parameter for each video called `max_vehicle_px` that 
comapres biggest size of the BBOX to it and if object bigger, then it excluded. It helped imrpove training by removing such wrong data.

To work with small object detections, I apply SAHI processing (Slicing Aided Hyper Inference), where we split image into overlapping slcies
and process it's parts as it is (without resizing), detecting all objects and merge detection back after processing all tiles.

This helps us to process image without resizing and loosing some information.

## Training Vehicle Detector
For training, I decided to use smallest YOLO 26N (nano) model, as it is most lightweight model and it developed for real-time processing on edge devices.
Using our labeled data we slices it for image size of 640x640 to tran our model additionnaly using on-a-fly augmentation techniques, such as mosaic, rotation, etc.

Trained model than evaluated and generate video visualizations. You can find them in the folder out_train_viz and see this eval example:
[Watch the video](./out_train_viz/eval/13722965_2160_3840_30fps_gt_vs_pred.mp4)

## Evaluation metrics
### Testing (Evaluation) part:
#### EVAL (held-out) :: ALL VIDEOS (aggregate)

| Metric                       | 0-200  | 200-400 |
|-----------------------------|-------:|--------:|
| Detection rate TP/(TP+FN)   | 0.328  | 0.123   |
| Precision TP/(TP+FP)        | 0.213  | 0.955   |
| False alarms / min          | 3124.63 | 79.47   |
| Time to first detection (s) | 1.65   | 0.00    |

**mAP@0.5 across both bands:** 0.1602  
**Videos:** 2  
**Frames:** 2784

**Counts:**  
- `0-200`: TP=1309, FP=4836, FN=2682  
- `200-400`: TP=2609, FP=123, FN=18669

### Training Part:

#### TRAIN :: ALL VIDEOS (aggregate)

| Metric                       | 0-200    | 200-400 |
|-----------------------------|---------:|--------:|
| Detection rate TP/(TP+FN)   | 0.884    | 0.870   |
| Precision TP/(TP+FP)        | 0.588    | 0.609   |
| False alarms / min          | 10568.50 | 4708.43 |
| Time to first detection (s) | 0.17     | 0.37    |

**mAP@0.5 across both bands:** 0.7763  
**Videos:** 4  
**Frames:** 2465

**Counts:**  
- `0-200`: TP=23056, FP=16179, FN=3015  
- `200-400`: TP=11225, FP=7208, FN=1674

## Notes:
Interesting, that added second eval clip `12897527_1920_1080_30fps.mp4` works similarly as training video `8968356-hd_1920_1080_30fps.mp4`. At the begging of `12897527_1920_1080_30fps.mp4`, 
when cars are placed orthogonally, detector works only on closest part of the road, but drone change perspective and cars start moving less vertically, detector works better. Same for 
Auto-labelling YOLO 26X model for `8968356-hd_1920_1080_30fps.mp4`, only cars moving not vertically detected efficiently. It is probably should be fixed at labeling stage and then trained model will perform better.

## Possible improvements
1. Estimating object sizes and spliting them into categories can be done using NN GeoCalib, that estiamtes camera intrinsics and camera relative possition + height.
2. Data labelling should have some additional steps to remove non-vehicle objects detected as vehicle. I found miss-classification of building and road-signs as cars.
To deal with it, we can additionnaly detect building and road signs and if such ojects BBOX overlaped with vehciles - exclude such vehicles from dataset.
3. For better training, we need deeply understand our test environments and add same augmentations (camera poses, lightgling conditions, objects sizes, etc.)
4. As we work with video, we should include some temporal information between frames to probably exclude some flicking objects (not related to detector task)
