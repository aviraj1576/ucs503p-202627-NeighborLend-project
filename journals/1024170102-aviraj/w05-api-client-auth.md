# Week 5 : One API Client and Protected Routes

## Problem

Every page calls the backend, and most calls need the user's login token. If
each page builds its own request and adds the token itself, one forgotten
header gives a confusing `401`, and an expired token leaves the user on a page
that silently fails.

## Relevant Context

The backend issues a JWT at login and expects it in an
`Authorization: Bearer <token>` header. The frontend keeps the token in
`localStorage` under the key `nl_token`.

## Key Observation

Adding the token and reacting to an expired session are the same for every
request. They belong in one place, not in every page.

## Solution

1. **One Axios instance** in `src/api/client.js`. A request interceptor adds
   the token, and a response interceptor sends the user to the login page on a
   `401` (except for the login and register calls themselves):

    ``` js
    const api = axios.create({
      baseURL: import.meta.env?.VITE_API_BASE_URL || '/api',
    });

    api.interceptors.request.use((config) => {
      const token = localStorage.getItem('nl_token');
      if (token) config.headers.Authorization = `Bearer ${token}`;
      return config;
    });
    ```

2. **An `AuthContext`** loads the current user once at start-up with
   `GET /auth/me` and exposes `user`, `isAuthenticated` and `loading`.

3. **A `ProtectedRoute` wrapper** for every route marked "login: yes" in the
   screen map:

    ``` jsx
    const ProtectedRoute = ({ children }) => {
      const { isAuthenticated, loading } = useAuth();
      if (loading) return <div>Loading...</div>;
      if (!isAuthenticated) return <Navigate to="/login" />;
      return children;
    };
    ```

**Because**

The `loading` check matters: without it, a logged-in user who refreshes the
page is sent to `/login` for a moment, because the user is not known until
`/auth/me` returns. The route guard only improves the experience; the backend
still checks the token on every protected endpoint.
