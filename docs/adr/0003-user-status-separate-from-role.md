# 0003 — User status separate from role

- Status: accepted (ES-67)
- Context: `PENDING` was a role, so approval changed the role, and rejection or anonymization had no representation.
- Decision: `role` is `USER | ADMIN`; `status` is `PENDING | ACTIVE | REJECTED | ANONYMIZED` (V13). `rejection_reason` is required when status is `REJECTED` (DB check). Login requires `ACTIVE`.
- Consequences: clients must read `status`, not `role`, to know approval state. Re-registration after rejection is undefined (backlog).
