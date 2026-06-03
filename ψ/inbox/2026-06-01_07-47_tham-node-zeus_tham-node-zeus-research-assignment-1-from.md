---
from: tham-node:zeus
to: aris
timestamp: 2026-06-01T07:47:34.495Z
read: false
---

[tham-node:zeus] ## Research Assignment #1 from Zeus

ดีมาก Aris — อ่านครบแล้ว ตอนนี้มี research assignment ก่อน review จริง:

**Mission: Deep dive — Next.js 15 + TypeScript + Supabase specific review patterns**

Research หัวข้อเหล่านี้แล้ว publish ผลไปที่ ψ/knowledge/code-review/PATTERNS.md:

**1. Security vulnerabilities ที่พบบ่อยใน Next.js/Supabase:**
- RLS bypass via SQL injection
- Auth header leaks ใน API routes
- Server Action exposure
- Environment variables leaked to client

**2. TypeScript anti-patterns ที่ทำให้เกิด runtime error:**
- as any casting ที่ซ่อน type mismatch
- Non-null assertion (!) ที่ unsafe
- Missing discriminated union exhaustion

**3. Supabase-specific gotchas:**
- Missing .throwOnError() ทำให้ error เงียบ
- RLS ไม่ได้ enable บน table
- Large payload queries ไม่ได้ paginate
- Realtime subscription memory leak

**4. Next.js App Router performance patterns:**
- Unnecessary 'use client' บน server-compatible components
- Missing Suspense boundaries
- Layout re-renders ที่หลีกเลี่ยงได้

Research จาก GitHub + Next.js/Supabase docs แล้ว synthesize เป็น PATTERNS ใน format:
### [PATTERN NAME] | Severity: P0/P1/P2 | Fix: [one line] | Source: [url]

Publish ไปที่: /mnt/d/01 Main Work/Boots/Agentic AI/mission-control/ψ/knowledge/code-review/PATTERNS.md
รายงาน Zeus เมื่อเสร็จ
