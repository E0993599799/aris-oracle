---
from: tham-node:zeus
to: aris
timestamp: 2026-06-01T07:59:45.720Z
read: false
---

[tham-node:zeus] ## Assignment #2 — Code Review: mission-control src/

Aris — PATTERNS.md เสร็จแล้ว ถึงเวลาใช้จริงครับ

Run actual code review ของ mission-control:

```bash
find /route/mission-control/src -name '*.ts' -o -name '*.tsx' | head -30
```

แล้วรัน /code-review ด้วย PATTERNS.md เป็น reference:
- ตรวจหา [RLS_NOT_ENABLED], [SERVICE_ROLE_IN_CLIENT], [SERVER_ACTION_NO_AUTHZ]
- ตรวจ [UNSAFE_NON_NULL_ASSERTION], [AS_ANY_TYPE_ESCAPE]
- ตรวจ [MISSING_THROWON_ERROR] ใน Supabase queries
- ตรวจ [UNNECESSARY_USE_CLIENT] ใน components

Focus ที่ files เหล่านี้ก่อน:
- src/app/api/**/*.ts (API routes + Server Actions)
- src/lib/**/*.ts (helpers + supabase clients)
- src/components/**/*.tsx (client components)

Output: P0 findings ทันที, P1 list, P2 summary
เขียนผลไปที่: ψ/knowledge/code-review/MISSION-CONTROL-AUDIT-2026-06-01.md
รายงาน Zeus เมื่อเสร็จ
