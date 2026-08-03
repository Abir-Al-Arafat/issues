# Express.js Route Ordering Issue

## Problem

When hitting the endpoint `GET {{baseURLLocal}}/rides/nearby?lng=13.1913&lat=32.8872`, the server returns a `500 Internal server error`.

However, moving the `router.get("/:rideId", controller.rideById)` route to the bottom of the route definitions magically fixes the issue.

### The Broken Setup

```typescript
// User (Rider/Driver) Endpoints
router.get("/self", controller.getMyRides);
router.get("/:rideId", controller.rideById); // <-- Problem is here

// Admin Endpoints
router.get("/", controller.getAllRides); // Admin only

// ... other routes ...

// Driver actions
router.get("/nearby", upload.none(), controller.getNearby); // <-- Never gets hit
```

## Root Cause

This is a classic Express.js routing trap caused by **route ordering** and **dynamic route parameters** (like `/:rideId`).

Express evaluates routes **top-down**, in the exact order they are defined.

In the broken setup, the dynamic route `/:rideId` is defined **before** the specific route `/nearby`. When the request `GET /rides/nearby` is made, Express does the following:

1. Checks `router.get("/self")` -> _No match._
2. Checks `router.get("/:rideId")` -> **Match!**

Because `/:rideId` acts as a wildcard, Express thinks the string `"nearby"` is the ID of a ride. It routes the request to `controller.rideById` and sets `req.params.rideId = "nearby"`.

The `rideById` controller then attempts to query the database using `"nearby"` as the ID. Since `"nearby"` is an invalid ID format (e.g., not a valid UUID or MongoDB ObjectId), the database driver throws an error, resulting in a `500 Internal Server Error`.

## Solution

Move the dynamic parameter route (`/:rideId`) to the bottom of the file, **after** all specific static routes.

### The Working Setup

```typescript
// Driver actions
router.get("/nearby", upload.none(), controller.getNearby); // <-- Gets hit successfully

// ... other routes ...

// Catch-all dynamic route at the bottom
router.get("/:rideId", controller.rideById);
```

## The Golden Rule of Express Routing

**Always put specific, static routes before dynamic, parameterized routes.**

Dynamic routes (`/:id`) act as wildcards and will eagerly swallow up requests meant for specific endpoints if placed too high up in the router.
