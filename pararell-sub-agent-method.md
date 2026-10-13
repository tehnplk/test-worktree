# วิธีใช้ Sub-Agent หลายตัวพร้อมกัน (Parallel + Worktree)

ลำดับขั้นตอนตั้งแต่เตรียม repo จนรวมงานและเก็บกวาด

## 1. เตรียม repo (ทำครั้งเดียว)

1. สร้าง git repo
   ```bash
   git init
   ```
2. สร้าง commit แรก ถ้ายังไม่มี commit เลย worktree จะสร้างไม่ได้
   ```bash
   git commit --allow-empty -m "Initial commit"
   ```
3. สร้าง `.gitignore` แล้ว **commit ก่อนส่ง agent ออกไปทำงาน** ไม่อย่างนั้น worktree จะไม่รู้จักไฟล์นี้ ไฟล์ขยะจากการทดสอบจะค้างอยู่ และต้องลบ worktree ด้วย `--force`
   ```
   .playwright-cli/
   .claude/worktrees/
   ```

## 2. เขียน agent definition

ไฟล์ `.claude/agents/developer.md`

```markdown
---
name: developer
description: Develop independent features
model: sonnet
isolation: worktree
---

Implement the assigned feature.
Run relevant tests.
Commit your changes to the worktree branch before reporting.
Report changes and branch name.
Do not merge automatically.
```

- `isolation: worktree` ทำให้ agent ทุกตัวได้ worktree และ branch ของตัวเอง แก้ไฟล์แล้วไม่ทับกัน
- ต้องสั่งให้ **commit** ไว้ ไม่อย่างนั้นงานจะค้างเป็นไฟล์ที่ยังไม่ commit และ branch จะว่าง
- ถ้าแก้ไฟล์ agent ระหว่าง session อาจต้องเปิด session ใหม่ก่อน ระบบถึงจะโหลดค่าใหม่

## 3. แบ่งงาน

- **1 agent ต่อ 1 ไฟล์หรือ 1 module** ถ้าแตะไฟล์ไม่ซ้ำกัน ตอน merge จะไม่มี conflict
- **ใส่รายละเอียดให้ครบในคำสั่ง** agent ไม่เห็นบทสนทนาหลัก ต้องบอกชื่อไฟล์ ข้อมูล library และ version
- **ถ้างานต้องออกมาหน้าตาเข้าชุดกัน ให้ส่งค่ากลางชุดเดียวกันให้ทุกตัว** เช่น design tokens หรือข้อมูลตัวอย่าง และต้องกำหนดครบทุกโหมดที่หน้าเว็บมี เช่นสี dark mode ด้วย ไม่อย่างนั้นแต่ละตัวจะเลือกค่าเอง
- **บอกสิ่งที่ห้ามทำ** เช่น "ไม่เปลี่ยน behavior", "ไม่เพิ่มฟีเจอร์", "ห้าม merge"

## 4. กันไม่ให้ agent ชนกันนอก repo

worktree แยกเฉพาะไฟล์ใน repo ส่วนสิ่งเหล่านี้ยังใช้ร่วมกัน:

- **scratchpad:** ให้แต่ละตัวตั้งชื่อไฟล์ขึ้นต้นต่างกัน เช่น `flat-table-`
- **Playwright:** ให้แต่ละตัวใช้ session ของตัวเอง เช่น `playwright-cli -s=<ชื่อ> open <url>` และปิดเมื่อเสร็จ
- **HTTP server:** `playwright-cli` เปิด `file://` ไม่ได้ ให้รัน server กลางไว้ตัวเดียวก่อน
  ```bash
  python -m http.server 8765 --bind 127.0.0.1 --directory .claude/worktrees
  ```
  agent แต่ละตัวเปิดไฟล์ของตัวเองได้ที่ `http://127.0.0.1:8765/<worktree-folder>/<file>`

## 5. ส่ง agent ออกไปพร้อมกัน

- เรียก Agent ทุกตัวใน **message เดียว** ทุกตัวจะทำงานพร้อมกันใน background
- ไม่ต้องคอยเช็คสถานะ แต่ละตัวทำเสร็จระบบจะแจ้งเอง

## 6. ตรวจรายงานของแต่ละตัว

- ดูว่ารายงานบอกครบไหม: branch, commit, สิ่งที่แก้, สิ่งที่ตรวจแล้ว และสิ่งที่**ยังไม่ได้**ตรวจ
- สุ่มตรวจตัวเลขหรือผลลัพธ์ด้วยตัวเอง เช่นยอดรวม
- ระวัง agent ที่**เลี่ยงกฎที่ถูกบล็อก** เช่นโดนบล็อก `eval` แล้วไปใช้ `run-code` แทน ต้องแจ้งผู้ใช้ และตัดสินใจว่าจะบล็อกให้ครบหรือจะปลดบล็อก
- เช็คว่ามี commit ครบทุก branch
  ```bash
  git log --oneline --all
  ```

## 7. ดูผลจริงก่อน merge (ถ้าเป็นงาน UI)

```bash
playwright-cli open --headed http://127.0.0.1:8765/<worktree>/<file>
playwright-cli tab-new http://127.0.0.1:8765/<worktree>/<file2>
```

เช็คว่าไม่มี console error, กราฟและข้อมูลแสดงครบ, การโต้ตอบใช้ได้ และไม่มี scroll แนวนอนที่ความกว้าง 375px

## 8. Merge

merge ทุก branch พร้อมกันในคำสั่งเดียว (octopus merge) ถ้าแต่ละ branch แตะไฟล์ไม่ซ้ำกัน จะไม่มี conflict

```bash
git merge <branch1> <branch2> <branch3> -m "Merge ..."
```

## 9. เก็บกวาด

1. ลบ worktree ทีละอัน ห้ามลบโฟลเดอร์ตรงๆ เพราะ git จะยังจำ worktree นั้นอยู่
   ```bash
   git worktree remove .claude/worktrees/<agent-folder>
   ```
   ถ้าขึ้น "contains modified or untracked files" ให้ดู `git status` ใน worktree นั้นก่อน ถ้าเป็นแค่ไฟล์ขยะค่อยใช้ `--force`
2. ลบ branch ด้วย `-d` คำสั่งนี้จะยอมลบเฉพาะ branch ที่ merge แล้ว
   ```bash
   git branch -d <branch1> <branch2> ...
   ```
3. เช็คว่าเหลือแค่ repo หลัก
   ```bash
   git worktree list
   ```
4. ปิด browser ของ Playwright และ HTTP server

## แก้งานรอบถัดไป

ส่ง agent **ตัวใหม่** ไปแก้ agent ใหม่จะได้ worktree ใหม่ที่แตกจาก `master` ซึ่งมีงานล่าสุดอยู่แล้ว จากนั้นวนทำข้อ 3–9 อีกรอบ การลบ worktree เก่าไม่กระทบงานรอบใหม่ แต่ห้ามลบ worktree ที่ agent ยังทำงานอยู่
