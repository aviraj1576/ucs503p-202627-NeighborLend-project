# Week 5 : Backend Skeleton: Settings, Database Sessions and Routers

## Goal

Set up the FastAPI project so that every later feature follows the same
pattern for configuration, database access and routing.

## Relevant Context

The same code has to run in three places: on a laptop with SQLite, in Docker
with PostgreSQL, and in the tests. Only the configuration differs.

## Key Observation

If configuration is read in many places with `os.environ`, a missing value is
found only when that code path runs. Reading it once into a typed object
fails at startup instead, with the name of the missing setting.

## Solution

1. **Typed settings** with `pydantic-settings`, read from the environment or
   a `.env` file, and created once:

    ``` python
    class Settings(BaseSettings):
        database_url: str = "sqlite:///./neighborlend.db"
        jwt_secret: SecretStr = Field(repr=False, frozen=True)
        jwt_algorithm: Literal["HS256"] = Field(default="HS256", frozen=True)
        jwt_expire_minutes: int = 60 * 24 * 7
        ...

    @lru_cache
    def get_settings() -> Settings:
        return Settings()
    ```

    `SecretStr` keeps secrets out of logs and error messages.

2. **One database session per request**, opened and always closed by a
   dependency:

    ``` python
    def get_db():
        db = SessionLocal()
        try:
            yield db
        finally:
            db.close()
    ```

    SQLite needs `check_same_thread=False`, because FastAPI may use the
    session from a different thread than the one that created it.

3. **One router per area**, registered with a prefix and a tag:

    ``` python
    app.include_router(auth.router, prefix="/api/auth", tags=["Auth"])
    app.include_router(listings.router, prefix="/api/listings", tags=["Listings"])
    ```

**Because**

With `Depends(get_db)`, a test can replace the database for the whole app in
one line, and the `finally` block guarantees that connections go back to the
pool even when an endpoint raises an exception.
