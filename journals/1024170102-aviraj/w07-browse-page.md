# Week 7 : The Browse Page for the Prototype (Search, Category, Distance)

## Goal

For the prototype evaluation, show that a borrower can find items near them:
search by name, filter by category, and limit the distance with a slider.

## Relevant Context

The backend search endpoint takes all filters as query parameters:

``` text
GET /api/listings/search?q=drill&category=Tools&lat=30.34&lon=76.39&radius_km=50
```

It computes the distance of each listing from `(lat, lon)` and drops those
outside `radius_km`. A radius of 500 km or more means "no distance limit".

## Key Observation

The page has four inputs (text, category, radius, location) that all change
the same result list. Building the query in one function, and calling it
whenever any filter changes, keeps the list and the filters in sync.

## Solution

``` jsx
const [radius, setRadius] = useState(50);

const fetchListings = async () => {
  const params = new URLSearchParams();
  if (searchQuery) params.append('q', searchQuery);
  if (category !== 'All') params.append('category', category);
  params.append('lat', location.lat);
  params.append('lon', location.lon);
  // The last slider position (100) means "all distances".
  params.append('radius_km', radius < 100 ? radius : 9999);
  const res = await api.get(`/listings/search?${params.toString()}`);
  setListings(res.data || []);
};

useEffect(() => { fetchListings(); }, [category, radius, location]);
```

- The heading says what is shown: "Listings within 50 km of …", or "all
  distances" at the end of the slider.
- The location comes from the browser's geolocation API. If the user blocks
  it, the page falls back to an approximate location from the IP address, and
  then to a default hub (Patiala).
- Each result is a `ListingCard` with the photo, the daily price and the
  distance.

**Because**

Sending the maximum value as "no limit" (9999 km, above the backend's 500 km
cut-off) avoids a separate "all distances" flag in the API. Keeping the
fallbacks explicit means the page always shows *some* listings, even when
location access is denied. (A bug in this flow, where an older search could
overwrite a newer one, was found and fixed in Week 17.)
