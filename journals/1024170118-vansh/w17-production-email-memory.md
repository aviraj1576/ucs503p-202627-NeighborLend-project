# Week 17 : Two Problems that Appeared Only in Production

## Part 1 : Verification Emails Not Delivered from Render

### Error

Registration sends a 6-digit OTP by email (Week 11). It worked locally, but
on the deployed backend the email never arrived: sending timed out.

### Relevant Context

`services/email.py` sent the OTP through Brevo's SMTP relay with `smtplib`,
on port 587 (STARTTLS).

### Key Observation

Cloud hosts such as Render block outbound connections on the SMTP ports 25,
465 and 587 to prevent spam. The code was correct; the network path was
closed. HTTPS on port 443 is not blocked, and Brevo offers a REST API for
transactional email.

### Solution

Send through Brevo's HTTPS API first, and keep SMTP as a fallback:

``` python
def _send_via_brevo_api(recipient: str, code: str, settings) -> bool:
    payload = {
        "sender": {"name": "NeighborLend", "email": settings.smtp_from_email},
        "to": [{"email": recipient}],
        "subject": "Your NeighborLend verification code",
        "textContent": f"Your NeighborLend verification code is {code}. ...",
    }
    req = urllib.request.Request(
        "https://api.brevo.com/v3/smtp/email",
        data=json.dumps(payload).encode("utf-8"), method="POST",
        headers={"api-key": api_key, "Content-Type": "application/json"},
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        return resp.status in (200, 201, 202)
```

The function returns `False` instead of raising on HTTP errors and timeouts,
so the caller can fall back to SMTP. Tests in `tests/test_email_service.py`
cover these paths. Registration also stays usable when no email channel
works.

**Because**

When a feature works locally but times out in the cloud, check the network
path before the code. Using the standard library's `urllib` avoided adding a
dependency to the Docker image.

## Part 2 : Keeping AI Inference Inside Render's Memory Limit

### Problem

The backend runs on Render's free instance with 512 MB of memory. Loading
PyTorch together with the detection model used most of it, and photo uploads
that ran item identification were failing on the deployed backend.

### Relevant Context

`services/ai_identifier.py` imported PyTorch and Ultralytics at import time,
so every process paid for them even though production only runs the ONNX
model through ONNX Runtime (Tikshan's Week 17 entry).

### Key Observation

PyTorch is needed only for a fallback path, so importing it at startup wasted
memory. Two more details mattered:

- ONNX Runtime's CPU memory arena keeps freed buffers for reuse, which raises
  peak memory;
- a call to glibc's `malloc_trim` through `ctypes`, meant to return memory to
  the OS, was itself causing crashes.

### Solution

1. **Lazy imports**: only check that Ultralytics is installed; import `torch`
   and the model class inside `_get_model()` when the fallback is used:

    ``` python
    YOLO_AVAILABLE = importlib.util.find_spec("ultralytics") is not None
    ```

2. **No PyTorch fallback in production**: without an ONNX session, return
   "model unavailable" instead of loading the large PyTorch model.

3. **A smaller ONNX footprint**:

    ``` python
    opts = ort.SessionOptions()
    opts.enable_cpu_mem_arena = False
    opts.intra_op_num_threads = 1
    opts.inter_op_num_threads = 1
    ```

4. **No native calls**: remove `malloc_trim`, and replace OpenCV's
   `NMSBoxes` with a short NumPy implementation of non-maximum suppression.

`/api/ai/status` now reports the process's resident memory
(`memory_rss_mb`). After these changes, peak memory stayed at about 238 MB.

**Because**

In a memory-limited container, what is *imported* matters as much as what is
*used*. Reporting memory through a status endpoint makes the next problem of
this kind a measurement rather than a guess.
