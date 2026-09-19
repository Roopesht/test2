# Mock: Firebase Sign-In

## Sign-In Screen (staff or patient)

```
+----------------------------------------+
|            Maternal Care Platform       |
|------------------------------------------|
|                                          |
|   Sign in to continue                    |
|                                          |
|   Email                                  |
|   [ jane.doe@example.com            ]    |
|                                          |
|   Password                               |
|   [ ********************            ]    |
|                                          |
|             [   Sign In   ]              |
|                                          |
|   Forgot password?                       |
|                                          |
+----------------------------------------+
```

## Auth Flow (email/password, Firebase replaces JWT issuance)

```
 Staff or Patient        Client App           Firebase Auth         API Gateway / Backend
       |                    |                       |                        |
       |  enters email/pwd  |                       |                        |
       |------------------->|                       |                        |
       |                    |  signInWithPassword    |                        |
       |                    |----------------------->|                        |
       |                    |                       |                        |
       |                    |   Firebase ID Token     |                        |
       |                    |<-----------------------|                        |
       |                    |                       |                        |
       |                    |  API request + ID token (Authorization: Bearer) |
       |                    |------------------------------------------------>|
       |                    |                       |   verify ID token       |
       |                    |                       |   (no custom JWT minted)|
       |                    |                       |   apply RBAC checks     |
       |                    |                       |   (Identity Service)    |
       |                    |          200 OK / response                     |
       |                    |<------------------------------------------------|
```

Notes:
- Same flow applies to both hospital staff and patients — no separate auth mechanism per user type.
- The Identity Service still owns RBAC decisions, but they run against the Firebase-verified identity rather than a locally-issued JWT.
- Patients only reach this screen after being registered by hospital staff (registration itself is out of scope here).
