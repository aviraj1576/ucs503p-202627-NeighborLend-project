# Week 11 : Registration with a CAPTCHA and an Email OTP

## Goal

Stop automated sign-ups, and make sure every account belongs to a real email
address before the user can list or borrow anything.

## Relevant Context

Registration is `POST /api/auth/register`, then `POST /api/auth/verify-email`
with a 6-digit code sent by email. `get_current_user` already rejects
unverified accounts (Week 6).

## Key Observation

Both the CAPTCHA and the OTP are short secrets that an attacker could try to
guess or reuse. Each one therefore needs an expiry, a limit on attempts, and
storage that does not reveal the value.

## Solution

**Math CAPTCHA.** The server creates the question and stores only the
answer, with a 5-minute expiry. A challenge can be checked only once: the row
is locked, compared, and deleted whether the answer was right or wrong.

``` python
num1, num2 = random.randint(1, 10), random.randint(1, 10)
question = f"{num1} + {num2} = ?"
...
challenge = db.query(CaptchaChallenge).filter(...).with_for_update().first()
is_valid = challenge.expires_at > datetime.utcnow() and int(answer) == challenge.answer
db.delete(challenge)
db.commit()
```

**Email OTP.** The code comes from the `secrets` module and is stored only as
an HMAC, keyed with the server secret:

``` python
def create_email_otp() -> str:
    return f"{secrets.randbelow(1_000_000):06d}"

def hash_email_otp(user_id: str, code: str) -> str:
    message = f"{user_id}:{code}".encode("utf-8")
    return hmac.new(settings.jwt_secret.get_secret_value().encode("utf-8"),
                    message, hashlib.sha256).hexdigest()

def verify_email_otp(user_id, code, expected_hash) -> bool:
    return hmac.compare_digest(hash_email_otp(user_id, code), expected_hash)
```

A code expires after 10 minutes. After 5 wrong attempts the endpoint answers
`429 Too many incorrect codes` and the user must request a new code.

**Because**

There are only one million 6-digit codes, so a plain hash could be reversed
by trying them all; an HMAC needs the server's secret as well. Including the
user id in the message means a code is valid only for that user.
`compare_digest` takes the same time whether the first or the last character
differs, so response times reveal nothing about the code.
