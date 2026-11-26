# Week 5 : Building and Labelling the Dataset in YOLO Format

## Goal

Prepare a labelled dataset for the nine classes, in the format the
Ultralytics trainer expects.

## Relevant Context

YOLO training needs, for every image, a text file with one line per object:

``` text
<class_id> <x_center> <y_center> <width> <height>
```

All four box values are **normalised** to 0–1 by the image width and
height, so the labels stay valid if the image is resized.

## Key Observation

Most labelling mistakes are silent: the trainer still runs, but learns the
wrong thing. Three checks catch most of them:

1. every image has a label file with the same name, and vice versa;
2. every class id is between 0 and 8, and every box value is in [0, 1];
3. near-duplicate images (the same photo resized or cropped) must not end
   up in both the training and the validation set, or the validation score
   will be too optimistic.

## Solution

Folder layout and the dataset file:

``` text
dataset/
  images/train/  images/val/
  labels/train/  labels/val/
```

``` yaml
# data.yaml
path: dataset
train: images/train
val: images/val
names:
  0: bicycle
  1: book
  2: drill
  3: hammer
  4: headphones
  5: ladder
  6: keyboard
  7: pliers
  8: umbrella
```

A short script checks the label files before every training run:

``` python
for label in Path("dataset/labels").rglob("*.txt"):
    image = label.parent.parent.parent / "images" / label.parent.name / (label.stem + ".jpg")
    assert image.exists(), f"no image for {label}"
    for line in label.read_text().splitlines():
        cls, *box = line.split()
        assert 0 <= int(cls) <= 8, f"bad class in {label}"
        assert all(0.0 <= float(v) <= 1.0 for v in box), f"box outside image in {label}"
```

The split into training and validation is made per *source photo*, so that
variants of one photo stay on the same side.

**Because**

The class order in `data.yaml` becomes part of the model: class id 2 means
"drill" only because of this file. The backend must use exactly the same
order when it turns predicted ids back into names (Week 17).
