# Week 17 : Running the Detector with ONNX Runtime on a Small Server

## Problem

The deployed backend has 512 MB of memory and no GPU. Loading PyTorch and
Ultralytics only to run one model left too little memory for the rest of the
application (see Vansh's Week 17 entry).

## Relevant Context

A YOLO model can be exported to ONNX, a portable model format, and run with
ONNX Runtime, which is much smaller than PyTorch. The backend exports the
checkpoint once if no `.onnx` file exists:

``` python
loaded.export(format="onnx", imgsz=640, dynamic=False, simplify=False)
```

## Key Observation

Without Ultralytics, the work it normally does around the model has to be
written by hand, and it must match the training exactly:

1. **pre-processing**: resize with the same *letterbox* padding used in
   training;
2. **decoding**: the raw output is a `(1, 4 + C, 8400)` tensor, i.e. 8400
   candidate boxes, each with a box `(cx, cy, w, h)` and one score per class;
3. **non-maximum suppression (NMS)**: remove overlapping boxes for the same
   object.

## Solution

**Letterbox to 640 × 640**, keeping the aspect ratio and padding with grey
(114), then scale to 0–1 and reorder to channels-first:

``` python
scale = 640 / max(w, h)
nw, nh = int(w * scale), int(h * scale)
img_resized = img.resize((nw, nh), Image.Resampling.BILINEAR)
new_img = Image.new("RGB", (640, 640), (114, 114, 114))
new_img.paste(img_resized, ((640 - nw) // 2, (640 - nh) // 2))
arr = np.array(new_img, dtype=np.float32).transpose(2, 0, 1)[None] / 255.0
```

**Decode** by taking the best class per box and dropping boxes below the
confidence threshold (0.25). The number of class columns tells which class
list applies: 9 for the first model, 24 for the YOLOE model.

``` python
out = session.run(None, {input_name: arr})[0][0].T     # (8400, 4 + C)
boxes, scores = out[:, :4], out[:, 4:]
class_name_dict = NINE_CLASS_NAMES if scores.shape[1] == 9 else YOLOE_NAMES
max_scores, best_classes = scores.max(axis=1), scores.argmax(axis=1)
mask = max_scores >= conf_thresh
```

**NMS in NumPy** (IoU threshold 0.45): keep the highest-scoring box, drop the
boxes that overlap it by more than the threshold, and repeat. Predictions of
excluded classes are removed, names are mapped to display names, and the
result is sorted by confidence.

**Because**

The model only gives correct results on inputs prepared exactly like its
training images, so the letterbox size and padding must match the export
(640, grey 114). The ONNX session is created once and reused, and a smaller
12 MB ONNX model is the one served in production (the YOLOE export is about
99 MB), which keeps memory inside the server's limit.
