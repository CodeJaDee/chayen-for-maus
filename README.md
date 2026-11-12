# chayen-for-maus

ชุด skill สำหรับทีม bot ใน [OpenMausBot](https://github.com/milind-soni/OpenMausBot) ที่รัน SDLC แบบอัตโนมัติเต็มรูปแบบ ตามแนวทาง [The AI-native SDLC playbook](https://claude.com/resources/articles/the-ai-native-sdlc-playbook) ของ Anthropic

หลักการ: แต่ละขั้นของงานจะ commit เอกสาร (artifact) ให้ขั้นถัดไปหยิบไปทำต่อ bot ทำงานทั้งหมดระหว่างขั้นเอง คนตัดสินใจแค่ 3 จุด

## Loop และ human gate

```text
idea / ticket / alert
   → Plan     intent.md                          (bot)
   → Design   spec.md + spec PR                  ── Gate 1: คน merge spec PR
   → Build    plan.md + code + tests + build PR  (bot)
   → Test     verify.md (bot ที่ไม่ได้เขียนโค้ดเอง)
                                                 ── Gate 2: code owner approve + merge
   → Deploy   dev → staging → release.md         ── Gate 3: release manager อนุมัติ production
   → Maintain alert / scan → intent.md ใหม่ → วนกลับไป Plan
```

| Gate | คนตัดสินใจอะไร | bot เตรียมอะไรให้ |
|---|---|---|
| 1. Spec | merge, ปิด หรือขอแก้ spec PR (open decision ที่ไม่ได้ติ๊กจะใช้ค่าที่ bot แนะนำ) | `intent.md` + `spec.md` พร้อมรายการ open decision |
| 2. PR | code owner approve แล้ว merge | CI ผ่าน, `verify.md` ผล `pass` ตรงกับ commit ล่าสุด, label `verified`, สรุปความเสี่ยง |
| 3. Production | อนุมัติ environment production ในระบบ deploy | `release.md`, ผลตรวจบน dev/staging, วิธี rollback |

- `intent.md`, `spec.md`, `plan.md` และ `verify.md` เก็บใน repo ของโปรดักต์ที่ `docs/sdlc/<work-id>/` ถ้าอยู่บน default branch แล้วถือว่า "อนุมัติแล้ว" ถ้าอยู่แค่ใน branch หรือ PR ถือว่า "เสนออยู่"
- `release.md` เก็บเป็น release record ของ platform เช่นคำอธิบาย release/tag หรือสรุปผลของ deploy run เพื่อไม่ต้องเปิด PR ซึ่งจะกลายเป็นขั้นที่ต้องรอคนเพิ่ม ถ้าทีมอยากเก็บใน git ให้รวมไปกับ build PR ถัดไปที่ `docs/sdlc/releases/<version>.md`
- **Combined gate:** งานแก้บั๊กที่มี test ยืนยันปัญหาได้ ไม่แตะ schema, auth, เงิน หรือ public contract และ diff ไม่เกินราว 200 บรรทัด รวม `intent.md`, `spec.md`, `plan.md` และ code ไว้ใน PR เดียวได้ code owner อนุมัติครั้งเดียวนับเป็นทั้ง gate 1 และ 2

รายละเอียดทั้งหมดอยู่ใน [chayen-sdlc-loop](chayen-sdlc-loop/SKILL.md)

## Skills

### Stage skills (แกนของ loop)

| Skill | หน้าที่ |
|---|---|
| [chayen-sdlc-loop](chayen-sdlc-loop/SKILL.md) | กติกากลางที่ **ทุก bot ต้องเปิด**: artifact, gate, กฎการตัดสินใจเองแทนการถาม, สิทธิ์ตาม environment, handoff, จำนวนรอบ rework สูงสุด, ศัพท์รายงานผล |
| [chayen-intent-spec](chayen-intent-spec/SKILL.md) | Plan + Design: เขียน `intent.md` → `spec.md` แล้วเปิด spec PR (gate 1) และแก้ spec ตาม review |
| [chayen-build](chayen-build/SKILL.md) | Build: เขียน `plan.md`, เขียนโค้ดและ test, เปิด build PR, แก้ CI และ rework |
| [chayen-verify](chayen-verify/SKILL.md) | Test: ตรวจ PR แบบอิสระ, เขียน `verify.md` ให้ผล pass, rework หรือ blocked, ถ้าผ่านจะขอ review จาก code owner |
| [chayen-release](chayen-release/SKILL.md) | Deploy: promote artifact ตัวเดิมไป dev/staging, เขียน `release.md`, ขออนุมัติ production (gate 3), เฝ้าดูหลัง release |
| [chayen-maintain](chayen-maintain/SKILL.md) | Maintain: triage alert, dependency/secret scan, CI ที่พัง, flaky test, เขียน postmortem แล้วสร้าง `intent.md` ใหม่ |
| [chayen-context](chayen-context/SKILL.md) | ดูแล `AGENTS.md`, `CLAUDE.md`, `REVIEW.md`, CODEOWNERS, PR template, eval ของ agent และกฎที่สรุปจากความผิดพลาดซ้ำ (ผ่าน PR เสมอ) |

### Domain skills (ความรู้เฉพาะด้าน)

| Skill | ครอบคลุม |
|---|---|
| [chayen-backend](chayen-backend/SKILL.md) | ASP.NET Core, Node servers (Express, Fastify, NestJS, Next.js, Nuxt), FastAPI/Django/Flask |
| [chayen-frontend](chayen-frontend/SKILL.md) | Angular, React/Next.js, Vue/Nuxt, Vite, app shells (PWA, Capacitor, Ionic, Electron, Tauri) |
| [chayen-data](chayen-data/SKILL.md) | SQL Server, PostgreSQL, migration แบบ expand → migrate → contract, สิทธิ์ตาม environment |
| [chayen-testing](chayen-testing/SKILL.md) | xUnit/NUnit/MSTest, Jest/Vitest/Karma, Playwright/Cypress, pytest, performance budget |
| [chayen-security](chayen-security/SKILL.md) | threat model, security review, scan triage, secret exposure, ความปลอดภัยของฟีเจอร์ agent/LLM |
| [chayen-devops](chayen-devops/SKILL.md) | CI/CD, container, IaC, observability, สิทธิ์รันคำสั่งแต่ละระดับ |
| [chayen-architecture](chayen-architecture/SKILL.md) | ADR, boundaries, multitenancy, resilience, migration, rollout, fitness test |

## ทีม bot ที่แนะนำ

| Bot | Stage | เปิด skill |
|---|---|---|
| Analyst | Plan, Design | sdlc-loop, intent-spec, architecture, security |
| Builder (1–n ตัว) | Build | sdlc-loop, build, backend, frontend, data, testing, devops, security |
| Verifier | Test | sdlc-loop, verify, testing, security, data, architecture, backend, frontend |
| Release | Deploy | sdlc-loop, release, devops, data |
| Ops | Maintain | sdlc-loop, maintain, devops, security, testing |
| Steward | Context | sdlc-loop, context |

Verifier ต้องเป็นคนละ bot กับ Builder เพราะ playbook ห้าม agent ตรวจงานที่ตัวเองเขียน และ bot ทุกตัวห้าม approve หรือ merge PR เอง

### Routines (MausBot triggers)

| Routine | Bot | Schedule (ตัวอย่าง) |
|---|---|---|
| Dispatcher: ตามงานที่ค้าง | Ops | `*/15 9-19 * * 1-5` Asia/Bangkok |
| CI บน default branch พัง | Ops | event หรือ `0 * * * *` Asia/Bangkok |
| Dependency + secret scan | Ops | `0 2 * * *` Asia/Bangkok |
| Flaky test triage | Ops | `0 8 * * 1` Asia/Bangkok |
| Context refresh + lessons | Steward | `0 9 * * 5` Asia/Bangkok |
| Alert webhook / release watch ผิดปกติ | Ops | event |

## สิ่งที่ต้องตั้งค่าเองนอก skill

Skill เป็นแค่ **คำแนะนำให้ agent ปฏิบัติตาม (advisory)** บังคับอะไรไม่ได้จริง gate ต้องให้ platform เป็นตัวบังคับ:

1. **Branch protection** บน default branch: ห้าม push ตรง, ต้องผ่าน required status checks (build, lint, test, scan) และต้องมี review จาก code owner
2. **CODEOWNERS**: กำหนดเจ้าของ `docs/sdlc/` (สำหรับ gate 1) และเจ้าของโค้ดแต่ละส่วน (สำหรับ gate 2)
3. **Environment protection** ของ production เช่น required reviewers ใน GitHub Environments, approvals ใน Azure DevOps หรือ protected environments ใน GitLab (สำหรับ gate 3)
4. **Token ของ bot**: ให้สิทธิ์เฉพาะ repo และ dev/staging เป็น token อายุสั้น และห้ามมีสิทธิ์เขียน production
5. **Approval level ใน MausBot**: ใช้ Full access เมื่อ bot รันบนเครื่องหรือ VM แยก ไม่อย่างนั้น routine ที่รันตามกำหนดเวลาจะค้างรอคนกด Allow
6. Skill ที่ import เข้า MausBot จะ **ถูกปิดไว้ก่อน** และ routine ที่ import จะ **ถูก pause** ต้องเปิดเองหลังอ่านแล้ว

**เริ่มใช้กับ repo ใหม่:** สั่ง Steward bot ให้ setup repo ตาม `chayen-context` แล้ว bot จะเปิด PR `sdlc/context-setup` ที่มี:

- `AGENTS.md` และ `CLAUDE.md` ที่ชี้ไปหา `AGENTS.md`
- `REVIEW.md`
- ร่าง CODEOWNERS (คนต้องเป็นผู้ใส่ชื่อเจ้าของจริงเอง)
- PR template
- `docs/sdlc/README.md`
- รายการ setting ของ platform ที่ยังขาดตามข้อ 1–3

## ข้อจำกัดของ MausBot ที่ skill ชุดนี้ออกแบบรองรับ

- MausBot ส่งต่อเฉพาะข้อความใน `SKILL.md` ไม่ส่งไฟล์อื่น ทุก skill จึงเขียนให้จบในไฟล์เดียว
- bot หนึ่งตัวเปิด skill ได้สูงสุด 30 ตัว ชุดนี้มีทั้งหมด 14 ตัว และ bot ที่ใช้ skill มากที่สุดในทีมแนะนำใช้ 8 ตัว
- คำถามที่ไม่มีใครตอบจะทำให้ routine หยุด skill จึงให้ bot ตัดสินใจเองแล้วบันทึกเป็น assumption และเก็บเรื่องที่ตัดสินใจเองไม่ได้ไปไว้ที่ gate 1
- run ที่ถูกตัดกลางทางจะเริ่มใหม่ได้ skill จึงให้ bot ตรวจของที่มีอยู่แล้ว (branch, PR, artifact, deploy) ก่อนสร้างใหม่ทุกครั้ง

## Changelog

**2026-11-12**: ปรับใหม่สำหรับ MausBot + AI-native SDLC (commit `f7f1de5`)

- เพิ่ม: `sdlc-loop`, `intent-spec`, `build`, `verify`, `release`, `maintain`, `context`
- เขียนใหม่ให้จบในไฟล์เดียวและทำงานอัตโนมัติเป็นค่าเริ่มต้น: `backend`, `frontend`, `data`, `testing`, `security`, `devops`, `architecture`
- ลบออก (ยังกู้คืนได้จาก git history ที่ commit `b8d09d4`):

| ตัวเดิม | เหตุผล / ย้ายไปที่ |
|---|---|
| chayen-implementation | ย้ายไป `chayen-build` |
| chayen-requirements | ย้ายไป `chayen-intent-spec` |
| chayen-init | ย้ายไป `chayen-context` (เปลี่ยนจาก `.chayen/` มาใช้ `AGENTS.md`) |
| chayen-documentation | แยกไปตามขั้น: PR description → build, code review → verify, release notes → release, เอกสารสำหรับ agent → context |
| chayen-repository | ให้ bot วิเคราะห์โค้ดเองระหว่างทำงาน ส่วน impact analysis ย้ายไปอยู่ใน `plan.md` |
| chayen-performance | ส่วน performance budget ย้ายไป `chayen-testing` และการเฝ้าดู latency/error หลัง release ย้ายไป `chayen-release` และ `chayen-maintain` |
| chayen-ai-engineering | เฉพาะทีมที่ทำฟีเจอร์ ML/LLM ส่วน eval ของ agent ย้ายไป `chayen-context` และความปลอดภัยของ agent/LLM ย้ายไป `chayen-security` |
| chayen-humanizer-en / -th | เป็นงานเกลาสำนวนข้อความ ไม่ใช่ขั้นตอนของ SDLC |
| ไฟล์ `references/` ทั้งหมด | ไม่เคยมีอยู่ใน repo นี้ และ MausBot ก็ไม่ส่งต่ออยู่แล้ว |
