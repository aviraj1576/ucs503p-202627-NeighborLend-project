# Week 16 : A Larger Model, and Uploads That Were Not Using It

## Part 1 : Extending the Classes with a YOLOE Model

The nine-class model (Week 6) missed many items that people actually listed.
A new checkpoint, `neighborlend_yoloe_best.pt`, was trained on 24 classes:

| Group | Classes |
|---|---|
| Tools | electric_drill, screwdriver, wrench, hammer |
| Electronics | smartphone, laptop, camera, headphones, keyboard, drone |
| Outdoor and sports | bicycle, scooter, tent, umbrella, basketball, dumbbell, barbell, camping_chair |
| Home | microwave, kettle, blender |
| Travel and music | backpack, suitcase, guitar |

`dumbbell` is excluded from the predictions shown in the app
(`EXCLUDED_CLASSES`), so 23 classes are used. A `CLASS_MAP` turns model names
into display names, for example `electric_drill` → "Drill".

## Part 2 : Uploads Were Not Using the New Model

### Problem

After the new checkpoint was added, photo uploads during listing creation
still did not identify items with it.

### Relevant Context

The backend had two separate model loaders:

- `services/ai_identifier.py`, used by `/api/ai/identify`, loaded the YOLOE
  checkpoint and limited predictions to the allowed classes;
- `services/ml_service.py`, used by the **upload** route, had its own loader
  with the plain `YOLO` class, and read the model path from the settings,
  whose default still pointed at the old file:

``` python
# config.py
ai_model_path: str = "./ai_service/best_model.pt"
ai_confidence_threshold: float = 0.45
```

The startup preload also warmed this second loader, not the one used by
`/api/ai/identify`.

### Key Observation

Two code paths that should behave the same each had their own copy of the
model-loading logic, and only one copy had been updated. Testing
`/api/ai/identify` looked correct, while users going through the upload flow
got the old behaviour.

### Solution

Keep one loader. `ml_service` now delegates to `ai_identifier`:

``` python
from app.services.ai_identifier import identify_item

def detect_objects(image_bytes: bytes):
    return [{"class": p["label"], "confidence": round(p["confidence"], 4)}
            for p in identify_item(image_bytes)["predictions"]]
```

The default settings point at the trained checkpoint, with a threshold suited
to it:

``` python
ai_model_path: str = "./ai_service/models/neighborlend_yoloe_best.pt"
ai_confidence_threshold: float = 0.25
```

The same path change was made in `.env.example`, `compose.yaml` and
`render.yaml`. The tests in `tests/test_ai_model_connectivity.py` check that
uploads use the same class ids and filtering as identification, that startup
warms the shared instance, and that uploads still work when the model is
missing.

**Because**

Duplicated set-up code drifts apart. With one function owning the model, the
next model change has to be made in one place only.
