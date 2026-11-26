# Week 15 : Listing Photos Lost on Redeploy, so Move Them to Supabase Storage

## Problem

Listing photos did not display reliably on the deployed site. Photos saved
in the backend's local `uploads/` folder disappear whenever Render rebuilds
or restarts the container, and the backend already warned about this at
startup:

> Supabase Storage is not configured; production uploads will use ephemeral
> local storage.

## Relevant Context

`services/storage_service.py` uploads listing photos to a Supabase Storage
bucket. At runtime it tried to *create* the bucket, and if that failed, to
*update* it to public:

``` python
try:
    supabase.storage.create_bucket(bucket_name, options={"public": True})
except Exception:
    try:
        supabase.storage.update_bucket(bucket_name, options={"public": True})
    except Exception as exc:
        raise HTTPException(500, f"Could not configure Supabase bucket '{bucket_name}': {exc}")
```

## Key Observation

Creating or reconfiguring a bucket is an infrastructure change and should not
happen inside a user's upload request. When it failed, the user saw a generic
500 error that did not say what was wrong.

## Solution

1. Use one fixed bucket, `listing_photos`, and only **check** it. If it is
   missing or private, fail with a clear 503 message:

    ``` python
    bucket = supabase.storage.get_bucket(bucket_name)
    is_public = bucket.get("public") if isinstance(bucket, dict) else getattr(bucket, "public", None)
    if is_public is not True:
        raise HTTPException(503,
            f"Make the existing Supabase Storage bucket '{bucket_name}' public to display listing photos.")
    ```

2. In production, `create_listing` accepts only photo URLs from this bucket
   (HTTP 422 otherwise); local `/uploads/` URLs are allowed only in
   development.

3. Add `backend/scripts/migrate_listing_photos_to_supabase.py` to copy older
   `/uploads` photos into the bucket and update their database rows. It runs
   as a read-only dry run unless `--apply` is passed, and production also
   needs `--confirm-production-write`.

**Because**

The bucket is now set up once, by a person, in the Supabase dashboard; the
application only checks that setup and reports exactly what to fix. Making
the migration script a dry run by default means running it by mistake
changes nothing.
