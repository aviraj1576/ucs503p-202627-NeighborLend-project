# Week 3 : Listing the API Resources and Agreeing on Conventions

## Goal

Turn the use cases into a list of REST resources, so that the frontend and
the backend can be developed in parallel against the same contract.

## Relevant Context

The backend is written in Python with FastAPI, which generates interactive
OpenAPI (Swagger) documentation at `/docs` from the route definitions. The
frontend team (Week 3 screen map) needed to know which calls each screen
would make.

## Key Observation

Most use cases are either *create/read a thing* (a listing, a message) or
*move a thing to its next state* (accept an offer, confirm a return). The
first kind maps to plain resources; the second is clearer as a named action
under the resource than as a generic `PATCH` of a status field.

## Solution

| Area | Prefix | Examples |
|---|---|---|
| Accounts | `/api/auth` | `POST /register`, `POST /verify-email`, `POST /login`, `GET /me` |
| Listings | `/api/listings` | `POST /`, `GET /search`, `GET /{id}`, `PUT /{id}` |
| Chat and offers | `/api` | `POST /conversations/{id}/messages`, `POST /offers/{id}/accept` |
| Rentals | `/api/borrow-requests` | `POST /{id}/handover/...`, `POST /{id}/return/...` |
| Notifications | `/api/notifications` | `GET /` |

Conventions agreed for every router:

- plural nouns for collections; actions as sub-paths (`/offers/{id}/counter`);
- **401** not logged in, **403** logged in but not allowed, **404** not
  found, **409** not allowed *in the current state*, **422** invalid input;
- errors return `{"detail": "<message for the user>"}`, so the frontend can
  show the text directly.

Each area is a separate `APIRouter` registered in `main.py` with its prefix
and a tag, so Swagger groups the endpoints by area.

**Because**

Separating 403 from 409 tells the frontend whether to hide an action (the
user may never do it) or only disable it for now (wrong state). Agreeing on
the error shape once meant no screen needed special error handling.
