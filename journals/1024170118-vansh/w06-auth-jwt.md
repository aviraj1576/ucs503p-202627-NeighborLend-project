# Week 6 : Password Hashing and JWT Authentication

## Goal

Let users register and log in, and let every protected endpoint know who is
calling, without storing passwords in a readable form.

## Relevant Context

The frontend stores a token after login and sends it as
`Authorization: Bearer <token>` (Aviraj's Week 5 entry). The backend has no
server-side session store.

## Key Observation

Two separate problems need two separate tools:

- **storing passwords**: a slow, salted hash, so that a leaked database does
  not reveal passwords; `bcrypt` adds a random salt to every hash;
- **identifying the caller**: a signed token that the server can check
  without a database lookup of a session; a JWT signed with a secret key does
  this.

## Solution

``` python
def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")

def verify_password(plain: str, hashed: str) -> bool:
    return bcrypt.checkpw(plain.encode("utf-8"), hashed.encode("utf-8"))

def create_access_token(user_id: str) -> str:
    expire = datetime.utcnow() + timedelta(minutes=settings.jwt_expire_minutes)
    return jwt.encode({"sub": user_id, "exp": expire},
                      settings.jwt_secret.get_secret_value(),
                      algorithm=settings.jwt_algorithm)
```

A dependency turns the token into a user, and rejects unverified accounts:

``` python
def get_current_user(token: str = Depends(oauth2_scheme),
                     db: Session = Depends(get_db)) -> User:
    user_id = decode_access_token(token)
    if user_id is None:
        raise credentials_exception          # 401
    user = db.query(User).filter(User.id == user_id).first()
    if user is None or not user.is_verified:
        raise credentials_exception
    return user
```

Any endpoint that needs a logged-in user declares
`current_user: User = Depends(get_current_user)`.

**Because**

The `exp` claim makes a stolen token useless after it expires, and the
signature makes it impossible to change the `sub` (user id) without the
secret. Checking `is_verified` inside the dependency means no endpoint can
forget that unverified accounts must not act.
