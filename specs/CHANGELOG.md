# Changelog: Specs
บันทึกการเปลี่ยนแปลงของ spec ทุกครั้ง เรียงจากใหม่ไปเก่า
การเปลี่ยนแปลงหลัง Draft v2 ต้องอ้างอิง change request ใน `specs/changes/`

## [SPEC-EQS-001 Draft v4 + shared v2 + backlog v4] - 2569-11-02
Change request: [CR-002](changes/CR-002.md) ให้ repo ตรงกับ Miro (รับทั้งหมด)

### Changed
- CON-DATA-01: สถานะอุปกรณ์ 6 ค่า -> 3 ค่า ว่าง / ถูกยืม / ไม่พร้อมใช้งาน (RQ-09)
- IF-IDP-01: ระบบยืนยันตัวตนของมหาวิทยาลัย (RQ-23) ไม่ใช่ UC-13
- FR-EQS-01, 02, 06, AC-EQS-01, 02, 05: "พร้อมให้ยืม" -> "ว่าง"
- FR-EQS-04, AC-EQS-04, AC-EQS-14: กำหนดคืนเป็นวันที่และเวลา เลยกำหนดเมื่อเวลาปัจจุบัน > กำหนดคืน (RQ-17)
- FR-EQS-05, AC-EQS-06: ป้ายเดียว "ไม่พร้อมใช้งาน"
- NFR-PERF-01, AC-EQS-10: p95 <= 3 วินาที (RQ-24)
- Scope: บทบาทเหลือ นักศึกษา / เจ้าหน้าที่, CON-DATA-02 อ้าง use case ตาม Use Case Diagram
- Q-SH-01 เปลี่ยนคำถาม, ปิด Q-SH-02, Q-07, Q-10, Q-12
- backlog: BL-04, 05, 06, 07, 08, 09, 10 อ้าง use case ตาม Miro
- plan v1.3, tasks v1 (แก้ตาม CR-002), SRS v0.2 (เขียนใหม่ตาม RQ-01 ถึง RQ-27)

### Removed
- FR-EQS-09 และ AC-EQS-12 (สถานะ "รออนุมัติ") ID ไม่นำกลับมาใช้, task T-15 และ T-25

### Added
- DOM-LOAN-01 (RQ-17), IF-NOT-01 (ระบบแจ้งเตือน Email), Q-SH-04 (กำหนดคืนของยืมค้างคืน)
- backlog BL-12 ถึง BL-23 (use case และ RQ จาก Miro ที่ยังไม่มี) BL-20 ยกเลิกคำขอยืม = Must ที่เหลือยังไม่จัดลำดับ
- specs/changes/CR-002.md

## [shared constraints v1 + SPEC-EQS-001 Draft v3.1] - 2569-11-02
เตรียมเปิดฟีเจอร์ 002-006 ตาม docs/shared-spec-guide.md ข้อ 5 ไม่มี CR เพราะข้อความและ ID ไม่เปลี่ยน

### Added
- specs/000-shared/constraints.md: CON-TECH-01, CON-TECH-02, CON-DATA-01, CON-DATA-03, DOM-PRIV-01,
  DOM-AUDIT-01, DOM-ACCESS-01, IF-IDP-01 (ย้ายจาก 001 ข้อความเดิม) และคำถามที่กระทบหลายฟีเจอร์ Q-SH-01 ถึง Q-SH-03
- specs/000-shared/data-model.md, glossary.md

### Changed
- SPEC-EQS-001: หมวด Constraints เปลี่ยนข้อที่ย้ายเป็นบรรทัด "ใช้ร่วม" เหลือ CON-DATA-02 เป็นข้อเฉพาะฟีเจอร์
- SPEC-EQS-001: ตรวจคำกำกวมตามที่อาจารย์สั่ง เพิ่ม Q-05 ถึง Q-12 (แบ่งหน้า, การค้นหา, "ชัดเจน", timeout,
  "หน้าจอมือถือ", สิทธิ์อาจารย์, ขอบเขต audit, ที่มาของกำหนดคืน) ยังไม่แก้ FR/AC จนกว่าจะได้คำตอบ
- plan.md ของ 001 Draft v1.2: โครงโฟลเดอร์เป็น backend/ frontend/ ตาม template ของรายวิชา

### Fixed
- ลิงก์ backlog ใน spec.md, CR-001.md และ CHANGELOG ชี้ไป specs/backlog.md (เดิมชี้ specs/changes/backlog.md ที่ไม่มีอยู่)
- backlog.md: รหัสจองล่วงหน้าใช้ BL-11 ทั้งไฟล์ (เดิมปน BL-12), แก้ตัวเลขกฎ 60% เป็น 22/24 = 91.7%

## [SPEC-EQS-001 Draft v3] - 2569-10-04
UC-01 ตรวจสอบสถานะอุปกรณ์ | Change request: [CR-001](changes/CR-001.md) (รับบางส่วน 2569-10-03) | การเจรจา: N-01 ใน [backlog](backlog.md) (2569-10-02)

### Changed
- FR-EQS-04: แสดงกำหนดคืนระดับวันที่เฉพาะเมื่อยังไม่เลยกำหนด ถ้าเลยแล้วนักศึกษาเห็นป้าย "เลยกำหนดคืน" แทนวันที่
  เจ้าหน้าที่เห็นทั้งวันที่และป้าย ป้ายนี้ไม่ใช่สถานะใหม่
- DOM-PRIV-01: เพิ่มป้าย "เลยกำหนดคืน" ในสิ่งที่นักศึกษาเห็นได้
- AC-EQS-04: ระบุว่ากำหนดคืนแสดงระดับวันที่
- Q-01: ปิดหลังยืนยันกับเจ้าหน้าที่ภาควิชา
- Q-02: ปิดตาม N-01 จองล่วงหน้า (BL-11) เป็น Won't เทอมนี้
- Scope: In scope เพิ่มป้าย "เลยกำหนดคืน" ในรายละเอียดรายชิ้น / Out of scope จองล่วงหน้าอ้าง N-01 แทน Q-02
- Traceability: FR-EQS-04 อ้างอิง CR-001 และ AC-EQS-14

### Added
- AC-EQS-14: กรณีอุปกรณ์เลยกำหนดคืน (มุมมองนักศึกษาเทียบกับเจ้าหน้าที่)
- Clarifications: Session 2569-10-02 (N-01), Session 2569-10-03 (CR-001)
- หัวเอกสาร: อ้างอิง Backlog BL-01 ถึง BL-04

### Related
- plan.md Draft v1.1: response รายชิ้นเพิ่ม `isOverdue`, test plan เพิ่ม AC-EQS-14, M3 เพิ่ม AC-14

## [SPEC-EQS-001 Draft v2] - 2569-09-20
- ปิด Q-01, Q-03; เพิ่ม FR-EQS-09 (รออนุมัติ), FR-EQS-10 / DOM-ACCESS-01 (เฉพาะภาควิชาตนเอง)
- FR-EQS-01: 1 แถว = 1 รุ่น/ชนิด แล้วเจาะลงรายชิ้นตามรหัสครุภัณฑ์

## [SPEC-EQS-001 Draft v1]
- ฉบับแรกของ UC-01
