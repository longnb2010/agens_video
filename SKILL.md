# AGNES VIDEO SKILL — COMPACT (v2)

Execution contract. Agent follow exact. No skip gate.

---

## PRIORITY (conflict order)
user request > approval/safety gate > scenario > brand/production policy > generation strategy > template/example

## TIE-BREAK
repeated_cast + high_identity_precision → keyframe wins (identity = hard constraint > continuity preference)
other unlisted conflict → STOP, ask user. Never silent-pick.

---

## HARD RULES (never break)
- video: 9:16, 8s default
- t2v: no readable text, no real UI, no precise logo, no subtitle-in-gen, no quoted dialogue
- audio: t2v audio = disposable. final audio = mute → vi male TTS → BGM
- ui/logo: gen forbidden, post_overlay allowed
- user_approval required before: video gen, new cast, final delivery
- default regen scope: single clip. never touch passed clips w/o user ask
- cast real person (khách hàng/nhân viên thật): dùng đúng project được giao, không tái dùng project khác trừ user confirm. User bảo bỏ cast → bỏ ngay kể cả clip đã approved (exception duy nhất cho "không đụng approved state")

---

## WORKFLOW
REQUEST → NORMALIZE → SCENARIO → [WAIT APPROVAL] → CAST RESOLVE → [WAIT APPROVAL] → MANIFEST → PREFLIGHT → GEN → PRE-QC → USER QC → FIX LOOP → FINAL QC → POST → DELIVERY

## STATE MACHINE (short)
```
SCENARIO_DRAFT --approve--> CAST_RESOLUTION --reject--> SCENARIO_DRAFT
CAST_RESOLUTION: missing/partial cast -> CAST_GENERATION -> CAST_APPROVAL
  (chỉ approve cast MỚI, cast approved cũ dùng thẳng)
CAST_APPROVAL --approve--> PREFLIGHT --reject--> CAST_GENERATION
PREFLIGHT --pass--> GENERATION --fail--> BLOCKED
BLOCKED: KHÔNG tự retry preflight. Báo user field fail + chờ input mới, mới re-run.
GENERATION -> PRE_QC --pass--> USER_QC --fail--> FIX_LOOP
USER_QC --approve--> FINAL_QC --reject--> FIX_LOOP
FIX_LOOP -> GENERATION (version += 1)
FINAL_QC --pass--> POST --> DELIVERY(terminal)
```
ambiguous feedback → stay state, ask clarify, no regen.

---

## SCENARIO
= source of truth nội dung. Full series scenario trước khi gen bất kỳ clip nào.
clip fields: clip_id, objective, cast[], action, environment, camera{shot,movement}, mood, narration_mapping, ui{required,assets}, branding{required,assets}, generation{identity_precision, motion_complexity}

## CAST
registry: cast_id{file, role, identity_lock(low/med/high), approved(bool), reused_from(optional)}
mapping: clip_id -> [cast_id...]
validate: scenario.cast == mapping.cast, all assets exist, approved==true → else PREFLIGHT FAIL
image read: chỉ lấy fact thật (subject/face/outfit/scene/logo/text/color). KHÔNG invent face/clothing/logo/env/identity.
missing cast: generate → CAST QC → USER APPROVE → usable. Never gen video với cast chưa approve.

## ASSETS
reference: cast only
post_overlay: ui, logo, subtitle
forbidden_in_gen: real_ui, readable_logo, readable_text

## STRATEGY ENGINE
repeated_cast → reference (fallback keyframe)
high_identity_precision → keyframe (fallback reference)
simple_motion → reference (fallback text)
no_cast_continuity → text
ui_required → t2v blank_gradient, actual ui = post_overlay
multi-rule match & conflict → apply TIE-BREAK trước khi build prompt.

## PROMPT COMPILER
= SCENARIO + CAST + CAMERA + MOOD + POLICIES, compiled không copy-paste tay.
Luôn English, kể cả scenario gốc tiếng Việt.
Cast ref template: "Use <Picture 1> as {ROLE_1}, <Picture 2> as {ROLE_2}..." — giữ đúng thứ tự.

## ANTI-TEXT LOCK (inject mọi prompt)
no text/letters/numbers/logo/signage/subtitle/watermark/nametag/writing-on-surface/foreign-script anywhere. Screens = blank or blue-white gradient only, zero UI/icon. No visible-print papers. Documentary look.

## AUDIO LOCK (inject mọi prompt)
no dialogue/speech/lip-sync/voiceover any language. Silent gesture only. Optional subtle ambience, no human voice.
Pipeline: t2v audio → MUTE → vi male TTS → BGM.

## UI/LOGO POLICY
Gen: physical device + blank gradient screen only. Post: motion-track + overlay real asset (vms_mobile.png / vms_pc.png / logo).

## BRAND
primary #0066CC, secondary #23C1F5. Font: Roboto/Inter/Be Vietnam Pro. Logo gen forbidden, post only.

---

## CLIP MANIFEST (per clip)
version, status{generation, pre_qc, user_approval}, scenario{...}, assets{references, post_overlay}, generation{model, mode, duration=8, size=720P, ratio=9:16}, prompt{scene, policies}, narration{vi, male, text}, post{mute=true, tts=male_vi, subtitles=true, ui_overlay, logo_overlay}, qc{technical, visual, audio, cast}, version_history[], output{working_path, final_path}
= source of truth cho clip đó.

## OUTPUT PATH
```
projects/{project}/{week}/renders/{clip_id}_v{version}.mp4   ← mọi version, không xoá
projects/{project}/{week}/final/{clip_id}.mp4                ← bản approved
projects/{project}/{week}/final/final_video.mp4              ← ghép series
```

## ASSET DISCOVERY (chạy TRƯỚC preflight, mỗi khi cast_registry/ui/logo đổi)
List thật assets/cast/*, assets/ui/*, assets/logo/* trên disk — KHÔNG suy ra file tồn tại chỉ vì có tên trong cast_registry/ui.yaml/brand.yaml.
Cross-check: mọi cast_registry[*].file, ui_assets[*].file (vms_mobile.png/vms_pc.png), brand.logo file phải thật sự có trong 3 thư mục trên.
Mismatch (claim có nhưng disk không có) → PREFLIGHT FAIL, fail_reason = "asset referenced nhưng không tồn tại trên disk: {filename}".
Field `approved: true` / `file: xxx.png` trong yaml chỉ là claim, không phải bằng chứng — assets_ready chỉ = true sau khi discovery confirm.

## PREFLIGHT (before every gen call)
check: asset_discovery đã pass (bắt buộc trước), scenario.approved, cast.mapping_valid+assets_ready+approval_valid, ui/logo ready if required, mode_selected+conflict_resolved, ratio=9:16, duration=8, prompt.english_only+anti_text_lock+audio_lock+cast_mapping_present, post.narration_separated+audio_discarded
FAIL → fail_reason required → NO API CALL → BLOCKED
PASS → proceed GENERATION

## GENERATION
Sequential only, 1 clip at a time: gen → record metadata → pre-QC → next per workflow. No parallel assume.

## API DEFAULTS
video: model=agnes-video-2.5-flash, mode=reference/keyframe/text, seconds=8, size=720P, ratio=9:16
image: model=agnes-image-2.1-flash, size=2K, ratio=9:16, endpoint=POST https://apihub.agnes-ai.com/v1/images/generations
video endpoint: chưa confirm trong spec — hỏi/xác nhận trước call đầu tiên mỗi project mới, không tự suy từ image endpoint.
rate limit = real constraint, không assume unlimited.

## TECHNICAL RETRY (lỗi hạ tầng — KHÁC lỗi content)
applies: timeout, 5xx, 429, malformed response, corrupted download
max 3 retry, exponential backoff (5s/15s/45s)
KHÔNG tăng clip version. Đếm riêng khỏi content QC attempts.
sau max → BLOCKED, báo lỗi kỹ thuật cho user rõ ràng, không âm thầm hạ chất lượng để né.

---

## QC

pre_qc (auto, sau mỗi gen): technical{file_exists, duration, ratio, resolution}, visual{text, foreign_script, fake_ui, fake_logo, cast_consistency, character_presence, background, artifact}, audio{unwanted_dialogue, foreign_lang, track} → PASS/FAIL

user_qc (chỉ gửi sau pre_qc PASS): identity{face, outfit, pet}, creative{action, background, composition, mood, story}, visual{unwanted_text, fake_ui, watermark}, format{ratio, duration} → APPROVED/NEED_FIX

final_qc (chỉ chạy khi ALL clip user-approved): scenario+cast PASS, all clips approved, technical/visual/audio/branding PASS → gate cho DELIVERY

---

## FIX LOOP
feedback → parse clip → classify failure → pick strategy (mục FAILURE HANDLERS) → update manifest → version+=1 → PREFLIGHT → regen CHỈ clip đó → pre-QC → user QC.
Không regen cả series.

## FAILURE TYPES (content only — lỗi hạ tầng xem TECHNICAL RETRY)
identity: face/outfit/pet_identity/character_swap
composition: missing_character/wrong_position/wrong_action/wrong_camera
environment: wrong_background/location/lighting
visual: text/watermark/fake_logo/fake_ui/anatomy/artifact
technical: aspect_ratio/duration/resolution/corrupted_file
audio: unwanted_dialogue/foreign_language/wrong_voice
creative: wrong_story/wrong_mood/wrong_message

## FAILURE HANDLERS
face → approved_cast_ref → keyframe → regen
outfit → approved_cast_ref → strengthen_instruction
pet_identity → approved_pet_ref → keyframe_if_needed
background → simplify_env → strengthen_prompt → keyframe → alt_model
text → remove_risk_object → strengthen_anti_text → regen/post_blur
fake_ui → blank_screen → remove_ui_from_gen → post_overlay
aspect_ratio → correct_api_param (no regen needed nếu chỉ param sai)
missing_character → validate_mapping → strengthen_picture_mapping
unwanted_dialogue → mute_only, NO REGEN, version không tăng
wrong_story → revise_scene → regen

## RETRY BUDGET = VERSION COUNTER (1 counter duy nhất, không tách attempt riêng)
max 3 version/clip cho lỗi content.
v1=attempt1, v2=attempt2(sau fix1), v3=attempt3(sau fix2).
Chạm max 3 → BLOCKED → require_user_decision. Không tự tạo v4.
Fix bằng post-production (VD mute audio) → KHÔNG tăng version.
Fix hạ tầng (TECHNICAL RETRY) → KHÔNG tăng version.

## VERSIONING
Never overwrite: clip_03_v001.mp4, v002, v003...
history log: version, status, reason.
version thành "current" chỉ khi PASS user_qc.

---

## PRESERVE APPROVED STATE
Không tự đổi: approved_scenario, approved_cast, approved_narration, approved_ui_assets, approved_logo_assets.
Chỉ đổi khi user yêu cầu hoặc failure handler bắt buộc.
Exception: real-person cast bị user revoke → bỏ ngay cả ở clip đã approved (xem HARD RULES).

## AGENT COMMAND MAP
"Tạo WEEK N" → parse → build scenario → WAIT approval
"OK scenario" → resolve cast (chỉ cast mới) → WAIT approval nếu có cast mới
"Gen đi" → verify approval state → preflight → generate
"Clip X sai mặt" → failure=face → tie-break→keyframe → regen clip X only, version+=1
"Clip X có tiếng nước ngoài" → mute audio only, NO regen, version không tăng
"Clip X, Y OK" → mark approved, không regen

## ANTI-DRIFT
NEVER: invent cast/logo/UI/narration, đổi approved scenario/cast (trừ exception), regen passed clip, skip approval, auto-retry preflight loop vô nghĩa (phải có input mới)
ALWAYS: reuse approved asset/scenario, track version (1 counter), track failure reason, update manifest, phân biệt lỗi hạ tầng vs lỗi content

## PROJECT DATA
KHÔNG hard-code WEEK cụ thể / customer / narration / cast mapping / campaign vào core skill.
Mỗi project riêng: projects/{week}/{scenario.yaml, cast_mapping.yaml, narration.yaml, manifests/, renders/, final/}

---

## NON-NEGOTIABLE (đọc kỹ, đây là hard gate)
1. NO APPROVAL → NO GENERATION
2. SCENARIO = SOURCE OF TRUTH
3. CAST MAPPING MUST MATCH SCENARIO
4. PREFLIGHT FAIL → NO API CALL
5. T2V TEXT FORBIDDEN
6. REAL UI = POST-PRODUCTION ONLY
7. T2V ORIGINAL AUDIO = DISPOSABLE
8. REPEATED CAST → REFERENCE/KEYFRAME (identity precision thắng khi conflict)
9. ONE FAILED CLIP → FIX ONE CLIP ONLY
10. NEVER DESTROY APPROVED VERSIONS
11. FINAL DELIVERY = ALL GATES PASS
12. VERSION = DUY NHẤT COUNTER CHO LỖI CONTENT; LỖI HẠ TẦNG RETRY RIÊNG, KHÔNG TĂNG VERSION
13. BLOCKED KHÔNG TỰ THOÁT — CẦN INPUT MỚI/LỆNH RÕ TỪ USER
14. FILE TRONG YAML KHÔNG PHẢI BẰNG CHỨNG — PHẢI LIST ASSETS/ THẬT TRƯỚC KHI COI LÀ SẴN CÓ