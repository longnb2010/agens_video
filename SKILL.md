# AGNES VIDEO — VMS PRODUCTION SKILL

Production skill cho Agnes Video dùng cho VMS marketing, WEEK series, Reels 9:16 và video documentary ngắn.

Skill này là **execution contract**. Agent phải tuân thủ workflow, approval gate, generation policy, QC policy và recovery policy bên dưới.

> **Changelog v2**: bổ sung tie-break cho strategy engine, reconcile counter attempt/version, làm rõ trigger của state BLOCKED, tách retry kỹ thuật (API) khỏi retry chất lượng (QC), thêm output path convention, làm rõ case cast một phần, thêm note xử lý ảnh người thật.

---

# 0. CORE CONTRACT

## 0.1. Priority

Khi có conflict, áp dụng thứ tự:

```text
USER explicit request
    >
APPROVAL / SAFETY GATES
    >
CURRENT SCENARIO
    >
BRAND / PRODUCTION POLICIES
    >
GENERATION STRATEGY
    >
TEMPLATES / EXAMPLES
```

Template hoặc example không được override scenario hiện tại.

---

## 0.2. Immutable production rules

```yaml
core_rules:

  video:
    aspect_ratio: "9:16"
    default_duration: "8s"

  t2v:
    readable_text: forbidden
    real_ui: forbidden
    precise_logo_rendering: forbidden
    subtitles: forbidden          # chỉ trong generation; post_overlay subtitle vẫn allowed, xem mục 8
    quoted_dialogue: forbidden

  audio:
    original_t2v_audio: disposable
    final_dialogue_source: "post_tts"
    final_voice_language: "vi"
    final_voice_gender: "male"

  ui:
    generation: forbidden
    post_overlay: allowed

  logo:
    generation: forbidden
    post_overlay: allowed

  repeated_cast:
    preferred_strategy: reference

  high_identity_precision:
    preferred_strategy: keyframe

  user_approval:
    scenario_before_video: required
    new_cast_before_video: required
    final_delivery: required

  regeneration:
    default_scope: "single_clip"
    passed_clips: do_not_regenerate_without_user_request
```

## 0.3. Conflict resolution (tie-break rules)

Một số rule ở 0.2 có thể áp dụng đồng thời lên cùng một clip và mâu thuẫn nhau. Khi đó:

```yaml
tie_break:

  repeated_cast + high_identity_precision:
    winner: high_identity_precision
    resolved_strategy: keyframe
    reason: >
      Identity precision là hard constraint (sai mặt = fail QC ngay).
      Continuity (reference) chỉ là preference tối ưu chi phí/tốc độ,
      có thể nhường khi identity risk cao.

  ambiguous_or_unlisted_conflict:
    action: escalate_to_user
    do_not: silently_pick_one_side
```

Nguyên tắc chung: nếu một rule là **hard constraint về chất lượng/identity** và rule kia là **preference về tốc độ/chi phí**, hard constraint thắng. Nếu conflict không nằm trong danh sách trên, agent không tự quyết — hỏi user.

---

# 1. WORKFLOW

Agent luôn thực hiện:

```text
REQUEST
  ↓
NORMALIZE INPUT
  ↓
SCENARIO
  ↓
WAIT SCENARIO APPROVAL
  ↓
CAST RESOLUTION
  ↓
WAIT CAST APPROVAL
  ↓
BUILD CLIP MANIFEST
  ↓
PREFLIGHT
  ↓
GENERATE
  ↓
PRE-QC
  ↓
USER QC
  ↓
FIX LOOP
  ↓
FINAL QC
  ↓
POST
  ↓
DELIVERY
```

Không được bỏ qua approval gate.

---

# 2. STATE MACHINE

```yaml
states:

  INIT:
    next: INPUT_READY

  INPUT_READY:
    next: SCENARIO_DRAFT

  SCENARIO_DRAFT:
    approval: required
    on_approved: CAST_RESOLUTION
    on_rejected: SCENARIO_DRAFT

  CAST_RESOLUTION:
    on_missing_cast: CAST_GENERATION
    on_partial_cast: CAST_GENERATION   # xem 2.1 — chỉ generate phần thiếu
    on_all_assets_ready: CAST_APPROVAL

  CAST_GENERATION:
    next: CAST_APPROVAL

  CAST_APPROVAL:
    approval: required
    on_approved: PREFLIGHT
    on_rejected: CAST_GENERATION

  PREFLIGHT:
    on_pass: GENERATION
    on_fail: BLOCKED

  BLOCKED:
    entry_reason: required           # phải ghi rõ preflight field nào fail
    exit_trigger: see 2.2            # KHÔNG tự động loop
    next: PREFLIGHT

  GENERATION:
    next: PRE_QC

  PRE_QC:
    on_pass: USER_QC
    on_fail: FIX_LOOP

  USER_QC:
    approval: required
    on_approved: FINAL_QC
    on_rejected: FIX_LOOP

  FIX_LOOP:
    next: GENERATION

  FINAL_QC:
    on_pass: POST
    on_fail: FIX_LOOP

  POST:
    next: DELIVERY

  DELIVERY:
    terminal: true
```

### 2.1. Cast resolution — case cast một phần

Thực tế phổ biến nhất: một số cast đã approved sẵn (VD: manager, receptionist tái sử dụng từ WEEK trước), một số cast mới cần generate (VD: khách hàng mới của WEEK này).

```text
Với mỗi cast_id trong cast_mapping:
  IF cast approved và asset tồn tại → dùng luôn, KHÔNG generate lại
  IF cast thiếu hoặc chưa approved → đưa vào CAST_GENERATION

CAST_APPROVAL chỉ chờ approval cho phần cast MỚI/THAY ĐỔI,
không yêu cầu re-approve cast đã approved từ trước (xem mục 28).
```

### 2.2. BLOCKED — exit trigger

State `BLOCKED` không tự động quay lại `PREFLIGHT`. Agent phải:

```text
1. Báo user rõ preflight field nào fail (scenario / cast / assets / prompt / post).
2. Đề xuất hành động cần thiết (VD: "cần user approve cast_manager trước").
3. Chờ user cung cấp thứ còn thiếu, hoặc user ra lệnh rõ ràng để tiếp tục.
4. Chỉ re-run PREFLIGHT sau khi điều kiện fail đã được giải quyết.
```

Agent không tự ý retry PREFLIGHT nhiều lần liên tiếp mà không có thay đổi input — đó là vòng lặp vô ích.

### Approval semantics

```yaml
approval_rules:

  sent_to_user:
    means: "submitted"
    does_not_mean: "approved"

  approved:
    requires: "explicit user confirmation"

  ambiguous_feedback:
    requires_clarification: true
    state_action: "stay in current approval state, do not advance, do not regenerate"

  explicit_feedback:
    parse_without_asking_again: true
```

---

# 3. INPUT NORMALIZATION

Agent phải convert request thành `week_brief`.

```yaml
week_brief:

  project:
  week:
  topic:
  objective:
  audience:

  clip_count:
  duration:

  narration:
    segments: []

  cast_requirement: []

  ui_requirement: []

  branding_requirement: []

  creative_direction:
```

Không bắt đầu generation trực tiếp từ user prose.

---

# 4. SCENARIO

Scenario là **source of truth về nội dung**.

Mỗi clip bắt buộc:

```yaml
clip:
  clip_id:
  objective:

  cast: []

  action:
  environment:

  camera:
    shot:
    movement:

  mood:

  narration_mapping:

  ui:
    required: false
    assets: []

  branding:
    required: false
    assets: []

  generation:
    identity_precision: normal
    motion_complexity: normal
```

Agent phải tạo scenario cho toàn bộ series trước khi video generation.

---

# 5. SCENARIO APPROVAL GATE

```yaml
scenario_gate:

  required: true

  before:
    - cast_generation
    - video_generation

  status:
    initial: DRAFT
    approved: APPROVED
    rejected: NEED_FIX
```

Nếu USER chưa approve:

```text
DO NOT GENERATE VIDEO
```

---

# 6. CAST RESOLUTION

## 6.1. Cast registry

```yaml
cast_registry:

  cast_id:
    file:
    role:
    description:
    identity_lock:
      low | medium | high

    approved:
      true | false

    reused_from:      # optional — WEEK/project gốc nếu tái sử dụng cast cũ
```

Ví dụ:

```yaml
cast_manager:
  file: cast_manager.png
  role: clinic_manager
  identity_lock: high
  approved: true
  reused_from: week10
```

---

## 6.2. Cast mapping

Mapping phải được tạo từ scenario:

```yaml
cast_mapping:

  clip_1:
    - cast_manager
    - cast_reception

  clip_2:
    - cast_customer
    - cast_pet

  clip_3:
    - cast_customer

  clip_4:
    - cast_customer
    - cast_reception
    - cast_pet
    - cast_manager
```

## 6.3. Validation

```yaml
cast_validation:

  scenario_cast_matches_mapping: required
  all_required_assets_exist: required
  approved_cast_available: required
```

Nếu:

```text
scenario_cast != mapped_cast
```

→ `PREFLIGHT FAIL`

---

# 7. CAST IMAGE POLICY

Khi có uploaded image:

```text
read_image_for_prompt()
```

Focus:

```text
subject
face
outfit
scene
logo
visible text
colors
```

Chỉ đưa facts thực tế từ image analysis vào prompt.

Không tự invent:

```text
face
clothing
logo
environment
identity
```

## 7.1. Missing cast

Nếu cast asset thiếu:

```text
Image generation
    ↓
CAST QC
    ↓
USER APPROVAL
    ↓
usable cast
```

Không dùng cast mới chưa được approve để generate video.

## 7.2. Ảnh người thật (khách hàng / nhân sự phòng khám)

Cast image có thể là ảnh thật của khách hàng hoặc nhân viên (không phải nhân vật hư cấu). Với loại ảnh này:

```text
- Chỉ dùng trong phạm vi project/WEEK được giao, không tái sử dụng sang project khác
  trừ khi user xác nhận rõ ràng (xem cast_registry.reused_from).
- Không lưu trữ/chia sẻ asset này ngoài phạm vi delivery output đã thống nhất.
- Nếu user yêu cầu xoá/ngừng dùng một cast thật, dừng dùng ngay ở mọi clip
  liên quan, kể cả clip đã approved trước đó (không giữ lại "vì đã pass").
```

---

# 8. ASSET POLICY

Assets chia thành 3 nhóm.

```yaml
assets:

  reference:
    allowed:
      - cast

  post_overlay:
    allowed:
      - ui
      - logo
      - subtitle

  forbidden_in_generation:
    - real_ui
    - readable_logo
    - readable_text
```

---

# 9. GENERATION STRATEGY ENGINE

Agent không tự chọn mode ngẫu nhiên.

```yaml
strategy_rules:

  repeated_cast:
    primary: reference
    fallback:
      - keyframe

  high_identity_precision:
    primary: keyframe
    fallback:
      - reference

  simple_motion:
    primary: reference
    fallback:
      - text

  no_cast_continuity:
    primary: text

  ui_required:
    t2v_screen: blank_gradient
    actual_ui: post_overlay
```

Khi một clip khớp nhiều hơn một rule ở trên và các rule đó mâu thuẫn (VD: vừa `repeated_cast` vừa `high_identity_precision`), áp dụng **tie-break ở mục 0.3** trước khi build prompt — không cộng dồn cả hai chiến lược, không tự chọn ngẫu nhiên.

---

# 10. REFERENCE / KEYFRAME / TEXT

## Reference

Dùng khi:

```text
same character
same pet
series continuity
```

Mặc định cho cast lặp lại (trừ khi bị tie-break ở 0.3 chuyển sang keyframe).

## Keyframe

Dùng khi:

```text
face consistency rất quan trọng
composition phức tạp
reference mode chưa giữ identity tốt
```

Pipeline:

```text
cast reference
    ↓
Image keyframe
    ↓
QC keyframe
    ↓
Video keyframe
```

## Text

Chỉ dùng khi:

```text
scene độc lập
không yêu cầu identity continuity
```

---

# 11. PROMPT COMPILER

Agent không được copy một prompt dài rồi sửa bằng tay cho từng clip.

Prompt được compile từ:

```text
SCENARIO
+
CAST
+
CAMERA
+
MOOD
+
POLICIES
```

Conceptual structure:

```text
CAST REFERENCE
+
ENVIRONMENT
+
ACTION
+
CAMERA
+
MOOD
+
AUDIO POLICY
+
ANTI-TEXT POLICY
```

Prompt luôn viết bằng tiếng Anh (xem `preflight.prompt.english_only`, mục 18), kể cả khi scenario/narration gốc bằng tiếng Việt.

---

# 12. CAST REFERENCE TEMPLATE

Khi có nhiều cast:

```text
CAST REFERENCE: Use <Picture 1> as {ROLE_1}, <Picture 2> as {ROLE_2}, <Picture 3> as {ROLE_3}.
```

Không map sai thứ tự.

Ví dụ:

```text
Picture 1 = manager
Picture 2 = receptionist
```

thì prompt phải giữ đúng mapping.

---

# 13. ANTI-TEXT POLICY

## Full

```text
ANTI-TEXT LOCK: absolutely no readable text anywhere in the frame, no letters, no numbers, no logos, no brand names, no subtitles, no captions, no watermarks, no signage, no posters, no name tags, no embroidered text on uniforms, no writing on walls doors or glass, no Thai script, no Chinese characters, no Japanese characters, no Korean hangul, no Vietnamese diacritics, no random glyphs. All phone tablet monitor and TV screens must be completely blank or show only a smooth soft blue-white color gradient with zero UI elements and zero icons. No clipboard documents papers receipts or forms with visible print. Documentary cinematic look only.
```

## Compact

```text
No text letters numbers logos signage anywhere. Blank gradient screens only. No Thai Chinese Japanese script. No clipboard papers. Documentary.
```

---

# 14. AUDIO POLICY

## Full

```text
AUDIO LOCK: completely silent human dialogue, no spoken words, no lip-sync talking, no voiceover in any language, no Thai speech, no foreign language chatter, characters communicate only through gestures and expressions with closed or barely moving lips, optional very subtle room ambience only with no human voices.
```

## Compact

```text
No dialogue no speech no lip-sync. Silent gestures only. No human voices in audio.
```

## Final audio pipeline

```text
T2V audio
   ↓
MUTE
   ↓
Vietnamese male TTS
   ↓
BGM
```

T2V audio không được xem là final audio source.

---

# 15. UI POLICY

Không generate actual application UI bằng T2V.

Thay bằng:

```text
physical phone/tablet
+
blank gradient screen
```

sau đó:

```text
post-production
+
motion tracking
+
real UI asset
```

Assets:

```yaml
ui_assets:

  mobile:
    file: vms_mobile.png
    usage: post_overlay

  pc:
    file: vms_pc.png
    usage: post_overlay
```

---

# 16. BRAND POLICY

Logo thật phải đưa vào post.

Không yêu cầu T2V render logo chính xác.

```yaml
brand:

  primary: "#0066CC"
  secondary: "#23C1F5"

  logo:
    generation: forbidden
    post_overlay: allowed

  typography:
    preferred:
      - Roboto
      - Inter
      - Be Vietnam Pro
```

---

# 17. CLIP MANIFEST

Sau scenario approval, tạo manifest cho từng clip.

```yaml
clip_manifest:

  clip_id:
  version: 1

  status:
    generation: NOT_GENERATED
    pre_qc: PENDING
    user_approval: PENDING

  scenario:
    objective:
    cast:
    action:
    environment:
    camera:
    mood:

  assets:
    references: []
    post_overlay: []

  generation:
    model:
    mode:
    duration: 8
    size: 720P
    aspect_ratio: "9:16"

  prompt:
    scene:
    policies: []

  narration:
    language: vi
    voice: male
    text:

  post:
    mute_original_audio: true
    tts: male_vi
    subtitles: true
    ui_overlay: []
    logo_overlay: []

  qc:
    technical: PENDING
    visual: PENDING
    audio: PENDING
    cast: PENDING

  metadata:
    version_history: []        # xem mục 26.1 — "version" là counter duy nhất
    generated_at:
    prompt_hash:

  output:
    working_path: "projects/{project}/{week}/renders/{clip_id}_v{version}.mp4"
    final_path: "projects/{project}/{week}/final/{clip_id}.mp4"
```

Manifest là source of truth cho clip đó.

---

# 18. PREFLIGHT

PREFLIGHT chạy **trước mỗi video generation attempt**.

```yaml
preflight:

  scenario:
    approved: true

  cast:
    mapping_valid: true
    assets_ready: true
    approval_valid: true

  assets:
    required_ui_ready: true
    required_logo_ready: true

  generation:
    mode_selected: true
    mode_conflict_resolved: true   # xem 0.3 — nếu strategy đụng nhau phải resolve trước
    aspect_ratio: "9:16"
    duration: 8

  prompt:
    english_only: true
    anti_text_lock: true
    audio_lock_when_required: true
    cast_mapping_present: true

  post:
    narration_separated: true
    original_audio_discarded: true

  result:
    PASS | FAIL
    fail_reason: <field cụ thể>   # bắt buộc khi FAIL, để BLOCKED (mục 2.2) biết báo gì cho user
```

## Generation gate

```yaml
generation_gate:
  condition: "preflight.result == PASS"
```

Nếu FAIL:

```text
NO API CALL
```

---

# 19. GENERATION

Mặc định:

```yaml
generation_execution:

  order: sequential

  rule:
    - "Generate one clip."
    - "Record metadata."
    - "Run pre-QC."
    - "Continue according to workflow."
```

Không giả định parallel generation.

---

# 20. API DEFAULTS

## Agnes Video

```json
{
  "model": "agnes-video-2.5-flash",
  "mode": "reference",
  "seconds": "8",
  "size": "720P",
  "aspect_ratio": "9:16",
  "images": [],
  "prompt": ""
}
```

## Keyframe

```json
{
  "model": "agnes-video-2.5-flash",
  "mode": "keyframe",
  "first_frame": "",
  "seconds": "8",
  "size": "720P",
  "aspect_ratio": "9:16",
  "prompt": ""
}
```

## Text fallback

```json
{
  "model": "agnes-video-2.5-flash",
  "mode": "text",
  "seconds": "8",
  "size": "720P",
  "aspect_ratio": "9:16",
  "prompt": ""
}
```

## Image

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "",
  "size": "2K",
  "ratio": "9:16",
  "extra_body": {
    "image": [],
    "response_format": "url"
  }
}
```

Endpoint from current project specification:

```text
POST https://apihub.agnes-ai.com/v1/images/generations
```

Video endpoint cụ thể chưa được xác nhận trong spec hiện tại — agent phải hỏi/xác nhận endpoint video trước lần gọi API đầu tiên của mỗi project mới, thay vì tự suy ra từ endpoint image.

Video rate limit should be treated as a project/API constraint, not assumed unlimited. Current VMS specification expects sequential generation where needed.

## 20.1. Technical / API failure handling

Đây là loại lỗi **khác** với content-quality failure ở mục 24-26. Lỗi kỹ thuật (timeout, 5xx, rate limit, network error, corrupted response) không tính vào `retry_policy.max_attempts_per_clip` của QC.

```yaml
technical_retry:

  applies_to:
    - timeout
    - http_5xx
    - rate_limit_429
    - malformed_response
    - corrupted_file_on_download

  max_retries: 3
  backoff: exponential          # VD: 5s, 15s, 45s
  counted_separately_from: content_qc_attempts

  after_max_retries:
    status: BLOCKED
    action: report_technical_error_to_user   # không âm thầm giảm chất lượng để né lỗi
```

Không được lẫn lỗi kỹ thuật vào failure taxonomy ở mục 24 (đó là lỗi nội dung/hình ảnh, không phải lỗi hạ tầng).

---

# 21. PRE-QC

Sau mỗi generation:

```yaml
pre_qc:

  technical:
    file_exists:
    duration:
    aspect_ratio:
    resolution:

  visual:
    readable_text:
    foreign_script:
    fake_ui:
    fake_logo:
    cast_consistency:
    character_presence:
    background:
    visual_artifact:

  audio:
    unwanted_dialogue:
    foreign_language:
    audio_track:

  result:
    PASS | FAIL
```

---

# 22. USER QC

Chỉ gửi USER sau khi pre-QC đã PASS.

USER kiểm:

```yaml
user_qc:

  identity:
    face:
    outfit:
    pet:

  creative:
    action:
    background:
    composition:
    mood:
    story_match:

  visual:
    unwanted_text:
    fake_ui:
    watermark:

  format:
    aspect_ratio:
    duration:

  decision:
    APPROVED | NEED_FIX
```

---

# 23. FIX LOOP

Khi USER feedback:

```text
USER FEEDBACK
    ↓
PARSE CLIP
    ↓
CLASSIFY FAILURE
    ↓
SELECT RECOVERY STRATEGY
    ↓
UPDATE MANIFEST
    ↓
INCREMENT VERSION
    ↓
PREFLIGHT
    ↓
REGENERATE ONLY FAILED CLIP
    ↓
PRE-QC
    ↓
USER QC
```

Không regenerate toàn bộ series.

---

# 24. FAILURE TAXONOMY

Chỉ áp dụng cho lỗi **nội dung/chất lượng**. Lỗi hạ tầng/API xem mục 20.1.

```yaml
failure_types:

  identity:
    - face
    - outfit
    - pet_identity
    - character_swap

  composition:
    - missing_character
    - wrong_position
    - wrong_action
    - wrong_camera

  environment:
    - wrong_background
    - wrong_location
    - wrong_lighting

  visual:
    - text
    - watermark
    - fake_logo
    - fake_ui
    - anatomy
    - artifact

  technical:
    - aspect_ratio
    - duration
    - resolution
    - corrupted_file

  audio:
    - unwanted_dialogue
    - foreign_language
    - wrong_voice

  creative:
    - wrong_story
    - wrong_mood
    - wrong_message
```

---

# 25. FAILURE HANDLERS

```yaml
failure_handlers:

  face:
    strategy:
      - approved_cast_reference
      - keyframe
      - video_generation

  outfit:
    strategy:
      - approved_cast_reference
      - strengthen_cast_instruction

  pet_identity:
    strategy:
      - approved_pet_reference
      - keyframe_if_needed

  background:
    strategy:
      - simplify_environment
      - strengthen_scene_prompt
      - keyframe_if_needed
      - alternate_model_if_available

  text:
    strategy:
      - remove_text_risk_object
      - strengthen_anti_text
      - regenerate_or_post_blur

  fake_ui:
    strategy:
      - blank_screen
      - remove_ui_from_generation
      - post_overlay

  aspect_ratio:
    strategy:
      - correct_api_parameter

  missing_character:
    strategy:
      - validate_cast_mapping
      - strengthen_picture_mapping

  unwanted_dialogue:
    strategy:
      - mute_original_audio
    regeneration_required: false

  wrong_story:
    strategy:
      - revise_scene
      - regenerate
```

---

# 26. RETRY BUDGET

Không retry vô hạn.

```yaml
retry_policy:

  max_attempts_per_clip: 3

  attempt_priority:
    1: prompt_fix
    2: strategy_change
    3: keyframe_or_alternate_model

  after_max_attempts:
    status: BLOCKED
    action: require_user_decision
```

## 26.1. "Attempt" = "version" — một counter duy nhất

Để tránh lệch giữa hai con số, skill này dùng **`clip_manifest.version`** làm counter chính thức cho mọi lần generate lại một clip vì lý do chất lượng (QC fail, user feedback). Không có counter "attempt" riêng biệt.

```text
version 1 = generation attempt 1
version 2 = generation attempt 2 (sau fix loop lần 1)
version 3 = generation attempt 3 (sau fix loop lần 2)

version 4 KHÔNG được tạo tự động — khi version chạm max_attempts_per_clip (3),
clip chuyển BLOCKED và cần user quyết định (xem after_max_attempts).
```

Retry kỹ thuật (mục 20.1) **không** tăng `version` — vì đó không phải một lần generate mới về nội dung, chỉ là gọi lại API do lỗi hạ tầng.

Nếu lỗi có thể sửa bằng post-production, không regenerate chỉ vì muốn generation "hoàn hảo".

Ví dụ:

```text
unwanted T2V audio
→ mute
```

không cần regenerate video (không tăng version).

---

# 27. VERSIONING

Không overwrite clip cũ.

```text
clip_03_v001.mp4
clip_03_v002.mp4
clip_03_v003.mp4
```

Manifest:

```yaml
history:

  - version: 1
    status: FAILED
    reason: wrong_face

  - version: 2
    status: FAILED
    reason: fake_ui

  - version: 3
    status: USER_APPROVED
```

Một version chỉ trở thành `current` khi version đó PASS.

Đường dẫn file theo convention ở `clip_manifest.output` (mục 17):
`projects/{project}/{week}/renders/{clip_id}_v{version}.mp4`. Khi version được USER_APPROVED, copy sang `final/{clip_id}.mp4` (không xoá bản renders).

---

# 28. PRESERVE APPROVED STATE

Khi fix clip:

```yaml
preserve:

  - approved_scenario
  - approved_cast
  - approved_narration
  - approved_ui_assets
  - approved_logo_assets
```

Không tự thay đổi chúng.

Chỉ thay đổi khi USER yêu cầu hoặc failure handler bắt buộc.

Ngoại lệ duy nhất: mục 7.2 — nếu user yêu cầu ngừng dùng một cast là người thật, approved_cast đó bị revoke ngay cả khi đã dùng ở clip đã approved; các clip liên quan quay lại BLOCKED chờ cast thay thế.

---

# 29. POST-PRODUCTION

## Audio

```text
raw video
→ mute original audio
→ Vietnamese male TTS
→ BGM
→ final mix
```

## UI

```text
blank device screen
→ motion tracking
→ vms_mobile.png / vms_pc.png
```

## Logo

```text
final frame
→ correct logo asset
→ placement
→ sizing
```

## Subtitle

```text
Vietnamese burn-in subtitle
```

Narration rate guideline:

```text
3.5–4.5 Vietnamese words/second
```

---

# 30. FINAL QC

Chỉ chạy khi tất cả required clips đã USER APPROVED.

```yaml
final_qc:

  scenario: PASS
  cast: PASS

  clips:
    all_user_approved: true

  technical:
    aspect_ratio: PASS
    duration: PASS
    resolution: PASS

  visual:
    text: PASS
    watermark: PASS
    fake_ui: PASS
    cast_consistency: PASS

  audio:
    original_t2v_removed: PASS
    vietnamese_tts: PASS
    foreign_dialogue: PASS

  branding:
    ui_overlay: PASS
    logo_overlay: PASS

  result:
    PASS | FAIL
```

---

# 31. DELIVERY GATE

Chỉ deliver khi:

```yaml
delivery_gate:

  scenario: APPROVED
  cast: APPROVED

  every_required_clip:
    user_approval: APPROVED

  final_qc:
    status: PASS
```

## 31.1. Output structure

```text
projects/{project}/{week}/
├── scenario.yaml
├── cast_mapping.yaml
├── manifests/
│   └── {clip_id}.yaml
├── renders/
│   └── {clip_id}_v{version}.mp4      # mọi version, không xoá
├── final/
│   ├── {clip_id}.mp4                 # bản approved cuối cùng mỗi clip
│   └── final_video.mp4               # ghép toàn bộ series
└── qc_report.yaml
```

Output:

```text
final_video.mp4
individual_clip_files
scenario.yaml
clip_manifests
qc_report.yaml
```

---

# 32. AGENT BEHAVIOR

## Khi USER nói:

```text
"Tạo WEEK 12"
```

Agent:

```text
parse WEEK
→ create scenario
→ WAIT approval
```

## Khi USER nói:

```text
"OK scenario"
```

Agent:

```text
resolve cast
→ WAIT cast approval if needed (chỉ cast MỚI, xem 2.1)
```

## Khi USER nói:

```text
"Gen đi"
```

Agent:

```text
verify approval state
→ preflight
→ generate
```

## Khi USER nói:

```text
"Clip 3 sai mặt"
```

Agent:

```text
clip_3
→ failure=face
→ tie-break: high_identity_precision thắng → keyframe strategy
→ regenerate clip_3 only (version += 1)
```

## Khi USER nói:

```text
"Clip 3 có tiếng Thái"
```

Agent:

```text
mute original audio
→ no regeneration unless visual issue exists
→ version KHÔNG tăng
```

## Khi USER nói:

```text
"Clip 2 và 3 OK"
```

Agent:

```text
mark clip_2 APPROVED
mark clip_3 APPROVED
do not regenerate them
```

---

# 33. ANTI-DRIFT RULES

Agent không được:

```text
invent new cast
invent new logo
invent UI
invent narration
change approved scenario
change approved cast (trừ revoke theo 7.2/28)
regenerate PASS clips
skip USER approval
tự động retry PREFLIGHT nhiều lần không có thay đổi input (xem 2.2)
```

Agent phải:

```text
reuse approved assets
reuse approved scenario
track clip versions (counter duy nhất, xem 26.1)
track failure reason
keep manifest updated
phân biệt lỗi kỹ thuật (20.1) và lỗi chất lượng (24-26)
```

---

# 34. PROJECT-SPECIFIC DATA

WEEK-specific data **không được hard-code vào core skill**.

Không đặt trực tiếp trong SKILL.md:

```text
WEEK 10
WEEK 11
WEEK 12
specific customer
specific narration
specific cast mapping
specific campaign
```

Thay vào đó mỗi WEEK có:

```text
projects/
  week10/
    scenario.yaml
    cast_mapping.yaml
    narration.yaml

  week11/
    scenario.yaml
    cast_mapping.yaml
    narration.yaml
```

Core skill chỉ cung cấp execution rules.

---

# 35. EXAMPLE PROJECT STRUCTURE

```text
agnes-video/
│
├── SKILL.md
│
├── policies/
│   ├── anti-text.yaml
│   ├── audio.yaml
│   ├── brand.yaml
│   └── ui.yaml
│
├── projects/
│   └── week10/
│       ├── scenario.yaml
│       ├── cast_mapping.yaml
│       ├── narration.yaml
│       ├── manifests/
│       ├── renders/
│       └── final/
│
└── assets/
    ├── cast/
    ├── ui/
    └── logo/
```

---

# 36. NON-NEGOTIABLE RULES

```text
1. NO APPROVAL → NO GENERATION.
2. SCENARIO IS THE SOURCE OF TRUTH.
3. CAST MAPPING MUST MATCH SCENARIO.
4. PREFLIGHT FAIL → NO API CALL.
5. T2V TEXT IS FORBIDDEN.
6. REAL UI IS POST-PRODUCTION.
7. ORIGINAL T2V AUDIO IS DISPOSABLE.
8. REPEATED CAST → REFERENCE / KEYFRAME (identity precision thắng khi xung đột, xem 0.3).
9. USER REPORTS ONE FAILED CLIP → FIX ONE CLIP.
10. NEVER DESTROY APPROVED VERSIONS.
11. FINAL DELIVERY REQUIRES ALL GATES PASS.
12. VERSION = ATTEMPT COUNTER DUY NHẤT CHO LỖI CHẤT LƯỢNG; LỖI HẠ TẦNG RETRY RIÊNG, KHÔNG TĂNG VERSION.
13. BLOCKED KHÔNG TỰ THOÁT — CẦN INPUT MỚI HOẶC LỆNH RÕ TỪ USER.
```
