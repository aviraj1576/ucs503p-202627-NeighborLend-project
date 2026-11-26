# Week 15 : Deploying the Backend on Render (Docker Build and Slow First Request)

For the second prototype, the backend was deployed on Render as a Docker
container. Two problems came up.

## Part 1 : OpenCV System Library Missing from the Docker Image

### Error

The image stopped building at the `apt-get install` step: the package
`libgl1-mesa-glx` could not be installed.

### Relevant Context

The item-detection code imports OpenCV (`cv2`), which loads the system
library `libGL.so.1`. The `Dockerfile` installs it with the other graphics
libraries:

``` dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    libxcb1 \
    libglib2.0-0 \
    libgl1-mesa-glx \
    libsm6
```

### Key Observation

The `python:3.11-slim` tag is not pinned to a Debian release. It now follows
Debian 13 (trixie), which no longer ships `libgl1-mesa-glx`. That name was
only a transitional package; the library itself is in `libgl1`.

### Solution

Install `libgl1` instead of `libgl1-mesa-glx`.

**Because**

A tag such as `3.11-slim` follows the latest Debian release, so package names
can change under an unchanged `Dockerfile`. Pinning the release (for example
`python:3.11-slim-bookworm`) makes such changes deliberate.

## Part 2 : Slow First AI Request, so Preload the Model at Startup

### Problem

The first photo upload after the server started took much longer than later
ones, because the detection model was loaded inside that first request.

### Key Observation

Loading the weights and running the first inference (which initialises the
runtime) are one-off costs. They belong at application startup, not in a
user's request.

### Solution

Move the startup work into a FastAPI `lifespan` handler, and load and warm
the model there with a dummy image:

``` python
# simplified from main.py
def _preload_model():
    try:
        model = YOLO(str(get_settings().ai_model_path_abs))
        model(np.zeros((640, 640, 3), dtype=np.uint8), verbose=False)  # warm-up
    except Exception as exc:
        logging.getLogger(__name__).warning(f"Model pre-load skipped: {exc}")

@asynccontextmanager
async def lifespan(app: FastAPI):
    Base.metadata.create_all(bind=engine)
    os.makedirs(settings.upload_dir_abs, exist_ok=True)
    _preload_model()
    yield

app = FastAPI(title="NeighborLend API", version="1.0.0", lifespan=lifespan)
```

**Because**

A failed preload is only logged, so the API still starts if the weights are
missing. The cost moves to startup, which matters less than a slow first
request on a free server that sleeps when idle.
