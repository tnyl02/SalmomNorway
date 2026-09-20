# Implementation Plan: ตรวจสอบสถานะอุปกรณ์ (Equipment Status)
Spec: SPEC-EQS-001 (specs/001-equipment-status/spec.md, Draft v2) | Plan status: Draft v1 | Updated: 2569-09-20

## 1. Summary
ฟีเจอร์ read-only ให้ผู้ใช้ที่ยืนยันตัวตนแล้วดูรายการอุปกรณ์แบบ "1 แถว = 1 รุ่น/ชนิด"
(พร้อมให้ยืม/ทั้งหมด) เจาะลงรายชิ้นตามรหัสครุภัณฑ์ ค้นหา/กรอง และดูรายละเอียดตามสิทธิ์
แนวทางหลัก:
- ฟีเจอร์ไม่เขียนสถานะอุปกรณ์เลย (CON-DATA-02) จึงออกแบบเป็น **read model** บน PostgreSQL
  ที่ UC-07/08/09/10 เป็นผู้เขียน
- บังคับสิทธิ์ (ภาควิชา, การซ่อนชื่อผู้ยืม) **ที่ฝั่ง server** ทั้งหมด ไม่พึ่งการซ่อนที่หน้าจอ
- ไม่ทำ cache ในแอป ให้ query ตรงจาก DB เพื่อให้ผ่าน NFR-DATA-01 (คลาดเคลื่อน <= 30 วินาที)
  ข้อมูล 1,000 รายการ + index เพียงพอต่อ NFR-PERF-01

## 2. Technical Context
spec กำหนดไว้เพียง PostgreSQL, HTTPS, Docker (CON-TECH-01/02) ส่วนที่เหลือ **เป็นข้อเสนอ รอยืนยัน**

| หัวข้อ | ตัวเลือกที่เสนอ | หมายเหตุ |
|---|---|---|
| Database | PostgreSQL 16 | CON-TECH-01 (ตามสั่ง) |
| API | Go (net/http + pgx) | เสนอ, เปลี่ยนได้ ไม่กระทบ design ส่วนอื่น |
| Web | React + Vite, responsive (mobile-first) | NFR-USE-01 |
| Migration | golang-migrate (SQL ล้วน) | |
| TLS / proxy | nginx หรือ Caddy หน้า API+Web, TLS >= 1.2, HSTS | CON-TECH-02, NFR-SEC-01 |
| Deploy | Docker Compose บน on-campus server / cloud VM | CON-TECH-02 |
| Test | go test + testcontainers (integration), Playwright (e2e), k6 (load) | |
| Auth | รับ token/session จาก auth service (UC-13) | โปรโตคอลยังไม่ระบุ ดู Q-P1 |

Scale: อุปกรณ์ 1,000 ชิ้น, ผู้ใช้พร้อมกัน 50 คน (ขนาดเล็ก ไม่ต้องมี cache/queue/replica)

Constitution check: ยังไม่มี `.specify/memory/constitution.md` ในโปรเจกต์ -> ข้าม

## 3. Project Structure (proposed)
```
SalmomNorway/
  api/
    cmd/server/main.go
    internal/
      auth/        # ตรวจ token, สร้าง Principal{userID, role, departmentID}
      equipment/   # handler, service, repository, dto (แยก dto ตาม role)
      audit/       # เขียน audit_log
      platform/    # config, db pool, http middleware (request id, error mapper)
  web/
    src/pages/EquipmentList, EquipmentDetail
    src/components/StatusBadge, ModelRow, ItemTable, EmptyState, ErrorState
  db/migrations/   # 0001_equipment_read_model.sql, 0002_audit_log.sql, ...
  deploy/          # docker-compose.yml, nginx.conf
  tests/           # e2e (Playwright), load (k6)
  specs/001-equipment-status/{spec.md, plan.md}
```

## 4. Data Model
ตารางเจ้าของข้อมูลจริงคือ UC-10 (equipment) และ UC-08/09 (loan) ฟีเจอร์นี้ **อ่านอย่างเดียว**
ถ้ายังไม่มีตารางจากฟีเจอร์เหล่านั้น ให้สร้าง migration เบื้องต้นตาม schema ด้านล่างแล้วให้ฟีเจอร์นั้นรับช่วงต่อ

```
department(id PK, name)
category(id PK, name)                                   -- ASM-03, Q-04 (ยังไม่ตัดสินว่าตายตัวหรือเพิ่มได้)
equipment_model(id PK, name, category_id FK, department_id FK)
equipment_item(
  id PK,
  asset_code TEXT UNIQUE NOT NULL,                      -- CON-DATA-03
  model_id FK,
  status TEXT NOT NULL CHECK (status IN
    ('available','pending_approval','borrowed','maintenance','damaged','lost')),  -- CON-DATA-01
  updated_at TIMESTAMPTZ NOT NULL)
loan(id PK, item_id FK, borrower_id, expected_return_date, returned_at NULL)   -- เขียนโดย UC-08/09
```

View สำหรับอ่าน (ซ่อน join ไว้ที่เดียว):
```
v_equipment_item_status =
  item + model + category + department
  + LEFT JOIN loan ON loan.item_id = item.id AND loan.returned_at IS NULL
  -> expected_return_date, borrower_id   (มีค่าเฉพาะสถานะ borrowed)
```

Index:
- `equipment_item(model_id, status)` -- นับพร้อมให้ยืม/ทั้งหมดต่อรุ่น
- `equipment_model(department_id, category_id)` -- scope ตามภาควิชา + กรองประเภท
- `pg_trgm` GIN บน `equipment_model.name` และ `equipment_item.asset_code` -- ค้นหา ILIKE
- `loan(item_id) WHERE returned_at IS NULL` (partial)

audit_log (ตารางใหม่ของฟีเจอร์นี้):
```
audit_log(id, actor_id, actor_role, occurred_at, action CHECK IN ('list','search','detail'),
          query JSONB, asset_code NULL, outcome CHECK IN ('ok','forbidden','error'))
```
ตารางเป็น append-only (ไม่ให้ app user ทำ UPDATE/DELETE)

## 5. API Contract (REST, JSON, HTTPS เท่านั้น, ทุก endpoint ต้องมี auth)

| Endpoint | ใช้กับ | หมายเหตุ |
|---|---|---|
| `GET /api/v1/equipment/models?q=&category=&status=&department=&page=&pageSize=` | FR-01, 02, 03, 09, 10 | 1 แถว = 1 รุ่น |
| `GET /api/v1/equipment/models/{modelId}/items?q=&status=` | FR-01, 04, 05, 06, 07, 09 | รายชิ้นตามรหัสครุภัณฑ์ |
| `GET /api/v1/equipment/items/{assetCode}` | FR-04-07, 09, 10 | รายละเอียดรายชิ้น |
| `GET /api/v1/equipment/summary` | FR-07 | เฉพาะ staff/lecturer, นับตามสถานะ |

Response (ตัวอย่างแถวรุ่น): `{modelId, name, category, availableCount, totalCount}`
- `availableCount` นับเฉพาะ `status='available'` (ไม่นับ pending_approval ตาม FR-09)
- `availableCount/totalCount` คำนวณจากทั้งรุ่นเสมอ ไม่ถูกลดตามตัวกรอง (ดู Q-P2)

Response (รายชิ้น): `{assetCode, status, expectedReturnDate?, canRequestLoan, borrowerName?}`
- `canRequestLoan = (status == 'available')` คำนวณที่ server ตาม FR-04/05/06/09
- `expectedReturnDate` ส่งเฉพาะ `borrowed`
- `borrowerName` **ไม่อยู่ใน DTO ของ student** (แยก struct ตาม role) มีเฉพาะ staff/lecturer (DOM-PRIV-01, FR-07)

Error contract:
| กรณี | HTTP | body | อ้างอิง |
|---|---|---|---|
| ไม่มี/หมดอายุ session | 401 | `{code:"UNAUTHENTICATED"}` | AC-09 |
| ต่างภาควิชา | 403 | `{code:"FORBIDDEN_DEPARTMENT", message}` และ **ไม่มีข้อมูลอุปกรณ์** | FR-10, AC-13 |
| DB ล่ม/timeout | 503 | `{code:"UPSTREAM_UNAVAILABLE", retryable:true}` | FR-08, AC-08 |
| ไม่พบผลค้นหา | 200 | `{items:[], total:0}` (ไม่ใช่ error) UI แสดง empty state | FR-03 |

การส่งต่อไป UC-02 (FR-06): หน้าจอลิงก์ไป `/loans/new?assetCode=...` ด้วย assetCode เท่านั้น
UC-02 ต้องตรวจสถานะซ้ำที่ฝั่งตัวเอง (สถานะอาจเปลี่ยนระหว่างรอ) -- ระบุเป็น dependency ของ UC-02

## 6. Key Design Decisions

### 6.1 การควบคุมสิทธิ์ (DOM-ACCESS-01, DOM-PRIV-01, FR-07, FR-10)
- middleware `auth` สร้าง `Principal{userID, role, departmentID}` จากผลของ auth service (IF-IDP-01)
  ถ้าไม่ได้ `departmentID` -> ปฏิเสธ (fail closed) ไม่เดา
- Student: repository เติม `WHERE department_id = principal.departmentID` **ทุก query** ที่ชั้น repository
  ถ้า request ระบุ `department` อื่น หรือ `assetCode` ของภาควิชาอื่น -> 403 (`FORBIDDEN_DEPARTMENT`)
  ตามที่ AC-13 กำหนดให้แจ้ง "ไม่มีสิทธิ์" และไม่ส่งข้อมูลกลับ
  (ข้อแลกเปลี่ยน: 403 บอกได้ว่ารหัสนั้นมีอยู่จริง แต่ spec สั่งให้แจ้งไม่มีสิทธิ์ จึงคงตาม spec)
- Staff/Lecturer: ขอบเขตภาควิชายังไม่ระบุใน spec (ดู Q-P3) ชั่วคราวให้เห็นทุกภาควิชา และเก็บเป็น policy เดียวในโค้ด
  (`scopeFor(principal)`) เพื่อแก้จุดเดียวได้
- DTO แยกตาม role (`StudentItemDTO` / `StaffItemDTO`) -- ฟิลด์ที่นักศึกษาไม่มีสิทธิ์ไม่มีอยู่ใน struct
  ป้องกันการหลุดโดยไม่ตั้งใจ
- ต้องมี test ว่า response ของ student ไม่มีคำว่า borrower ใดๆ เลย

### 6.2 ความสดของข้อมูล (NFR-DATA-01) และความทนทาน (FR-08)
- ไม่ cache ที่ server; response ตั้ง `Cache-Control: no-store` ป้องกันเบราว์เซอร์/proxy เก็บค่าเก่า
- ฝั่ง web: เมื่อ request ล้มเหลว ให้ล้าง state รายการเดิมทิ้งก่อนแสดง error + ปุ่มลองใหม่
  (ห้ามใช้รูปแบบ stale-while-revalidate ที่โชว์ข้อมูลเก่า) และตั้ง timeout ของ request (เช่น 8 วินาที)
  เพื่อไม่ให้หน้าจอค้าง (AC-08)
- DB query timeout ที่ server (เช่น 5 วินาที) แล้วแปลงเป็น 503

### 6.3 ค้นหา/กรอง (FR-02, FR-03)
- `q` ค้นทั้ง `equipment_model.name` และ `equipment_item.asset_code` แบบ ILIKE + trigram index
- รุ่นจะแสดงเมื่อมี **อย่างน้อย 1 ชิ้นที่ตรงเงื่อนไข** ทั้ง q/category/status
- เมื่อเจาะลงรายชิ้น แสดงเฉพาะชิ้นที่ตรงตัวกรองเดียวกัน (ให้ AC-02 ผ่าน)
- parameterized query ทุกจุด ห้ามต่อสตริง SQL; จำกัดความยาว `q` (เช่น 100 ตัวอักษร)
- แบ่งหน้าแบบ offset (page size เริ่มต้น 20) เพียงพอสำหรับขนาดข้อมูลนี้

### 6.4 Audit (DOM-AUDIT-01, AC-11)
- บันทึกทุก request ประเภท list / search / detail: ผู้เข้าถึง, เวลา, เงื่อนไขที่ค้น (`query`), `assetCode` (กรณี detail)
  รวมถึงกรณีถูกปฏิเสธ (`outcome='forbidden'`) เพื่อใช้ตรวจสอบย้อนหลัง
- เขียนแบบ synchronous ใน request เดียวกัน (ปริมาณต่ำ, DB เดียวกับข้อมูลอยู่แล้ว)
  ถ้าเขียน audit ไม่สำเร็จ -> ตอบ 503 (fail closed) เพราะ spec ระบุ "ทุกครั้ง" (ดู Q-P4)
- `query` เก็บเฉพาะพารามิเตอร์ค้นหา ไม่เก็บ token/ข้อมูลส่วนตัวอื่น

### 6.5 Session หมดอายุ (AC-09)
- API ตอบ 401 -> web redirect ไปหน้า login ของ auth service พร้อม `returnTo` (path + query ปัจจุบัน)
  พร้อมข้อความแจ้ง; หลังล็อกอินกลับมาหน้าเดิม
- ตรวจ `returnTo` ให้เป็น path ภายในระบบเท่านั้น (กัน open redirect)

### 6.6 ความปลอดภัย/ความพร้อมใช้งาน (NFR-SEC-01, NFR-AVAIL-01)
- TLS 1.2+ ที่ reverse proxy, redirect HTTP->HTTPS, HSTS; API ไม่เปิดพอร์ตออกนอก Docker network ตรงๆ
- container `restart: unless-stopped` + `/healthz`; DB มี backup รายวัน; งาน maintenance/deploy นอกเวลา 08.30-16.30 จ-ศ
- เป้าหมาย 99% ในช่วงเวลาทำการ ~ downtime ไม่เกิน ~5 นาที/สัปดาห์ในช่วงนั้น

### 6.7 UI (NFR-USE-01)
- mobile-first: รายการรุ่นเป็นการ์ด/แถวที่กดเจาะได้, ช่องค้นหาอยู่บนสุด, ตัวกรองเป็น chip
- `StatusBadge` ใช้สี **และ** ข้อความ/ไอคอนแยกกันสำหรับทั้ง 6 สถานะ (FR-05, FR-09) ไม่พึ่งสีอย่างเดียว
- ปุ่ม "ส่งคำขอยืม" แสดงเมื่อ `canRequestLoan == true` เท่านั้น (ค่ามาจาก server)

## 7. Test Plan (Traceability กับ AC)

| AC | ชนิด | สิ่งที่ตรวจ |
|---|---|---|
| AC-EQS-01 | integration + e2e | seed 1 รุ่น 5 ชิ้น พร้อม 3 -> แสดง "3/5" และเจาะได้ 5 ชิ้น |
| AC-EQS-02 | integration | q="Notebook" + status=available คืนเฉพาะที่ตรง |
| AC-EQS-03 | e2e | ไม่พบ -> ข้อความ + ปุ่มล้างตัวกรอง |
| AC-EQS-04 | integration | ตั้ง borrowed (จำลอง UC-08) -> เห็นกำหนดคืน ไม่มี canRequestLoan |
| AC-EQS-05 | integration | ปิด loan (จำลอง UC-09) -> refresh <= 30 วินาที กลับ available |
| AC-EQS-06 | integration + e2e | damaged -> ป้ายแยก ไม่มีปุ่ม |
| AC-EQS-07 | integration | response student ไม่มีชื่อผู้ยืม / staff มี |
| AC-EQS-08 | integration + e2e | หยุด DB -> 503 + ปุ่มลองใหม่ ไม่มีข้อมูลเก่า |
| AC-EQS-09 | e2e | 401 -> หน้า login -> กลับหน้าเดิม |
| AC-EQS-10 | load (k6) | 1,000 รายการ, 50 VU, p95 <= 2 วินาที |
| AC-EQS-11 | integration | มีแถว audit_log ครบ actor/เวลา/assetCode |
| AC-EQS-12 | integration + e2e | pending_approval -> ป้ายแยก ไม่นับใน availableCount |
| AC-EQS-13 | integration | ต่างภาควิชา -> 403 และ body ไม่มีข้อมูลอุปกรณ์ทั้งจาก list และ detail ตรง |

ทดสอบเพิ่ม: DTO นักศึกษา (ไม่หลุดฟิลด์ borrower), `returnTo` ภายนอกระบบถูกปฏิเสธ, SQL injection ผ่าน `q`

## 8. Milestones
1. **M1 Data & auth skeleton**: migration + seed 1,000 รายการ, middleware auth -> Principal, healthz
2. **M2 List/search API**: models + items + filter + department scope (FR-01/02/03/09/10, AC-01/02/03/12/13)
3. **M3 Detail + role DTO + audit**: FR-04/05/06/07, AC-04/05/06/07/11
4. **M4 Web UI**: หน้ารายการ/รายละเอียด, empty/error state, redirect login (AC-08/09), mobile
5. **M5 Hardening**: k6 load test (AC-10), TLS/proxy, backup, usability test 10 คน (NFR-USE-01)

## 9. Risks & Dependencies
- **ตารางจาก UC-08/09/10 ยังไม่มี**: schema ในข้อ 4 เป็นข้อสมมติ ต้องตกลงกับเจ้าของ UC เหล่านั้นก่อน freeze
- **auth service (UC-13)**: ต้องยืนยันโปรโตคอลและต้องส่ง `departmentID` กลับมา (IF-IDP-01) ไม่มีจะทำ DOM-ACCESS-01 ไม่ได้
- **NFR-DATA-01** ตั้งอยู่บนข้อสมมติว่า UC-07-10 เขียน DB ทันที (ไม่มี batch/ล่าช้า)

## 10. Open Points ที่ต้องยืนยัน (นอกเหนือจาก Q-02, Q-04 ใน spec)
- **Q-P1** auth service ใช้โปรโตคอลอะไร (OIDC / SAML / token ภายใน) และส่ง role + department มาในรูปแบบใด
- **Q-P2** ตัวเลข "พร้อมให้ยืม/ทั้งหมด" ควรคงที่ทั้งรุ่น (ที่เสนอ) หรือเปลี่ยนตามตัวกรอง
- **Q-P3** เจ้าหน้าที่/อาจารย์เห็นทุกภาควิชา หรือเฉพาะภาควิชาของตน (spec ระบุเฉพาะนักศึกษา)
- **Q-P4** เขียน audit ไม่สำเร็จ ควรบล็อกการแสดงข้อมูล (fail closed ที่เสนอ) หรือปล่อยผ่านแล้วแจ้งเตือน
- **Q-P5** "กำหนดคืนโดยประมาณ" มาจากไหน (ฟิลด์ใน loan ที่ UC-08 กำหนด หรือคำนวณจากนโยบายจำนวนวันยืม)
- Q-01 ที่ spec แนะนำให้ยืนยันกับเจ้าหน้าที่ภาควิชาอีกครั้งก่อน freeze spec ยังค้างอยู่
