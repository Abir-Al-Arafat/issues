# Problem: Express Route Ordering Causing Authentication Issues

## Problem Description

While implementing a user status toggler feature, I encountered an issue where the endpoint `PATCH /users/toggle-my-status` was incorrectly matching with admin-only routes. The authentication middleware was showing only admin roles (`['admin', 'sub_admin', 'super_admin']`) instead of all authorized roles including `'user'` and `'rider'`.

## Root Cause

**Express.js matches routes in the order they are defined.** When a request comes in, Express checks each route sequentially and uses the first match it finds.

In our routes file, we had:

```typescript
// ❌ WRONG ORDER
router.patch(
  '/:id', // This matches ANY string including "toggle-my-status"
  auth(USER_ROLE.admin, USER_ROLE.sub_admin, USER_ROLE.super_admin),
  upload.single('profile'),
  parseData(),
  userController.updateUser,
);

router.patch(
  '/toggle-my-status', // This never gets reached!
  auth(
    USER_ROLE.admin,
    USER_ROLE.sub_admin,
    USER_ROLE.super_admin,
    USER_ROLE.user,
    USER_ROLE.rider,
  ),
  userController.toggleMyStatus,
);
```

When a request came to `PATCH /users/toggle-my-status`:

1. Express checked the first route: `PATCH /:id`
2. It matched! (treating "toggle-my-status" as the `:id` parameter)
3. The admin-only auth middleware was executed
4. The second route was never checked

## Solution

**Move all specific paths BEFORE dynamic parameter routes:**

```typescript
// ✅ CORRECT ORDER

// Specific paths first
router.patch(
  '/toggle-my-status', // Specific path
  auth(
    USER_ROLE.admin,
    USER_ROLE.sub_admin,
    USER_ROLE.super_admin,
    USER_ROLE.user,
    USER_ROLE.rider,
  ),
  userController.toggleMyStatus,
);

router.patch(
  '/update-my-profile', // Specific path
  auth(
    USER_ROLE.admin,
    USER_ROLE.sub_admin,
    USER_ROLE.super_admin,
    USER_ROLE.user,
  ),
  upload.single('profile'),
  parseData(),
  userController.updateMyProfile,
);

// Dynamic parameter routes last
router.patch(
  '/:id', // Dynamic parameter route
  auth(USER_ROLE.admin, USER_ROLE.sub_admin, USER_ROLE.super_admin),
  upload.single('profile'),
  parseData(),
  userController.updateUser,
);
```

## Key Takeaways

### Route Ordering Rules in Express:

1. **Specific paths before dynamic parameters**: `/toggle-my-status` before `/:id`
2. **More specific before less specific**: `/users/profile/edit` before `/users/:id`
3. **Static routes before wildcards**: `/api/health` before `/api/*`

### Best Practices:

```typescript
// Order routes like this:
router.get('/'); // Root path
router.get('/specific-path'); // Specific static paths
router.get('/another-specific'); // More specific paths
router.get('/:id'); // Dynamic single parameters
router.get('/:id/nested'); // Nested with parameters
router.get('/*'); // Wildcards last
```

## Impact

After reordering the routes:

- ✅ `PATCH /users/toggle-my-status` correctly allows users and riders to access
- ✅ `PATCH /users/:id` still works for admin updates
- ✅ Authentication middleware receives the correct role list
- ✅ No conflicts between similar route patterns

## Related Files Modified

- `src/app/modules/users/users.routes.ts` - Reordered all routes
- `src/app/modules/users/user.controller.ts` - Added `toggleMyStatus` controller
- `src/app/modules/users/user.service.ts` - Added `toggleUserStatus` service
- `prisma/schema.prisma` - Added `onlineStatus` enum and field

## Prevention

To avoid this issue in the future:

1. **Always define specific paths first** in route files
2. **Group routes by specificity** (static → dynamic → wildcard)
3. **Add comments** to mark where dynamic routes begin
4. **Test with different user roles** during development
5. **Use route debugging** to verify which route is matched

## Testing

After the fix, verify with:

```bash
# As a regular user
PATCH /users/toggle-my-status
# Should return: "Your status is now online/offline"

# As an admin
PATCH /users/60d5ec49f1b2c8b1f8c4e5a1/:id
# Should toggle another user's status
```

---

**Date Solved:** February 17, 2026  
**Developer:** Development Team  
**Severity:** Medium (blocked user feature access)  
**Time to Resolve:** ~15 minutes after identifying root cause
