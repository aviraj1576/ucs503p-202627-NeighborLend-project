# Week 17 : Two Concurrency Bugs Found on the Live Deployment

Both bugs appeared only when several requests were in flight at once, so
neither showed up when testing one action at a time on a laptop.

## Part 1 : An Older Search Response Overwrites a Newer One

### Error

On the Browse page, "all distances" sometimes showed only 5 listings after
the browser blocked location access, and a keyword search sometimes showed
unfiltered results.

### Key Observation

Every filter change calls `fetchListings()` (Week 7), and responses can
arrive in a different order than the requests were sent. A slow earlier
search, such as the initial 50 km search, could finish **after** a newer one
and replace its results.

### Solution

Number each search with a counter kept in a `useRef`, and let only the newest
one update the state:

``` jsx
const latestSearch = useRef(0);

const fetchListings = async () => {
  const searchId = latestSearch.current + 1;
  latestSearch.current = searchId;
  setLoading(true);
  try {
    const res = await api.get(`/listings/search?${params.toString()}`);
    if (searchId !== latestSearch.current) return;   // a newer search exists
    setListings(res.data || []);
  } catch (err) {
    if (searchId !== latestSearch.current) return;
    setListings([]);
  } finally {
    if (searchId === latestSearch.current) setLoading(false);
  }
};
```

**Because**

A `useRef` value survives re-renders without causing one, so every pending
call compares against the same counter. An `AbortController` would also work
and would cancel the network request as well.

## Part 2 : Damage-Check Photos Fail with HTTP 500 (Connection Pool Starvation)

### Error

The damage-check page loads several condition photos at once. Some photo
requests hung for about 15 seconds and then returned HTTP 500.

### Relevant Context

The database is Supabase PostgreSQL in session-pooling mode, which allows 15
client sessions in total, so the SQLAlchemy pool had been kept very small:

``` python
return {"pool_size": 2, "max_overflow": 0, "pool_timeout": 15}
```

Each photo request authenticates the user through `get_current_user`, which
uses the request's database session. The damage-check module then opens its
own short session to read the photo.

### Key Observation

Each photo request needed **two** connections: the request session, held
until the request ended, and one more for the photo. With a pool of two and
two parallel requests, each held one connection and waited for another.
Neither could finish, so both waited for `pool_timeout` (15 s) and failed.

### Solution

1. Return the request's connection as soon as authentication is done:

    ``` python
    def damage_user_id(user: User = Depends(get_current_user),
                       db: Session = Depends(get_db)) -> str:
        user_id = str(user.id)
        db.close()   # the module opens its own short sessions
        return user_id
    ```

2. Size the pool for the real deployment. Render runs a single backend
   instance, so it can use more of Supabase's 15 sessions:

    ``` python
    return {"pool_size": 5, "max_overflow": 3, "pool_timeout": 15}
    ```

**Because**

A pool deadlocks when every request needs *k* connections at once and the
pool has fewer than *k* × (concurrent requests). Releasing the first
connection early brings *k* down to one at a time, and the larger pool gives
headroom for several photos loading in parallel.
