# 0001 — Anonymize users instead of deleting them

- Status: accepted (ES-67)
- Context: users can ask to delete their account (RF02/RF04/RNF03, LGPD). Hard delete orphaned term acceptances (V12 dropped the FK) and loses the consent trail.
- Decision: `DELETE /api/users/me` and admin `DELETE /api/users/{id}` anonymize: name `Anonymized`, email `anon-<uuid>@anonymized.invalid`, unusable password, `active=false`, status `ANONYMIZED`. The row and term acceptances are kept; the email is freed so the person can register again as a new account. Self-requests create a `data_subject_requests` row; every anonymization writes `audit_log` (ids only, no PII).
- Consequences: no personal data remains in `users`; V15 restores `fk_uta_user`. Old tokens stop working because the JWT subject (email) no longer exists. Retention periods for the audit tables are not defined (Unverified legal review).
