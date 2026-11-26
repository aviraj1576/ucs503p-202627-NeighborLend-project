# Week 4 : Why YOLOv8 for Item Identification (Proposal)

## Goal

For the proposal, choose the model family for identifying an item from the
lender's photo, and justify the choice.

## Relevant Context

Requirements for the model:

- it runs on the backend, on a CPU, for one photo at a time;
- it answers in about a second, while the lender is filling in the form;
- it can be trained on a small custom dataset (Week 3 classes).

## Key Observation

Two kinds of model could do this:

| | Image classifier | Object detector (YOLO) |
|---|---|---|
| output | one label for the whole photo | a box, a label and a confidence per object |
| photo with background clutter | the background affects the label | the item is located first |
| several items in one photo | cannot report them | reports each one |

A detector also tells *where* the item is, which matters for ordinary phone
photos taken on a table or the floor. YOLOv8 is a one-stage detector: it
predicts boxes and classes in a single pass, which keeps inference fast.

## Solution

- **Model**: YOLOv8 (Ultralytics), starting from weights pretrained on the
  COCO dataset, then fine-tuned on our classes (*transfer learning*).
- **Size**: the nano variant (`yolov8n`) first, because it is small and fast
  on a CPU; a larger variant only if accuracy is not enough.
- **Interface**: a backend endpoint receives the photo and returns
  `[{"label": ..., "confidence": ...}]`, sorted by confidence.
- **Evaluation**: precision, recall, mAP@50 and mAP@50–95 on a validation
  set (Week 7).

**Because**

Transfer learning reuses the general visual features learned from COCO, so a
small dataset is enough to learn the new classes. Some of our classes
(bicycle, umbrella, keyboard, book) also exist in COCO, which helps the model
start from useful features.
