# Week 8 : Improvement Plan: Nearby Search with the Haversine Formula

## Problem

The proposal planned PostgreSQL with the PostGIS extension for
location-based search. For the improvement plan, the search had to work on
every database the project uses, including SQLite for local development and
tests, where PostGIS is not available.

## Relevant Context

Each listing stores `latitude` and `longitude` as plain numbers. The search
endpoint receives the user's position and a radius:
`GET /api/listings/search?lat=..&lon=..&radius_km=..`.

## Key Observation

At the scale of a student project (hundreds of listings, not millions),
computing the distance to each published listing in Python is fast enough,
and it needs no database extension. The great-circle distance between two
points on a sphere is given by the haversine formula.

## Solution

``` python
def haversine(lat1, lon1, lat2, lon2):
    R = 6371  # Earth radius in km
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = (math.sin(dlat / 2) ** 2
         + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2))
         * math.sin(dlon / 2) ** 2)
    return R * 2 * math.asin(math.sqrt(a))
```

The search first filters in SQL (published, category, name `ILIKE`), then
computes distances, keeps those inside the radius, and sorts by distance. A
radius of 500 km or more means "no limit"; listings without coordinates are
then shown at the end.

If the browser does not share a location, `GET /api/listings/detect-location`
estimates it from the client's IP address (reading the first address in
`X-Forwarded-For` behind Render's proxy), and falls back to a default hub,
Patiala (30.3362, 76.3922).

**Because**

Filtering in SQL first keeps the Python loop short. PostGIS remains the path
for growth: a spatial index would avoid computing the distance to every
listing once there are many of them.
