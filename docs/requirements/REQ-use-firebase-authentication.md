# Use Firebase for Authentication

## Summary
Adopt Firebase Authentication (email/password) as the authentication mechanism for the platform, for both hospital staff and patients, replacing the Identity Service's own JWT issuance.

## Background
- Decision: `dec-go-use-firebase` (parent feature: "Authentication and Session Management")
- The project blueprint assigns the Identity Service responsibility for "Authentication, session/JWT, RBAC," but explicitly leaves the auth mechanism itself as "Not specified (Identity Service scope only)."
- This requirement resolves that gap by specifying Firebase Authentication as the concrete mechanism, and clarifies how it interacts with the existing session/JWT responsibility already assigned to the Identity Service.

## Details
- **Sign-in method**: email/password, via Firebase Authentication.
- **Scope**: applies to every user type that signs in to the platform — both hospital staff and patients.
- **Session/JWT ownership**: Firebase Authentication fully replaces the Identity Service's own JWT issuance. Clients authenticate directly against Firebase; the Identity Service (and other backend services behind the API Gateway) verify Firebase-issued ID tokens instead of minting and validating their own JWTs.
- **RBAC**: Role-based access control, already assigned to the Identity Service per the blueprint, is unaffected in ownership — it continues to run server-side, evaluated against the authenticated Firebase identity, rather than moving to Firebase-native custom claims/roles.
- **Patient login timing unaffected**: per the blueprint, patients are registered by hospital staff before a patient ever signs in themselves. Firebase Authentication governs the sign-in step once a patient (or staff member) does log in; it does not change the staff-driven registration step that precedes it.

## Acceptance Criteria
- Hospital staff and patients can sign in with email/password credentials via Firebase Authentication.
- The API Gateway / backend services verify Firebase ID tokens on authenticated requests.
- The Identity Service no longer issues or validates a custom JWT for authentication — the Firebase ID token is the sole session credential.
- RBAC checks continue to apply per authenticated user, evaluated server-side against the verified Firebase identity.

## Out of Scope
- Additional sign-in methods (phone OTP, Google sign-in, etc.) — not confirmed as part of this decision; may be addressed by a future requirement.
- Patient self-registration flows — unaffected; patients continue to be registered by hospital staff before they sign in themselves.

## Resolved Open Questions
- **Sign-in method**: email/password.
- **Session/JWT ownership**: Firebase Authentication fully replaces the Identity Service's JWT issuance layer.
- **Scope**: applies to both hospital staff and patients.
