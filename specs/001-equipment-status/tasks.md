# Tasks: ตรวจสอบสถานะอุปกรณ์ (Equipment Status)
Spec: SPEC-EQS-001 (spec.md Draft v4) | Plan: plan.md Draft v1.3 | สร้าง: 2569-11-02 | แก้ตาม CR-002: 2569-11-02 | Owner: (ใส่ชื่อ)

สรุป
- 15 เฟสที่ใช้งาน (ฐานราก + 13 เฟสตาม AC + รอคำตอบ) รวม 30 task ยกเลิกโดย CR-002 2 task (T-15, T-25) เหลือ 28 task
- รอ Q-xx 7 task (แบ่งหน้า, วิธีค้นหา, timeout, จอมือถือ, ขอบเขตเจ้าหน้าที่, audit ของรายการ, ตัวเลขเมื่อกรอง, auth จริง)
- เฟสแรกที่ลองใช้งานได้จริง: เฟส 1 (เห็นรายการรุ่นและรายชิ้นบนหน้าจอ)

เทคโนโลยีตาม plan.md ข้อ 2: Go (net/http + pgx), PostgreSQL 16, React + Vite (ทีมเลือกเอง ไม่ได้มาจาก spec ยกเว้น PostgreSQL / HTTPS / Docker)
คำสั่งเปิดระบบ (ใช้ในทุกจุดตรวจ):
- ฐานข้อมูล: `docker compose -f deploy/docker-compose.yml up -d db`
- หลังบ้าน: `cd backend && go run ./cmd/server`
- หน้าจอ: `cd frontend && npm run dev`
- test: `cd backend && go test ./... -v` และ `cd frontend && npm test`

---

## เฟส 0: ฐานราก

### T-01 สร้างโครงหลังบ้าน Go หน้าจอ React และฐานข้อมูลใน Docker
- รองรับ: CON-TECH-01, CON-TECH-02
- ตรวจด้วย: ไม่มี AC ตรง ๆ เป็นงานพื้นฐานของทุก task
- ไฟล์ที่แตะ: `backend/go.mod`, `backend/cmd/server/main.go`, `backend/internal/config/config.go`, `backend/internal/platform/db.go`, `backend/internal/platform/http.go`, `deploy/docker-compose.yml`, `frontend/` (โครงจาก template รายวิชา), `.gitignore`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: ไม่มี
- เสร็จเมื่อ: `GET /healthz` ตอบ 200 เมื่อต่อฐานข้อมูลได้ และ `npm test` ของโครงหน้าจอผ่าน
- สถานะ: พร้อมทำ

### T-02 สร้างตาราง read model และข้อมูลตั้งต้น
- รองรับ: CON-DATA-01, CON-DATA-03, ASM-01
- ตรวจด้วย: ไม่มี AC ตรง ๆ เป็นงานพื้นฐานของ T-04 ขึ้นไป
- ไฟล์ที่แตะ: `backend/db/migrations/0001_equipment_read_model.sql`, `backend/db/seed/seed.sql`, `backend/internal/platform/migrate.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-01
- เสร็จเมื่อ: `TestMigration_StatusCheck` ผ่าน (insert สถานะนอก 3 ค่าถูกปฏิเสธ) และ `TestMigration_AssetCodeUnique` ผ่าน (รหัสครุภัณฑ์ซ้ำถูกปฏิเสธ)
- สถานะ: พร้อมทำ

### T-03 สร้าง middleware ยืนยันตัวตนแบบจำลอง
- รองรับ: IF-IDP-01, NFR-SEC-01 (ส่วน "ต้องยืนยันตัวตนก่อนทุกครั้ง")
- ตรวจด้วย: ไม่มี AC ตรง ๆ เป็นงานพื้นฐาน (AC-EQS-09 ตรวจในเฟส 12)
- ไฟล์ที่แตะ: `backend/internal/auth/principal.go`, `backend/internal/auth/middleware.go`, `backend/internal/auth/mock_provider.go`, `backend/internal/auth/middleware_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-01
- เสร็จเมื่อ: `TestAuth_NoToken_401` และ `TestAuth_MissingDepartment_Rejected` ผ่าน (ไม่ได้ภาควิชา = ปฏิเสธ ตาม plan 6.1)
- สถานะ: พร้อมทำ (auth จำลองเป็นการตัดสินใจของทีม ไม่ได้มาจาก spec โปรโตคอลจริงรอ Q-SH-03 ดู T-30)

---

## เฟส 1: เห็นรายการรุ่นและรายชิ้น (AC-EQS-01)

### T-04 สร้าง API รายการรุ่นพร้อมจำนวนว่าง/ทั้งหมด
- รองรับ: FR-EQS-01
- ตรวจด้วย: AC-EQS-01
- ไฟล์ที่แตะ: `backend/internal/equipment/repository.go`, `backend/internal/equipment/service.go`, `backend/internal/equipment/handler.go`, `backend/internal/equipment/dto.go`, `backend/internal/equipment/handler_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-02, T-03
- เสร็จเมื่อ: `TestAC_EQS_01_ModelRowCounts` ผ่าน (รุ่น 5 ชิ้น ว่าง 3 -> 1 แถว availableCount=3 totalCount=5)
- สถานะ: พร้อมทำ (ยังไม่แบ่งหน้า ดู T-23)

### T-05 สร้าง API รายชิ้นของรุ่น
- รองรับ: FR-EQS-01, CON-DATA-03
- ตรวจด้วย: AC-EQS-01
- ไฟล์ที่แตะ: `backend/internal/equipment/repository.go`, `backend/internal/equipment/handler.go`, `backend/internal/equipment/handler_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-04
- เสร็จเมื่อ: `TestAC_EQS_01_ItemsOfModel` ผ่าน (ได้ 5 ชิ้นตามรหัสครุภัณฑ์ พร้อมสถานะแต่ละชิ้น)
- สถานะ: พร้อมทำ

### T-06 สร้างหน้ารายการอุปกรณ์และรายชิ้น
- รองรับ: FR-EQS-01
- ตรวจด้วย: AC-EQS-01
- ไฟล์ที่แตะ: `frontend/src/pages/EquipmentList/EquipmentList.jsx`, `frontend/src/components/ModelRow.jsx`, `frontend/src/components/ItemTable.jsx`, `frontend/src/components/StatusBadge.jsx`, `frontend/src/api/client.js`, `frontend/src/__tests__/AC-EQS-01.test.jsx`
- แบบหน้าจอ: ไม่มี (ยังไม่มี mockup)
- ต้องทำหลัง: T-05
- เสร็จเมื่อ: `AC-EQS-01.test.jsx` ผ่าน (เห็น "ว่าง 3 / ทั้งหมด 5" และกดแล้วเห็น 5 ชิ้น)
- สถานะ: พร้อมทำ

#### จุดตรวจเฟส 1
- test ที่ต้องผ่านทั้งหมด: TestMigration_*, TestAuth_*, TestAC_EQS_01_*, AC-EQS-01.test.jsx
- เปิดระบบ: ดูคำสั่งด้านบน ล็อกอินจำลองเป็นนักศึกษาภาควิชาที่มีข้อมูลตั้งต้น
- คนลองเองตาม AC-EQS-01:
  Given ข้อมูลตั้งต้นมีรุ่นหนึ่ง 5 ชิ้น ว่าง 3 ชิ้น
  When เปิดหน้ารายการอุปกรณ์ แล้วเลือกรุ่นนั้น
  Then เห็นรุ่นนั้น 1 แถว มีประเภท และ "ว่าง 3 / ทั้งหมด 5" และเห็น 5 ชิ้นตามรหัสครุภัณฑ์พร้อมสถานะ
- ยังขาด: แบ่งหน้า (รอ Q-05), อุปกรณ์ที่มีคำขอค้างนับเป็นว่างไหม (รอ Q-SH-01)

---

## เฟส 2: ค้นหาและกรอง (AC-EQS-02)

### T-07 เพิ่มตัวกรองประเภทและสถานะใน API
- รองรับ: FR-EQS-02
- ตรวจด้วย: AC-EQS-02 (ส่วนกรองสถานะ)
- ไฟล์ที่แตะ: `backend/internal/equipment/repository.go`, `backend/internal/equipment/handler.go`, `backend/internal/equipment/handler_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-05
- เสร็จเมื่อ: `TestAC_EQS_02_FilterStatus` ผ่าน (กรอง "ว่าง" ได้เฉพาะชิ้นที่ว่าง) parameterized query ทุกจุด
- สถานะ: พร้อมทำ

### T-08 เพิ่มช่องค้นหาและตัวกรองบนหน้าจอ
- รองรับ: FR-EQS-02
- ตรวจด้วย: AC-EQS-02 (ส่วนกรองสถานะ)
- ไฟล์ที่แตะ: `frontend/src/pages/EquipmentList/EquipmentList.jsx`, `frontend/src/components/SearchBar.jsx`, `frontend/src/__tests__/AC-EQS-02.test.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-07
- เสร็จเมื่อ: `AC-EQS-02.test.jsx` ส่วนกรองสถานะผ่าน
- สถานะ: พร้อมทำ

#### จุดตรวจเฟส 2
- test ที่ต้องผ่านทั้งหมด: test ของเฟส 1 + TestAC_EQS_02_FilterStatus + AC-EQS-02.test.jsx
- คนลองเองตาม AC-EQS-02:
  Given อยู่ในหน้ารายการอุปกรณ์
  When กรองสถานะ "ว่าง"
  Then เห็นเฉพาะอุปกรณ์ที่ว่าง
- ยังขาด: การจับคู่คำค้น "Notebook" (รอ Q-06 ดู T-24), ตัวเลขพร้อม/ทั้งหมดเมื่อกรอง (รอ Q-P2 ดู T-30)

---

## เฟส 3: ค้นหาไม่พบ (AC-EQS-03)

### T-09 แสดงข้อความไม่พบและปุ่มล้างตัวกรอง
- รองรับ: FR-EQS-03
- ตรวจด้วย: AC-EQS-03
- ไฟล์ที่แตะ: `frontend/src/components/EmptyState.jsx`, `frontend/src/pages/EquipmentList/EquipmentList.jsx`, `frontend/src/__tests__/AC-EQS-03.test.jsx`, `backend/internal/equipment/handler_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-08
- เสร็จเมื่อ: `TestAC_EQS_03_EmptyIs200` ผ่าน (ไม่พบ = 200 items ว่าง ไม่ใช่ error) และ `AC-EQS-03.test.jsx` ผ่าน
- สถานะ: พร้อมทำ

#### จุดตรวจเฟส 3
- คนลองเองตาม AC-EQS-03:
  Given ค้นหาด้วยคำที่ไม่มีในระบบ
  When กดค้นหา
  Then เห็นข้อความไม่พบอุปกรณ์ และกดล้างตัวกรองแล้วกลับไปเห็นรายการทั้งหมด
- ยังขาด: ไม่มี

---

## เฟส 4: รายละเอียดอุปกรณ์ที่ถูกยืม (AC-EQS-04)

### T-10 สร้าง API รายละเอียดรายชิ้น พร้อม canRequestLoan และกำหนดคืน (วันที่และเวลา)
- รองรับ: FR-EQS-04, CON-DATA-02
- ตรวจด้วย: AC-EQS-04
- ไฟล์ที่แตะ: `backend/internal/equipment/repository.go`, `backend/internal/equipment/service.go`, `backend/internal/equipment/dto.go`, `backend/internal/equipment/handler_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-05
- เสร็จเมื่อ: `TestAC_EQS_04_BorrowedShowsDueDate` ผ่าน (ถูกยืม ยังไม่เลยกำหนด -> มี dueAt วันที่และเวลา canRequestLoan=false) และไม่มี endpoint ใดเขียนสถานะ (CON-DATA-02)
- สถานะ: พร้อมทำ

### T-11 สร้างหน้ารายละเอียดอุปกรณ์
- รองรับ: FR-EQS-04
- ตรวจด้วย: AC-EQS-04
- ไฟล์ที่แตะ: `frontend/src/pages/EquipmentDetail/EquipmentDetail.jsx`, `frontend/src/__tests__/AC-EQS-04.test.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-10
- เสร็จเมื่อ: `AC-EQS-04.test.jsx` ผ่าน (เห็น "ถูกยืม" กำหนดคืน และไม่มีปุ่มส่งคำขอยืม)
- สถานะ: พร้อมทำ

#### จุดตรวจเฟส 4
- คนลองเองตาม AC-EQS-04:
  Given ข้อมูลตั้งต้นมีอุปกรณ์สถานะ "ถูกยืม" ที่ยังไม่เลยกำหนดคืน (จำลองผลของ UC-08)
  When นักศึกษาเปิดดูอุปกรณ์ชิ้นนั้น
  Then เห็นสถานะ "ถูกยืม" กำหนดคืน (วันที่และเวลา) และไม่มีปุ่มส่งคำขอยืม
- ยังขาด: ไม่มี (กำหนดคืนอ่านจาก loan ที่ UC-08 บันทึก)

---

## เฟส 5: เลยกำหนดคืน (AC-EQS-14, CR-001, CR-002)

### T-12 คำนวณ isOverdue ด้วยวันที่และเวลา Asia/Bangkok และแยก DTO นักศึกษา/เจ้าหน้าที่
- รองรับ: FR-EQS-04, DOM-PRIV-01, DOM-LOAN-01, CON-DATA-01 (ป้ายไม่ใช่สถานะที่ 4)
- ตรวจด้วย: AC-EQS-14
- ไฟล์ที่แตะ: `backend/internal/config/config.go`, `backend/internal/equipment/service.go`, `backend/internal/equipment/dto.go`, `backend/internal/equipment/service_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-10
- เสร็จเมื่อ: `TestAC_EQS_14_OverdueStudentVsStaff` ผ่าน (กำหนดคืน 2569-10-10 16:00 ตั้งนาฬิกา 17:00 วันเดียวกัน: นักศึกษาได้ isOverdue=true ไม่มี dueAt, เจ้าหน้าที่ได้ทั้งคู่ สถานะยังเป็น borrowed / ตั้งนาฬิกา 15:59 -> isOverdue=false)
- สถานะ: พร้อมทำ

### T-13 แสดงป้าย "เลยกำหนดคืน"
- รองรับ: FR-EQS-04
- ตรวจด้วย: AC-EQS-14
- ไฟล์ที่แตะ: `frontend/src/components/StatusBadge.jsx`, `frontend/src/pages/EquipmentDetail/EquipmentDetail.jsx`, `frontend/src/__tests__/AC-EQS-14.test.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-12, T-11
- เสร็จเมื่อ: `AC-EQS-14.test.jsx` ผ่าน
- สถานะ: พร้อมทำ

#### จุดตรวจเฟส 5
- คนลองเองตาม AC-EQS-14:
  Given อุปกรณ์ "ถูกยืม" แบบปกติ กำหนดคืน 16:00 ของวันนี้ และตอนนี้เลย 16:00 แล้ว (ข้อมูลตั้งต้น)
  When นักศึกษาเปิดดู แล้วล็อกอินจำลองเป็นเจ้าหน้าที่เปิดดูชิ้นเดียวกัน
  Then นักศึกษาเห็น "ถูกยืม" + ป้าย "เลยกำหนดคืน" ไม่เห็นกำหนดคืน ไม่มีปุ่ม / เจ้าหน้าที่เห็นกำหนดคืนและป้าย
- ยังขาด: กำหนดคืนของยืมค้างคืน (Q-SH-04) ไม่กระทบโค้ดเพราะอ่าน due_at จาก loan

---

## เฟส 6: อุปกรณ์ไม่พร้อมใช้งาน (AC-EQS-06)

### T-14 ป้าย "ไม่พร้อมใช้งาน" และไม่มีปุ่มส่งคำขอยืม
- รองรับ: FR-EQS-05
- ตรวจด้วย: AC-EQS-06
- ไฟล์ที่แตะ: `frontend/src/components/StatusBadge.jsx`, `frontend/src/__tests__/AC-EQS-06.test.jsx`, `backend/internal/equipment/service_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-11
- เสร็จเมื่อ: `TestAC_EQS_06_UnavailableCannotRequest` และ `AC-EQS-06.test.jsx` ผ่าน
- สถานะ: พร้อมทำ

#### จุดตรวจเฟส 6
- คนลองเองตาม AC-EQS-06:
  Given อุปกรณ์มีสถานะ "ไม่พร้อมใช้งาน"
  When นักศึกษาเปิดดูรายละเอียด
  Then เห็นป้าย "ไม่พร้อมใช้งาน" และไม่มีปุ่มส่งคำขอยืม
- ยังขาด: ไม่มี

---

## เฟส 7: [ยกเลิกโดย CR-002] อุปกรณ์รออนุมัติ (AC-EQS-12)

### T-15 [ยกเลิกโดย CR-002] ไม่นับ "รออนุมัติ" ในจำนวนพร้อมให้ยืม
- เหตุผล: FR-EQS-09 และ AC-EQS-12 ถูกยกเลิก ไม่มีสถานะรออนุมัติใน CON-DATA-01 แล้ว (RQ-09)
- สถานะ: ยกเลิก

---

## เฟส 8: คืนแล้วกลับมายืมได้ (AC-EQS-05)

### T-16 ไม่ cache ข้อมูลสถานะ และแสดงปุ่มส่งคำขอยืมเมื่อว่าง
- รองรับ: FR-EQS-06, NFR-DATA-01
- ตรวจด้วย: AC-EQS-05
- ไฟล์ที่แตะ: `backend/internal/platform/http.go` (Cache-Control: no-store), `backend/internal/equipment/handler_test.go`, `frontend/src/pages/EquipmentDetail/EquipmentDetail.jsx`, `frontend/src/__tests__/AC-EQS-05.test.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-11
- เสร็จเมื่อ: `TestAC_EQS_05_ReturnedBecomesAvailable` ผ่าน (ปิด loan จำลอง UC-09 แล้วเรียกใหม่ได้ available, canRequestLoan=true, มี header no-store) และปุ่มลิงก์ไป `/loans/new?assetCode=...`
- สถานะ: พร้อมทำ (หน้า `/loans/new` เป็นของฟีเจอร์ 002 ฟีเจอร์นี้แค่ส่งรหัสครุภัณฑ์ไป)

#### จุดตรวจเฟส 8
- คนลองเองตาม AC-EQS-05:
  Given ปิด loan ของอุปกรณ์ชิ้นหนึ่งในฐานข้อมูล (จำลองการคืนสภาพปกติ)
  When นักศึกษารีเฟรชหน้ารายการภายใน 30 วินาที
  Then สถานะเป็น "ว่าง" และมีปุ่มส่งคำขอยืม
- ยังขาด: ไม่มี

---

## เฟส 9: เห็นข้อมูลตามสิทธิ์ (AC-EQS-07)

### T-17 เพิ่มชื่อผู้ยืมใน DTO เจ้าหน้าที่ และ API สรุปจำนวนตามสถานะ
- รองรับ: FR-EQS-07, DOM-PRIV-01
- ตรวจด้วย: AC-EQS-07
- ไฟล์ที่แตะ: `backend/internal/equipment/dto.go`, `backend/internal/equipment/handler.go`, `backend/internal/equipment/handler_test.go`, `frontend/src/pages/EquipmentDetail/EquipmentDetail.jsx`, `frontend/src/__tests__/AC-EQS-07.test.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-12
- เสร็จเมื่อ: `TestAC_EQS_07_StudentNoBorrower` ผ่าน (response ของนักศึกษาไม่มีคำว่า borrower เลย) และ `TestAC_EQS_07_StaffSeesBorrower` ผ่าน
- สถานะ: พร้อมทำ

#### จุดตรวจเฟส 9
- คนลองเองตาม AC-EQS-07:
  Given อุปกรณ์ถูกยืมโดยนักศึกษาคนอื่น
  When นักศึกษาเปิดดูรายละเอียด แล้วเจ้าหน้าที่เปิดดูชิ้นเดียวกัน
  Then นักศึกษาไม่เห็นชื่อผู้ยืม เจ้าหน้าที่เห็น
- ยังขาด: ไม่มี (อาจารย์ไม่ใช่ผู้ใช้ระบบ ปิด Q-10 โดย CR-002)

---

## เฟส 10: เฉพาะภาควิชาของตนเอง (AC-EQS-13)

### T-18 จำกัดทุก query ของนักศึกษาด้วยภาควิชา และตอบ 403 เมื่อขอข้ามภาควิชา
- รองรับ: FR-EQS-10, DOM-ACCESS-01, IF-IDP-01
- ตรวจด้วย: AC-EQS-13
- ไฟล์ที่แตะ: `backend/internal/equipment/repository.go`, `backend/internal/equipment/handler.go`, `backend/internal/equipment/handler_test.go`, `frontend/src/components/ErrorState.jsx`, `frontend/src/__tests__/AC-EQS-13.test.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-10
- เสร็จเมื่อ: `TestAC_EQS_13_OtherDepartment403` ผ่าน ทั้งจาก list และเรียกรายละเอียดตรง body ไม่มีข้อมูลอุปกรณ์
- สถานะ: พร้อมทำ (ขอบเขตของเจ้าหน้าที่รอ Q-P3 ดู T-28)

#### จุดตรวจเฟส 10
- คนลองเองตาม AC-EQS-13:
  Given ล็อกอินจำลองเป็นนักศึกษาภาควิชา A
  When เปิด URL รายละเอียดของอุปกรณ์ภาควิชา B โดยตรง
  Then เห็นข้อความไม่มีสิทธิ์ และไม่มีข้อมูลอุปกรณ์ใน response
- ยังขาด: ไม่มี

---

## เฟส 11: audit log ของการเปิดรายละเอียด (AC-EQS-11)

### T-19 บันทึก audit log เมื่อเปิดรายละเอียด
- รองรับ: DOM-AUDIT-01
- ตรวจด้วย: AC-EQS-11
- ไฟล์ที่แตะ: `backend/db/migrations/0002_audit_log.sql`, `backend/internal/audit/writer.go`, `backend/internal/audit/writer_test.go`, `backend/internal/equipment/handler.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-10
- เสร็จเมื่อ: `TestAC_EQS_11_DetailWritesAudit` ผ่าน (มีแถวที่มีผู้เข้าถึง เวลา รหัสครุภัณฑ์) และตารางเป็น append-only
- สถานะ: พร้อมทำ (การดูรายการ/ค้นหารอ Q-11 ดู T-29, ถ้าเขียน audit ไม่สำเร็จทำอย่างไรรอ Q-P4 ดู T-29)

#### จุดตรวจเฟส 11
- คนลองเองตาม AC-EQS-11:
  Given เปิดดูรายละเอียดอุปกรณ์ชิ้นหนึ่ง
  When การเข้าถึงเสร็จ
  Then ในตาราง audit_log มีแถวของผู้ใช้นั้น เวลา และรหัสครุภัณฑ์
- ยังขาด: audit ของรายการและการค้นหา (Q-11)

---

## เฟส 12: ฐานข้อมูลล่ม (AC-EQS-08)

### T-20 ตอบ 503 เมื่อต่อฐานข้อมูลไม่ได้ และหน้าจอแสดงปุ่มลองใหม่โดยไม่แสดงข้อมูลเก่า
- รองรับ: FR-EQS-08
- ตรวจด้วย: AC-EQS-08
- ไฟล์ที่แตะ: `backend/internal/platform/http.go`, `backend/internal/equipment/handler_test.go`, `frontend/src/components/ErrorState.jsx`, `frontend/src/pages/EquipmentList/EquipmentList.jsx`, `frontend/src/__tests__/AC-EQS-08.test.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-06
- เสร็จเมื่อ: `TestAC_EQS_08_DBDown503` ผ่าน (ปิดการเชื่อมต่อ -> 503 retryable) และ `AC-EQS-08.test.jsx` ผ่าน (รายการเดิมถูกล้างก่อนแสดง error)
- สถานะ: พร้อมทำ (กรณีต่อได้แต่ช้า ต้องรอกี่วินาทีรอ Q-08 ดู T-26)

#### จุดตรวจเฟส 12
- คนลองเองตาม AC-EQS-08:
  Given หยุด container ฐานข้อมูล (`docker compose stop db`)
  When เปิดหน้ารายการอุปกรณ์
  Then เห็นข้อความผิดพลาดพร้อมปุ่มลองใหม่ ไม่มีรายการเก่าค้าง
- ยังขาด: timeout (Q-08)

---

## เฟส 13: session หมดอายุ (AC-EQS-09)

### T-21 จัดการ 401 บนหน้าจอ: ไปหน้าล็อกอินจำลองพร้อมข้อความ แล้วกลับหน้าเดิม
- รองรับ: IF-IDP-01, NFR-SEC-01
- ตรวจด้วย: AC-EQS-09
- ไฟล์ที่แตะ: `frontend/src/api/client.js`, `frontend/src/pages/Login/MockLogin.jsx`, `frontend/src/__tests__/AC-EQS-09.test.jsx`, `backend/internal/auth/middleware_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-03, T-06
- เสร็จเมื่อ: `AC-EQS-09.test.jsx` ผ่าน (401 -> ไปหน้าล็อกอินพร้อมข้อความ -> กลับ path เดิม) และ returnTo ภายนอกระบบถูกปฏิเสธ
- สถานะ: พร้อมทำ (หน้าล็อกอินเป็นแบบจำลอง UC-13 อยู่นอกขอบเขต)

#### จุดตรวจเฟส 13
- คนลองเองตาม AC-EQS-09:
  Given ลบ token ใน browser (จำลอง session หมดอายุ)
  When เรียกดูรายการอุปกรณ์
  Then ถูกส่งไปหน้าล็อกอินพร้อมข้อความ และหลังล็อกอินกลับมาหน้าเดิม
- ยังขาด: auth จริง (Q-SH-03)

---

## เฟส 14: ประสิทธิภาพ (AC-EQS-10)

### T-22 ทดสอบโหลด 1,000 รายการ ผู้ใช้พร้อมกัน 50 คน
- รองรับ: NFR-PERF-01
- ตรวจด้วย: AC-EQS-10
- ไฟล์ที่แตะ: `backend/db/seed/seed_1000.sql`, `tests/load/search.js` (k6 ผ่าน docker image `grafana/k6`), `backend/db/migrations/0003_search_indexes.sql`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-07
- เสร็จเมื่อ: รายงาน k6 แสดง p95 <= 3 วินาที บันทึกผลลง `tests/load/RESULT.md`
- สถานะ: พร้อมทำ (วัดบนเครื่อง Codespace ไม่ใช่ server จริง ต้องเขียนข้อจำกัดนี้ในผล)

#### จุดตรวจเฟส 14
- คนลองเองตาม AC-EQS-10:
  Given seed 1,000 รายการ
  When รัน k6 50 VU ค้นหาอุปกรณ์
  Then p95 <= 3 วินาที
- ยังขาด: ไม่มี

---

## เฟส 15: รอคำตอบ

### T-23 แบ่งหน้ารายการรุ่น
- รองรับ: FR-EQS-01
- ตรวจด้วย: AC-EQS-01
- ไฟล์ที่แตะ: `backend/internal/config/config.go`, `backend/internal/equipment/repository.go`, `frontend/src/components/Pagination.jsx`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-06
- เสร็จเมื่อ: `TestAC_EQS_01_Pagination` ผ่าน ด้วยจำนวนต่อหน้าที่ได้จากคำตอบ
- สถานะ: รอ Q-05

### T-24 จับคู่คำค้นกับชื่อรุ่นและรหัสครุภัณฑ์
- รองรับ: FR-EQS-02
- ตรวจด้วย: AC-EQS-02
- ไฟล์ที่แตะ: `backend/internal/equipment/repository.go`, `backend/internal/equipment/handler_test.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-07
- เสร็จเมื่อ: `TestAC_EQS_02_SearchNotebook` ผ่าน
- สถานะ: รอ Q-06

### T-25 [ยกเลิกโดย CR-002] กำหนดสีของป้ายสถานะ
- เหตุผล: Q-07 ปิดแล้ว เหลือป้ายเดียว "ไม่พร้อมใช้งาน"
- สถานะ: ยกเลิก

### T-26 ตั้ง timeout ของ query และ request
- รองรับ: FR-EQS-08
- ตรวจด้วย: AC-EQS-08
- ไฟล์ที่แตะ: `backend/internal/config/config.go`, `frontend/src/config.js`, `frontend/src/api/client.js`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-20
- เสร็จเมื่อ: `TestAC_EQS_08_SlowDBTimeout` ผ่าน ด้วยค่าที่ได้จากคำตอบ
- สถานะ: รอ Q-08

### T-27 ตรวจหน้าจอที่ความกว้างมือถือ
- รองรับ: NFR-USE-01
- ตรวจด้วย: ไม่มี AC ควรเสนอทีมเพิ่ม
- ไฟล์ที่แตะ: `frontend/src/index.css`, หน้าจอทุกหน้าของฟีเจอร์นี้
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-13
- เสร็จเมื่อ: หน้าจอไม่ล้นแนวนอนที่ความกว้างที่ได้จากคำตอบ
- สถานะ: รอ Q-09

### T-28 ขอบเขตภาควิชาของเจ้าหน้าที่
- รองรับ: FR-EQS-07, DOM-ACCESS-01
- ตรวจด้วย: AC-EQS-07
- ไฟล์ที่แตะ: `backend/internal/auth/principal.go` (`scopeFor`), `backend/internal/equipment/dto.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-17, T-18
- เสร็จเมื่อ: test ตามคำตอบผ่าน
- สถานะ: รอ Q-P3

### T-29 audit ของการดูรายการและการค้นหา และกรณีเขียน audit ไม่สำเร็จ
- รองรับ: DOM-AUDIT-01
- ตรวจด้วย: AC-EQS-11
- ไฟล์ที่แตะ: `backend/internal/audit/writer.go`, `backend/internal/equipment/handler.go`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-19
- เสร็จเมื่อ: test ตามคำตอบผ่าน
- สถานะ: รอ Q-11, Q-P4

### T-30 ตัวเลขว่าง/ทั้งหมดเมื่อกรอง และเชื่อม auth service จริง
- รองรับ: FR-EQS-01, IF-IDP-01
- ตรวจด้วย: AC-EQS-02, AC-EQS-09
- ไฟล์ที่แตะ: `backend/internal/equipment/repository.go`, `backend/internal/auth/`
- แบบหน้าจอ: ไม่มี
- ต้องทำหลัง: T-07, T-21
- เสร็จเมื่อ: test ตามคำตอบผ่าน
- สถานะ: รอ Q-P2, Q-SH-03

---

## ตารางตรวจความครบ

### 1. AC -> task
| AC ID | task ที่ตรวจ AC นี้ |
|---|---|
| AC-EQS-01 | T-04, T-05, T-06, T-23 |
| AC-EQS-02 | T-07, T-08, T-24, T-30 |
| AC-EQS-03 | T-09 |
| AC-EQS-04 | T-10, T-11 |
| AC-EQS-05 | T-16 |
| AC-EQS-06 | T-14 |
| AC-EQS-07 | T-17, T-28 |
| AC-EQS-08 | T-20, T-26 |
| AC-EQS-09 | T-21, T-30 |
| AC-EQS-10 | T-22 |
| AC-EQS-11 | T-19, T-29 |
| AC-EQS-12 | (ยกเลิกโดย CR-002) |
| AC-EQS-13 | T-18 |
| AC-EQS-14 | T-12, T-13 |

### 2. Constraint -> task (รวมที่ใช้ร่วมจาก specs/000-shared/constraints.md)
| Constraint ID | task ที่ทำให้เป็นจริง |
|---|---|
| CON-TECH-01 | T-01, T-02 |
| CON-TECH-02 | T-01 (Docker), ส่วน HTTPS/TLS ที่ reverse proxy: ยังไม่มี task ดูหมายเหตุ |
| CON-DATA-01 | T-02, T-12, T-14 |
| DOM-LOAN-01 | T-12 |
| CON-DATA-02 | T-10 |
| CON-DATA-03 | T-02, T-05 |
| DOM-PRIV-01 | T-12, T-17 |
| DOM-AUDIT-01 | T-19, T-29 |
| DOM-ACCESS-01 | T-18, T-28 |
| IF-IDP-01 | T-03, T-18, T-21, T-30 |

หมายเหตุ CON-TECH-02 และ NFR-SEC-01 (TLS 1.2): การตั้ง reverse proxy ด้วย HTTPS เป็นงานของทั้งระบบ ไม่ใช่ของฟีเจอร์นี้คนเดียว
เสนอให้ทีมตัดสินว่าใครทำ (ของกลาง) ก่อนส่งงาน

### 3. FR -> AC -> เฟส
| FR ID | AC ที่ตรวจ FR นี้ | เฟส |
|---|---|---|
| FR-EQS-01 | AC-EQS-01 | 1, 15 |
| FR-EQS-02 | AC-EQS-02 | 2, 15 |
| FR-EQS-03 | AC-EQS-03 | 3 |
| FR-EQS-04 | AC-EQS-04, AC-EQS-14 | 4, 5 |
| FR-EQS-05 | AC-EQS-06 | 6 |
| FR-EQS-06 | AC-EQS-05 | 8 |
| FR-EQS-07 | AC-EQS-07 | 9 |
| FR-EQS-08 | AC-EQS-08 | 12, 15 |
| FR-EQS-09 | (ยกเลิกโดย CR-002) | - |
| FR-EQS-10 | AC-EQS-13 | 10 |

NFR ที่ไม่มี AC: NFR-SEC-01 (บางส่วนตรวจผ่าน AC-EQS-09), NFR-AVAIL-01, NFR-USE-01 -> ไม่มี AC ควรเสนอทีมเพิ่ม
NFR-USE-01 ต้องทดสอบกับผู้ใช้จริง 10 คน เป็นงานของคน ไม่ใช่ task เขียนโค้ด

## สิ่งที่ยังไม่ทำ (Open Questions จาก spec และของกลาง)
| คำถาม | task ที่รอ |
|---|---|
| Q-04 ประเภทอุปกรณ์ตายตัวหรือเพิ่มได้ | ไม่มี (ใช้ข้อมูลตั้งต้น ตาม backlog) |
| Q-05 แบ่งหน้ากี่รายการ | T-23 |
| Q-06 วิธีจับคู่คำค้น | T-24 |
| Q-07 [ปิด CR-002] | - |
| Q-08 timeout | T-26 |
| Q-09 ความกว้างจอมือถือ | T-27 |
| Q-10 [ปิด CR-002] | - |
| Q-11 audit ของรายการ/ค้นหา | T-29 |
| Q-12 [ปิด CR-002] | - |
| Q-SH-01 อุปกรณ์ที่มีคำขอค้างนับเป็นว่างไหม | T-04 (ถ้าคำตอบคือไม่นับ ต้องแก้ T-04) |
| Q-SH-04 กำหนดคืนของยืมค้างคืน | ไม่มี (กระทบฟีเจอร์ 005) |
| Q-P2 ตัวเลขเมื่อกรอง | T-30 |
| Q-P3 ขอบเขตภาควิชาของเจ้าหน้าที่ | T-28 |
| Q-P4 เขียน audit ไม่สำเร็จ | T-29 |
| Q-SH-03 โปรโตคอล auth | T-30 |
