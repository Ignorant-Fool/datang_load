# Dataset Manifest

Source: `F:\大唐道路\datasets\raw`

## Materialized

| Item | Count |
|---|---:|
| driver_emotion_images | 35887 |
| lane_pairs | 7252 |
| vehicle_image_label_pairs | 17856 |
| traffic_sign_roi_images | 26640 |

## Additional Open Labeled Sources Used

| Dataset directory | Task | Splits / count | License or provenance note |
|---|---|---:|---|
| `datasets/processed/driver_emotion_rafdb` | 7-class facial expression classification | train 14329, val 3071, test 3071 | RAF-DB-derived Hugging Face mirror; source terms must be reviewed before redistribution |
| `datasets/processed/lane_carla` | semantic road-line segmentation | train 6750, val 750, test 1500 | MIT; Town05 is an unseen-map test split |
| `datasets/processed/lane_tusimple_full` | paired lane-mask segmentation | train 6547, val 705 | Existing paired TuSimple archive; no separate lane test split in the materialized archive |
| `datasets/processed/lane_rlmd` | binary lane-line segmentation derived from 25-class road-marking masks | train 1737, val 400 | RLMD; MIT; line classes 4-10 and 16 retained, arrows/crosswalks/symbols excluded |
| `datasets/processed/lane_tusimple_rlmd` | mixed binary lane-line segmentation | train 8284, val 1105 | TuSimple + RLMD; used by the default Swin V2 checkpoint |

## Archives

| Archive | Files | Extensions |
|---|---:|---|
| `fer2013_images.zip` | 35887 | .jpg: 35887 |
| `tusimple_preprocessed.zip` | 14504 | .jpg: 14504 |
| `train_images.zip` | 16356 | .jpg: 16356 |
| `train_labels.zip` | 16356 | .txt: 16356 |
| `test_images.zip` | 1500 | .jpg: 1500 |
| `test_labels.zip` | 1500 | .txt: 1500 |
| `gtsrb_training_fixed.zip` | 26684 | .csv: 43, .ppm: 26640, .txt: 1 |
| `kitti_object_image_2.zip` | 14999 | .png: 14999 |

## Model Assignment

| Model | Task | Status | Algorithm |
|---|---|---|---|
| `pytorch_cnn` | driver_emotion_classification | **trainable** | OpenCV preprocessing + PyTorch CNN |
| `yolo26` | traffic_sign_detection | **trainable** | YOLO26 detection; GTSRB ROI converted to YOLO boxes |
| `yolo26_deepsort` | vehicle_detection_and_tracking | **trainable_and_inference_ready** | YOLO26 detection + DeepSORT tracking |
| `swin_v2` | lane_segmentation | **trainable** | PyTorch Swin Transformer V2 encoder + segmentation decoder |
| `none` | unlabeled_kitti_images | **blocked_missing_labels** | not used until label_2 is supplied |
