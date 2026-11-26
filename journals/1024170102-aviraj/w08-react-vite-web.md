# Week 8 : Improvement Plan: From React Native to a React + Vite Web App

## Problem

The proposal planned a mobile app in React Native with Expo. At the prototype
review, the main difficulty was showing the app to others: every evaluator or
tester needed the app installed on a phone, and each change had to be rebuilt
for the device.

## Relevant Context

The screens built so far (Weeks 4–7) used plain React components, CSS and
Axios. None of them depended on a native phone feature; the camera and
location are both available to web pages through browser APIs
(`<input type="file" accept="image/*" capture>` and
`navigator.geolocation`).

## Key Observation

For this project, a web app gives the same features with a much shorter path
to users: a single URL that works on a phone or a laptop, and every push
deployed automatically. The existing React code could be kept almost as it
was.

## Solution

1. Move the frontend to **React + Vite**. Vite serves the app in development
   and builds static files into `dist/` for production.
2. Use **React Router** for the routes from the screen map.
3. Deploy the static build on **Vercel**. Because routing happens in the
   browser, every path must return `index.html`; otherwise reloading a page
   such as `/listing/42` gives a 404 from the server. `frontend/vercel.json`:

    ``` json
    {
      "buildCommand": "npm run build:web",
      "outputDirectory": "dist",
      "rewrites": [
        { "source": "/(.*)", "destination": "/index.html" }
      ]
    }
    ```

4. The backend URL comes from the environment variable `VITE_API_BASE_URL`,
   so the same build works against a local or the deployed backend.

An Android wrapper (Capacitor) was tried along the way. It was removed later
because the web deployment already covered every use case.

**Because**

A single-page application has only one real HTML file. The rewrite rule hands
every URL to that file, and React Router then decides what to show. Changing
the platform at the improvement-plan stage was cheap because the UI had been
built from plain React components.
