# กติกาของโปรเจกต์นี้ (อ่านก่อนทำงานทุกครั้ง)

โปรเจกต์นี้เป็นงานในรายวิชา 520461-165 Software Requirement Specification and Management
ระบบ: ยืม-คืนอุปกรณ์ภาควิชา (SalmomNorway)
เราทำงานแบบ Spec-Driven Development: **spec.md คือความจริงหนึ่งเดียว** โค้ดและแผนงานต้องมาจาก spec เท่านั้น

## ไฟล์สำคัญ

- `specs/000-shared/constraints.md` = Constraint (CON / DOM / IF) ที่ใช้มากกว่า 1 ฟีเจอร์ เขียน**ที่เดียว** spec ของฟีเจอร์อ้าง ID ในบรรทัด "ใช้ร่วม" และนับเป็น Constraint ของฟีเจอร์นั้นทุกข้อ ส่วน `data-model.md` และ `glossary.md` คือข้อมูลและคำศัพท์ที่ทุกฟีเจอร์ใช้ร่วมกัน
- `specs/NNN-<feature>/spec.md` = ข้อกำหนดของฟีเจอร์ 1 ฟีเจอร์ (คนเขียน คนตรวจ) Constraint เฉพาะฟีเจอร์เขียนที่นี่ Constraint ที่ใช้ร่วมอ้าง ID จากไฟล์ร่วม
- `specs/NNN-<feature>/plan.md` = แผนทางเทคนิคของฟีเจอร์นั้น (AI ร่าง คนตรวจ)
- `specs/NNN-<feature>/tasks.md` = งานย่อยเรียงลำดับ (AI ร่างด้วย /tasks คนตรวจ)
- `specs/NNN-<feature>/test-cases.md` = test cases ที่แปลงจาก AC (คนออกแบบ AI เขียนโค้ด test ตาม) **test ที่มาจากตารางนี้ (ชื่อขึ้นต้น `TestTC_`) ห้ามแก้ให้ผ่าน** ถ้าไม่ผ่านให้แก้โค้ด หรือถามทีม
- `specs/NNN-<feature>/rtm.md` = ตารางตามรอย spec โค้ด test และข้อค้นพบ (AI สร้างด้วย /verify ทีมเขียนช่อง "ทีมตัดสิน" เอง)
- `specs/NNN-<feature>/mockups/` = แบบหน้าจอที่ตกลงกับผู้ใช้แล้ว (ถ้ามี) **ถ้า mockup กับ spec ขัดกัน ให้ถือ spec เป็นหลัก แล้วถามทีม**
- `docs/srs/SRS.md` = SRS ฉบับเต็มสำหรับคนอ่าน ใช้อ้างอิงเมื่อ spec.md ไม่ชัด (diagram อยู่ใน Miro ของทีม)
- `prompt-log.md` = บันทึกทุกคำสั่งที่ใช้กับ AI คำถามที่ AI ถาม และคำตอบของทีม **เพิ่มต่อท้ายเท่านั้น ห้ามแก้หรือลบบันทึกเดิม** ทุกบันทึกต้องระบุว่า**ใครในทีม**เป็นคนสั่ง และทำในฟีเจอร์ไหน

## กฎ 9 ข้อ

1. ก่อนทำอะไรก็ตาม ให้อ่าน `specs/*/spec.md` ของฟีเจอร์ที่เกี่ยวข้องก่อนเสมอ
2. **ห้ามเดา** ถ้า spec กำกวมหรือขาดข้อมูล ให้ถามกลับ หรือเขียนไว้ในหัวข้อ Open Questions ห้ามตัดสินใจแทนทีม
3. **ห้ามเพิ่มความต้องการ** ที่ไม่มีใน spec แม้จะ "ระบบแบบนี้มักมี" ก็ตาม
4. **ห้ามสร้างสิ่งที่อยู่ใน Out of scope**
5. **ห้ามเปลี่ยนหรือสร้าง ID ใหม่** ทุก FR / NFR / CON / IF / DOM / AC / UI ต้องใช้ ID เดิมจาก spec ถ้าจำเป็นต้องมีข้อใหม่ ให้เสนอเป็นคำถาม ไม่ใช่เขียนลงไปเอง
6. ระบุเทคโนโลยี (ภาษา ฐานข้อมูล framework) ได้เฉพาะที่อยู่ในหัวข้อ Constraints หรือใน `plan.md` ถ้า spec ไม่ระบุ ให้เขียนกำกับว่า "ทีมเลือกเอง ไม่ได้มาจาก spec"
7. ทุกอย่างที่สร้างต้องอ้างกลับไปหา ID ใน spec ได้ เช่น ฟังก์ชันมีคอมเมนต์ระบุว่ารองรับ FR ข้อใด test ตั้งชื่อตาม AC ID
8. **หน้าจอทำตาม mockup เฉพาะส่วนที่ spec บอกว่า "ต้องตรง"** ห้ามคัดลอกไฟล์ mockup ทั้งไฟล์มาเป็นโค้ด และถ้า mockup มีปุ่มหรือข้อความที่ไม่มี FR รองรับ ห้ามสร้าง ให้ถามทีม
9. **ตัวเลขและกฎที่มาจาก spec อยู่ที่เดียว** คือ `backend/internal/config/config.go` (และ `frontend/src/config.js` ถ้าหน้าจอใช้) พร้อมคอมเมนต์อ้าง ID ห้ามฝังตัวเลข เช่น 30 วินาที 20 รายการต่อหน้า กระจายในโค้ด

## ทีมนี้ใช้ Go ไม่ใช่ Python (อ่านแทนค่าในไฟล์คำสั่ง)

ไฟล์คำสั่งใน `.claude/commands/`, `.github/prompts/`, `.cursor/commands/` มาจาก template ของรายวิชาที่เขียนไว้สำหรับ Python
ทีมเลือกใช้ Go สำหรับหลังบ้าน (ดู `plan.md` ข้อ 2 ของแต่ละฟีเจอร์) **เมื่อคำสั่งพูดถึงสิ่งในคอลัมน์ซ้าย ให้ใช้คอลัมน์ขวาแทน**

| ในไฟล์คำสั่ง (Python) | ใน repo นี้ (Go) |
|---|---|
| `backend/app/` | `backend/internal/` และ `backend/cmd/server/` |
| `backend/app/main.py` | `backend/cmd/server/main.go` |
| `backend/app/config.py` | `backend/internal/config/config.go` |
| `backend/tests/` | ไฟล์ `*_test.go` ข้างโค้ดที่ทดสอบ (เช่น `backend/internal/equipment/handler_test.go`) |
| `cd backend && pytest -v` | `cd backend && go test ./... -v` |
| ชื่อ test `test_AC_EQS_01` | `TestAC_EQS_01` |
| ชื่อ test `test_TC_...` | `TestTC_...` (ห้ามแก้ให้ผ่าน เหมือนเดิม) |
| `requirements.txt` | `backend/go.mod` |

ฝั่งหน้าจอเหมือน template ทุกอย่าง (`frontend/`, React + Vite, `npm test` ด้วย vitest)

## ใครทำอะไร (สำคัญสำหรับการส่งงาน)

- 1 ฟีเจอร์ = 1 โฟลเดอร์ `specs/NNN-<feature>/` = 1 เจ้าของ ดูตารางใน `specs/README.md`
- โค้ดของแต่ละฟีเจอร์อยู่ใน package ของตัวเอง เช่น `backend/internal/<feature>/` และ `frontend/src/pages/<Feature>/` ห้ามแก้ไฟล์ของฟีเจอร์อื่นโดยไม่บอกเจ้าของ
- ของกลาง (`backend/internal/platform/`, `auth/`, `audit/`, `db/migrations/`) แก้ได้เมื่อจำเป็น แต่ต้องเขียนใน prompt-log ว่าแก้เพราะฟีเจอร์ไหน
- commit message ขึ้นต้นด้วยเลขฟีเจอร์ เช่น `002: T-03 done`

## ภาษา

ตอบและเขียนเอกสารเป็นภาษาไทย ยกเว้นชื่อไฟล์ โค้ด คำสั่ง และ ID
