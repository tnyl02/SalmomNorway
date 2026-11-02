# SalmomNorway: ระบบยืม-คืนอุปกรณ์ภาควิชา

Term Project รายวิชา 520461-165 Software Requirement Specification and Management (1/2569)
ภาควิชาคอมพิวเตอร์ คณะวิทยาศาสตร์ มหาวิทยาลัยศิลปากร

## ทีม

- ชื่อทีม:
- เครื่องมือ AI ที่ใช้: (Copilot ใน Codespaces / Claude Code / Cursor)

| สมาชิก | GitHub | ฟีเจอร์ที่รับผิดชอบ | ส่วนอื่นที่ทำ |
|---|---|---|---|
| | | 001-equipment-status | |
| | | 002-borrow-request | |
| | | 003-cancel-request | |
| | | 004-approve-request | |
| | | 005-handover | |
| | | 006-return | |

ดูว่าใครทำอะไรได้จาก 3 ที่: ตารางนี้, ช่อง `Owner:` ใน spec.md ของแต่ละฟีเจอร์, และ `git log --author=<ชื่อ>`

## สิ่งที่ส่ง

| งาน | อยู่ที่ |
|---|---|
| SRS | `docs/srs/SRS.md` (diagram อยู่ใน Miro ลิงก์ในภาคผนวกของ SRS) |
| spec.md | `specs/00N-<feature>/spec.md` ทั้ง 6 ฟีเจอร์ |
| Prototype | `backend/` (Go) และ `frontend/` (React) |
| Traceability Matrix | `specs/00N-<feature>/rtm.md` (ทำหลังทุกฟีเจอร์เสร็จ) |

## โครงของ repo

```
README.md                 ไฟล์นี้
AGENTS.md                 กติกาที่ AI ต้องทำตาม (รวมตารางเทียบ Go กับคำสั่งของรายวิชา)
CLAUDE.md                 ชี้ไป AGENTS.md (สำหรับ Claude Code)
prompt-log.md             บันทึกการสั่ง AI ทุกครั้ง พร้อมชื่อคนสั่ง
docs/srs/SRS.md           SRS ฉบับเต็ม
specs/README.md           ดัชนีฟีเจอร์และผู้รับผิดชอบ
specs/000-shared/         Constraint ข้อมูล และคำศัพท์ที่ทุกฟีเจอร์ใช้ร่วมกัน
specs/00N-<feature>/      spec.md plan.md tasks.md test-cases.md rtm.md ของแต่ละฟีเจอร์
specs/backlog.md          backlog จัดลำดับแล้ว และบันทึกการเจรจา N-01
specs/changes/            change request (CR-001)
specs/CHANGELOG.md        บันทึกว่า spec แต่ละเวอร์ชันเปลี่ยนอะไร
backend/                  หลังบ้าน Go + PostgreSQL
frontend/                 หน้าจอ React + Vite
.claude/commands/         คำสั่ง /clarify /plan /tasks /implement /testcases /verify (Claude Code)
.github/prompts/          คำสั่งชุดเดียวกัน (Copilot)
.cursor/commands/         คำสั่งชุดเดียวกัน (Cursor)
```
