# 大唐道路自动驾驶视觉训练工程

本工程按文档中的算法路线整理为四个可训练模型。所有 YOLO 任务固定使用 **YOLO26**，代码会拒绝 `yolov5`、`yolo11` 等其他 YOLO 权重。

## 文档算法对应关系

- 驾驶员表情：使用公开 RAF-DB 7 类数据集镜像 + OpenCV 图像处理 + ImageNet 预训练 ResNet18 PyTorch CNN。
- 交通标识：使用文档的交通标志目标检测路线和 GTSRB 数据处理方式；文档列出过 YOLOv3，项目要求已覆盖为 **YOLO26**，代码拒绝其他 YOLO 版本。
- 车辆检测与跟踪：使用文档要求的目标检测 + DeepSORT；检测器固定为 **YOLO26**，DeepSORT 在推理阶段完成卡尔曼滤波、匹配和轨迹管理。
- 车道线：使用文档要求的 Swin Transformer 检测/分割路线；采用预训练 Swin Transformer V2 编码器加 PyTorch 分割解码器。最终训练使用质量过滤后的 TuSimple，RLMD/CARLA 仅作跨域诊断，不混入最终权重。

文档中的 YOLOv3/YOLOv5 示例、旧版依赖和课程环境命令不是本项目的运行约束；本项目以用户明确指定的 YOLO26 和当前可用 CUDA 环境为准。

| 模型目录 | 任务 | 算法 | 数据 |
|---|---|---|---|
| `training/driver_emotion` | 驾驶员表情分类 | OpenCV + PyTorch CNN | RAF-DB 七类表情 |
| `training/traffic_sign` | 交通标识检测 | YOLO26 detect | GTSRB 43 类，ROI 转 YOLO 框 |
| `training/vehicle_detection` | 车辆检测与跟踪 | YOLO26 detect + DeepSORT | 三类车辆框标注；类别契约仍需补充确认 |
| `training/lane_detection` | 车道线分割 | PyTorch Swin Transformer V2 | TuSimple + RLMD 车道/道路标线掩膜 |

`datasets/raw/vehicle_images_unlabeled` 中的 KITTI `image_2` 压缩包没有 `label_2`，只作为待补标注素材，不参加训练。

## Latest verified results

Results below are from the current checkpoints and independent holdout evaluations. They are not target values or training-set scores.

| Model | Checkpoint | Evaluation | Result |
|---|---|---|---:|
| Driver emotion | `training/driver_emotion/weights/best.pt` | RAF-DB-derived test accuracy | **86.91%** |
| Traffic sign | `training/traffic_sign/runs/yolo26_traffic_sign-2/weights/best.pt` | YOLO26 mAP50 | **97.86%** |
| Vehicle + DeepSORT detector | `training/vehicle_detection/runs/yolo26_vehicle/weights/best.pt` | YOLO26 mAP50 | **92.30%** |
| Lane segmentation | `training/lane_detection/weights/best.pt` | Complete unfiltered TuSimple validation, strict binary masks, mIoU | **71.03%** |

The emotion result remains below the requested 90%--95% acceptance target. The lane result is measured on the complete unfiltered TuSimple validation split. `datasets/processed/lane_tusimple_filtered` is used only for training; the selected checkpoint uses Swin V2 compact decoding with Lovasz IoU loss, stronger false-positive Tversky weighting, and threshold `0.20`. The previous 69.19% checkpoint remains `best_tusimple_precision.pt`.

## PySide6 实时检测界面

使用 Conda 的 `smart` 环境启动桌面界面。交付到其他电脑时，先使用项目根目录的 `environment.smart.yml` 创建目标用户自己的环境：

```powershell
.\app\run_smart.ps1
```

PyCharm 6 中把解释器设置为目标电脑实际创建的 Conda 环境，例如 `C:\Users\你的用户名\.conda\envs\road_smart\python.exe`，运行脚本设置为 `app/main.py`，工作目录设置为项目根目录。界面支持上传 MP4、AVI、MOV、MKV 视频，默认使用 CUDA，可切换 CPU，并同时显示车辆 DeepSORT 轨迹、交通标识框、驾驶员表情框、车道线分割和锚点。详细说明见 `app/README.md` 和 `docs/启动指南.md`。

实时界面的车辆模块使用项目根目录 `yolo26n.pt` 的 YOLO26 COCO 车辆类别（自行车、汽车、摩托车、公交车、卡车），再交给 DeepSORT 管理轨迹。训练目录中的旧车辆权重在用户道路截图上实测无输出，故不再作为界面默认权重；原始训练产物仍保留，便于后续补齐类别定义后重新训练。

## 环境

本机已确认单卡 NVIDIA GeForce RTX 4060 Laptop GPU，显存 8 GiB，PyTorch CUDA 可用。建议使用 Python 3.11/3.12 虚拟环境；当前 Python 3.13 也可先运行数据整理和 smoke test。

```powershell
python -m pip install -r requirements.txt
python tools/prepare_datasets.py --report-only
python tools/prepare_datasets.py --extract
python tools/prepare_emotion_rafdb.py --overwrite
```

## 训练

每个训练入口的时间预算可由参数调整；车道线默认配置允许长周期训练。实际精度以完整验证集结果为准，90%--95% 不是在未知数据分布上可以预先保证的指标。

```powershell
python training/driver_emotion/src/train.py
python training/traffic_sign/src/train.py
python training/vehicle_detection/src/train.py
python training/lane_detection/src/train.py
```

模型和指标分别写入对应的 `training/<model>/weights` 与 `training/<model>/runs`，四个模型的内部目录保持一致。

## 跟踪与推理

车辆检测模型训练完成后，把 YOLO26 checkpoint 传给 DeepSORT：

```powershell
python tools/run_deepsort.py --source input.mp4 --detector training/vehicle_detection/weights/best.pt --output training/vehicle_detection/runs/tracking.mp4
```

## 数据报告

数据对应关系、样本数、标注完整性和阻塞项保存在 `reports/dataset_manifest.md`。
