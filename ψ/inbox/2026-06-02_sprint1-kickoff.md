# Sprint 1 Kickoff Brief — Aris Quality Gate

**From**: Aris  
**To**: codex-rider (04-dheva / 04-luxi)  
**Date**: 2026-06-02  
**Based on**: Luxi Sprint Plan `orry-p0-sprint-plan.md`  
**Status**: APPROVED with 1 correction  
**Tag**: `[MARCUZ:Aris]`

---

## Quality Gate Result: GO ✅

Sprint 1 plan is technically sound. DB schema for vendors is confirmed complete.  
One correction below — must apply before execution.

---

## Correction (Plan Error)

**Luxi wrote**: update `lib/navigation.ts`  
**Actual**: nav items live in `components/Sidebar/Sidebar.tsx` — `NAV_ITEMS` array  
`lib/navigation.ts` is next-intl routing utility only (Link, redirect, usePathname)

**Fix**: ทุกครั้งที่ sprint plan บอกให้ update `lib/navigation.ts`  
→ ให้ update `components/Sidebar/Sidebar.tsx` ใน `NAV_ITEMS` array แทน

---

## Task 1.1 — Company Setup Enhancement

**File to modify**: `app/[locale]/(app)/settings/page.tsx` (314 lines, มีอยู่แล้ว)

**Files to create**:
```
components/modules/settings/
├── CompanyProfileSection.tsx
├── BranchSection.tsx
├── DocumentNumberSection.tsx
└── FiscalYearSection.tsx
```

**Pattern**: ดู `components/modules/customers/CustomerForm.tsx` เป็น template

**Key specs**:
- Settings page เพิ่ม 4 tabs: Company / Branches / Documents / Fiscal Year
- Thai Tax ID validation: ต้องเป็น 13 digits เท่านั้น (`/^\d{13}$/`)
- Doc number preview: real-time แสดง pattern เช่น `SO-2026-00001`
- Branch table: code, name, is_main_office (toggle), is_active (toggle)
- First-run banner: ถ้า `company.name` ว่าง → แสดง "ตั้งค่าบริษัทก่อนใช้งาน"
- DB: ใช้ `companies` + `branches` tables — ไม่ต้อง migration

**Effort**: 2 วัน

---

## Task 1.2 — Vendors Module

### Files to create:

**1. Page (SSR)**
```
app/[locale]/(app)/vendors/page.tsx
```
Pattern: copy `app/[locale]/(app)/customers/page.tsx` แล้วปรับ

**2. Components**
```
components/modules/vendors/
├── VendorsList.tsx        — table: name | tax_id | contact | WHT rate | payment terms | status
├── VendorForm.tsx         — create/edit modal/page
├── VendorDetail.tsx       — detail + purchase history
└── VendorListSkeleton.tsx — loading state
```
Pattern: copy `components/modules/customers/` ทั้ง folder แล้วปรับ:

| Customer field | Vendor field |
|----------------|-------------|
| `credit_limit` | `withholding_tax_rate` (DECIMAL 0-100%) |
| — | `bank_account` (เพิ่ม) |
| — | `payment_terms_id` selector dropdown |

**3. API Route**
```
app/api/vendors/route.ts
```
Pattern: copy `app/api/customers/route.ts` ทั้งไฟล์ แล้วปรับ:
- import จาก `@/lib/vendors` แทน `@/lib/customers`
- filter type: `{ search?, status?, sortBy? }`

**4. Lib types (ใหม่)**
```
lib/vendors/
├── types.ts    — Vendor interface, VendorFilter interface
├── queries.ts  — fetchVendors(), fetchVendorById()
└── index.ts    — barrel export
```
Pattern: copy `lib/customers/` ทั้ง folder แล้วปรับ field names

**5. Sidebar navigation** ← CORRECTION จากแผน Luxi
```
components/Sidebar/Sidebar.tsx
```
เพิ่มใน `NAV_ITEMS` array:
```typescript
{ label: 'Vendors', path: 'vendors', icon: '🏪' },
```
วางหลัง `{ label: 'Purchase', path: 'purchase', icon: '🛍️' }`

### Vendor UX specs:
- WHT rate badge: ถ้า `withholding_tax_rate > 0` → badge สีส้ม (visual reminder ก่อน pay)
- Filter: All / Active / Inactive + search by name/tax_id
- Empty state: "เพิ่ม Vendor แรก" message

**Effort**: 3 วัน

---

## Execution Order (strict)

```
1. Task 1.1 ก่อน — settings enhancement (2d)
   └── test ใน dev ก่อน proceed

2. Task 1.2 — vendors module (3d)
   ├── lib/vendors/ types + queries ก่อน
   ├── api/vendors/route.ts
   ├── components/modules/vendors/
   └── app/[locale]/(app)/vendors/page.tsx สุดท้าย
   └── Sidebar.tsx update (ไม่ใช่ lib/navigation.ts!)
```

---

## DB Confirmation (ไม่ต้อง migration)

Vendors schema verified ใน `supabase/migrations/20260517142016_init_orry_serenity_schema.sql`:

```sql
-- ✅ columns ครบตามที่ Luxi spec ไว้:
withholding_tax_rate DECIMAL(5,2) DEFAULT 0
bank_account VARCHAR(50)
payment_terms_id UUID
is_active BOOLEAN DEFAULT TRUE
```

---

## Acceptance Criteria

### Task 1.1 Done When:
- [ ] Settings page มี 4 tabs ทำงานได้
- [ ] บันทึก company info ได้ (save + reload แสดงค่าเดิม)
- [ ] Tax ID validation reject ถ้าไม่ใช่ 13 digits
- [ ] Doc number preview แสดง live
- [ ] First-run banner แสดงถ้า company.name ว่าง

### Task 1.2 Done When:
- [ ] `/vendors` page โหลดได้, แสดง list
- [ ] Filter by status + search ทำงาน
- [ ] Create vendor ผ่าน form ได้ (บันทึกลง DB)
- [ ] WHT badge แสดงถ้า rate > 0
- [ ] "Vendors" nav item ปรากฏใน Sidebar
- [ ] `/api/vendors` GET ตอบ `{ success, vendors, count }`
- [ ] TypeScript: ไม่มี `any` ใหม่ในไฟล์ที่สร้าง

---

[MARCUZ:Aris]
