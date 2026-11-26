# Week 6 : Training the First Detection Model

## Goal

Fine-tune YOLOv8 on the nine-class dataset (Week 5) and produce a checkpoint
the backend can load.

## Relevant Context

Training starts from the pretrained `yolov8n.pt` weights. Images are resized
to 640 × 640 during training, which is also the input size used later for
inference.

## Key Observation

With a small dataset, the model can memorise the training images
(over-fitting). The validation metrics, not the training loss, show whether
it is still improving. Ultralytics keeps two checkpoints: `last.pt` (the
final epoch) and `best.pt` (the epoch with the best validation score), and
`best.pt` is the one to deploy.

## Solution

``` python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")                 # COCO-pretrained, transfer learning
model.train(data="data.yaml", imgsz=640)   # augmentation is on by default
metrics = model.val()                      # precision, recall, mAP on the val set
```

Points checked after training:

- the class names stored in the checkpoint match `data.yaml`;
- predictions on photos that were **not** in the dataset (taken with our
  own phones) look sensible;
- the checkpoint is small enough to ship inside the backend Docker image
  (the trained nine-class checkpoint is under 6 MB).

The checkpoint is stored in the backend as `ai_service/best_model.pt`.

**Because**

Ultralytics applies data augmentation (mosaic, scaling, colour changes,
flips) by default, which partly makes up for a small dataset. Testing on our
own phone photos is closer to real use than the validation set, because
lenders take photos in ordinary rooms and light.
