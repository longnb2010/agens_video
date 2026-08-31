---
name: agnes-video-brand
---

# Agnes Video — Brand prompt SSOT (anti-text + continuity)

Skill **brand** cho mọi clip Agnes (VMS marketing, WEEK series, Reels 9:16).

**Nguyên tắc vàng:** T2V / image ref **không** mang ngôn ngữ / UI / chữ thương hiệu — chỉ hình + ánh sáng + hành vi. Tiếng Việt = **narration + burn-in sub** (post).

**Generation mặc định VMS:**

```yaml
generation_mode:
  upload_images: true          # cast sheet + UI plate — URL public hoặc base64
  video_mode: reference        # ưu tiên giữ nhân vật; keyframe nếu 1 anchor frame
  image_mode: img2img          # Image 2.1 — multi-image composition cho keyframe
  fallback_video_mode: text    # chỉ khi không có cast ref (cảnh motion đơn lẻ)
```

---

### Bổ sung vào skill anh (Mục 0 — đầu skill)

## 0. Quy trình xử lý ảnh upload (bắt buộc mỗi phiên)

Khi có ảnh thật (upload) trong available materials, PHẢI:

Bước 1: Đọc ảnh bằng read_image_for_prompt()
  - focus: "logo, subject, outfit, scene, visible text, colors"
  - Lấy chính xác facts từ ảnh thật

Bước 2: Dùng ảnh làm reference image
  - images: [{"ref": "img_xxxx", "role": "reference"}]
  - KHÔNG chỉ dùng mã màu text thay thế cho logo thật

Bước 3: Mô tả trong prompt dựa trên facts từ ảnh đã đọc
  - VD: "VMS logo on the tablet" (từ ảnh thật)
  - VD: "woman wearing a blue scrub top with embroidery" (từ ảnh thật)

⚠️ Nếu bỏ qua → logo sai, nhân vật sai, bối cảnh sai

## 1. Vì sao cần anti-text

| Triệu chứng | Nguyên nhân | Sửa ở đâu |
|-------------|-------------|-----------|
| Chữ **Thái / Trung / Nhật** trên biển, tablet, áo | Model hallucinate chữ khi prompt có clipboard, UI, signage | **Prompt** (giảm) + **Post** blur/mosaic (bắt buộc nếu còn) |
| Chữ **không phải tiếng Việt** trên điện thoại | T2V vẽ “app giả” | **Post** overlay `vms_mobile.png` — không nhờ T2V |
| **Giọng nói tiếng Thái / ngoại ngữ** trong clip | Agnes T2V **tự sinh audio** (ambience + đôi khi lời thoại) khi cảnh có người “nói chuyện”; API **không** có flag tắt audio | **Post: mute 100% audio T2V** → chỉ TTS nam VI; **Prompt** giảm (mục 2b) |
| **Nhân vật khác mặt** giữa các clip | Mỗi `POST /v1/videos` mode `text` = cast mới | **`reference` / `keyframe`** trước gen, hoặc chấp nhận documentary |

Preflight `language_match` trong workflow VMS chỉ kiểm **narration/sub post** — **không** quét chữ trên frame hay audio T2V. Thêm bước **visual + audio QC tay** (mục 7).

---

## 2. Khối anti-text chuẩn (bắt buộc cuối mọi `t2v_prompt`)

Copy **nguyên khối** sau vào **cuối** mọi prompt English (sau mô tả cảnh):

```text
ANTI-TEXT LOCK: absolutely no readable text anywhere in the frame, no letters, no numbers, no logos, no brand names, no subtitles, no captions, no watermarks, no signage, no posters, no name tags, no embroidered text on uniforms, no writing on walls doors or glass, no Thai script, no Chinese characters, no Japanese characters, no Korean hangul, no Vietnamese diacritics, no random glyphs. All phone tablet monitor and TV screens must be completely blank or show only a smooth soft blue-white color gradient with zero UI elements and zero icons. No clipboard documents papers receipts or forms with visible print. Documentary cinematic look only.
```

**Rút gọn khi prompt dài** (vẫn giữ ý — dùng khi API giới hạn độ dài):

```text
No text letters numbers logos signage anywhere. Blank gradient screens only. No Thai Chinese Japanese script. No clipboard papers. Documentary.
```

### 2b. Khối anti-voice / anti-dialogue (bắt buộc — cảnh có người)

Agnes Video 2.5 Flash **xuất MP4 kèm audio** (doc quickstart: *natural ambience*). Khi prompt mô tả người *explaining / talking*, model hay sinh **lời thoại ngẫu nhiên** — thường nghe như **tiếng Thái** hoặc ngôn ngữ Đông Nam Á khác. **Không có** tham số API `silent: true`.

Dán **ngay trước** `ANTI-TEXT LOCK`:

```text
AUDIO LOCK: completely silent human dialogue, no spoken words, no lip-sync talking, no voiceover in any language, no Thai speech, no foreign language chatter, characters communicate only through gestures and expressions with closed or barely moving lips, optional very subtle room ambience only with no human voices.
```

**Rút gọn:**

```text
No dialogue no speech no lip-sync. Silent gestures only. No human voices in audio.
```

**Quy tắc VMS (bắt buộc post):** coi **toàn bộ audio T2V là rác** — luôn **mute** track gốc, chỉ giữ **TTS narration nam tiếng Việt** + BGM documentary (nếu cần).

```bash
# Tách video không tiếng (giữ hình)
ffmpeg -i clip_raw.mp4 -an -c:v copy clip_muted.mp4
```

---

## 3. Template prompt đầy đủ

```text
Vertical 9:16, {DURATION}s, documentary veterinary clinic in Southeast Asia, warm natural lighting, shallow depth of field, soft handheld camera.

SCENE: {SCENE_DESCRIPTION — English only, no quoted dialogue}

PEOPLE: {AGE_GENDER_ROLE — generic, no names}, {WARDROBE — plain solid colors, no text on fabric}

CAMERA: {shot type — wide / medium / close-up}, {motion — slow push-in / static / gentle pan}

MOOD: {professional / thoughtful / warm / celebratory}

{AUDIO_LOCK — section 2b if scene has people}

{ANTI_TEXT_LOCK — full block from section 2}
```

### Tham số API — Video (Agnes Video 2.5 Flash)

**Mặc định multi-clip VMS — `reference` + cast URLs:**

```json
{
  "model": "agnes-video-2.5-flash",
  "mode": "reference",
  "seconds": "8",
  "size": "720P",
  "aspect_ratio": "9:16",
  "images": [
    "https://YOUR_HOST/cast_manager.png",
    "https://YOUR_HOST/cast_reception.png"
  ],
  "prompt": "Use the person in <Picture 1> as the clinic manager and <Picture 2> as the receptionist. Vertical 9:16 documentary veterinary clinic ... {AUDIO_LOCK} {ANTI_TEXT_LOCK}"
}
```

**Keyframe** (1 anchor frame đã QC):

```json
{
  "model": "agnes-video-2.5-flash",
  "mode": "keyframe",
  "first_frame": "https://YOUR_HOST/keyframe_clip1.png",
  "seconds": "8",
  "size": "720P",
  "aspect_ratio": "9:16",
  "prompt": "<scene + locks>"
}
```

**Fallback — `text` only** (cảnh cần motion, không cần cùng cast — vd. khách dắt chó ra cửa):

```json
{
  "model": "agnes-video-2.5-flash",
  "mode": "text",
  "seconds": "8",
  "size": "720P",
  "aspect_ratio": "9:16",
  "prompt": "<template above>"
}
```

Flash: tối đa **5** ảnh `images`; không hỗ trợ `videos` reference.

### Tham số API — Image keyframe (Agnes Image 2.1 Flash)

Dùng khi cần frame tĩnh QC trước hoặc anchor cho `keyframe` video:

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "Documentary veterinary clinic, cast from reference, no text ...",
  "size": "2K",
  "ratio": "9:16",
  "extra_body": {
    "image": [
      "https://YOUR_HOST/cast_manager.png",
      "https://YOUR_HOST/cast_reception.png"
    ],
    "response_format": "url"
  }
}
```

Endpoint: `POST https://apihub.agnes-ai.com/v1/images/generations`

Rate limit video thường **1 request / phút** — gen tuần tự từng clip.

---

## 4. Vật thể — tránh vs thay thế

| Tránh trong prompt | Vì sao | Thay bằng |
|--------------------|--------|-----------|
| `clipboard`, `report`, `chart on tablet` | Kéo chữ số / bảng giả | NV **gesturing** khi nói, tay trống hoặc cầm bút không giấy |
| `tablet showing data`, `dashboard` | UI + chữ Thái | Tablet **tắt màn** hoặc **mặt sau máy**, ánh sáng phản chiếu mờ |
| `smartphone app interface`, `loyalty points screen` | App giả sai ngôn ngữ | Điện thoại **màn gradient blur** — overlay UI **post** |
| `clinic sign`, `name badge`, `logo on shirt` | Chữ brand sai | **Plain uniform**, tường trơn, logo chỉ **hình học mờ** không chữ |
| `receipt`, `prescription paper`, `form` | Chữ in hallucinate | Không đưa giấy vào frame |
| `Vietnamese`, `Vietnam text` trong prompt | Không điều khiển được ngôn ngữ chữ trên hình | Mô tả **địa lý / kiến trúc** nhẹ: `Southeast Asian urban clinic` |

---

## 5. Scene templates (VMS — copy + điền `{...}`)

### 5.1 Quản lý + lễ tân (báo cáo — không clipboard)

```text
Vertical 9:16, 8 seconds, documentary veterinary clinic, warm window light.
Male clinic manager in light blue shirt sits at wooden desk, listening attentively.
Female receptionist in plain white uniform stands beside desk, gesturing with open hands while explaining monthly performance through silent body language only, no papers no devices with screens facing camera.
Modern clean waiting area blurred in background, cinematic shallow focus.
{AUDIO_LOCK}
{ANTI_TEXT_LOCK}
```

**Narration VI (post):** *"Anh quản lý, tháng này khách mới tăng 20%. Nhưng em xem lại — tỷ lệ quay lại chỉ 30% thôi ạ."*

### 5.2 Khách trung thành / rời phòng khám

```text
Vertical 9:16, 8 seconds, documentary veterinary clinic exterior and entrance, golden hour warm light.
Middle-aged male manager brief thoughtful close-up indoors, soft bokeh.
Cut to happy woman leaving glass entrance with golden retriever on leash, relaxed body language, no visible store signs.
{ANTI_TEXT_LOCK}
```

### 5.3 Tích điểm — điện thoại (UI = post overlay)

```text
Vertical 9:16, 8 seconds, indoor clinic reception close-up, warm soft light.
Young woman holds smartphone vertically at chest level, screen shows only smooth blurred blue-white glow with no interface details.
Female staff in plain white uniform points gently toward the phone, friendly expression, focus on faces and hands.
No pets, no outdoor scene.
{ANTI_TEXT_LOCK}
```

**Post:** composite `brand/agens_video/assets/vms_mobile.png` (hoặc `.cursor/skills/agnes-vms-brand-requirements/assets/dashboard-mobile.png`) lên vùng màn hình.

### 5.4 Đổi điểm lấy quà

```text
Vertical 9:16, 8 seconds, indoor clinic front counter, warm documentary light.
Smiling receptionist in plain white uniform hands a small plain gift bag with pet treats to delighted young woman customer.
Customer's other hand holds phone with blank gradient screen facing slightly away from camera.
Focus on gift exchange and facial expressions, plain counter surface without labels.
{ANTI_TEXT_LOCK}
```

### 5.5 UI demo thuần (chỉ post — không T2V app)

Không gen màn hình app bằng T2V. Pipeline:

1. Gen **cánh tay + khung điện thoại mờ** (template 5.3), hoặc dùng stock plate.
2. Overlay PNG UI VMS + track motion nhẹ.
3. Narration VI giải thích tính năng.

---

## 6. Đồng bộ nhân vật (multi-clip) — `upload_images: true`

Mặc định VMS **bật** upload ảnh cast — không dùng `mode: text` thuần cho clip có quản lý / lễ tân lặp lại.

| Nhu cầu | Cách làm |
|---------|----------|
| Cùng quản lý / lễ tân nhiều clip | **`mode: reference`** — URL cast public trong `images` (≤5), prompt: `Use <Picture 1> as clinic manager...` |
| Anchor frame đã QC (Image 2.1) | Gen keyframe PNG → **`mode: keyframe`** + `first_frame` URL |
| Image pipeline (không motion) | Image 2.1 multi-image → ffmpeg `zoompan` / `xfade` — không audio T2V |
| Cảnh motion đơn, không cần cast | **`mode: text`** fallback — vẫn mute + TTS post |

**Cast sheet** (bắt buộc trước gen series):

| File | Mô tả |
|------|--------|
| `cast_manager.png` | Nam, áo trơn, không chữ/badge, nền PK mờ |
| `cast_reception.png` | Nữ, đồng phục trơn, không chữ |
| `vms_mobile.png` (optional) | UI plate — post overlay hoặc composition ref |

Host URL **public HTTPS** (Agnes phải fetch được) hoặc Data URI base64 trong `extra_body.image` (Image API).

```yaml
generation_mode:
  upload_images: true
  mode: reference              # video
  cast_assets:
    - cast_manager.png
    - cast_reception.png
  image_keyframes: true        # Image 2.1 khi cần QC frame trước video
```

---

## 7. Visual QC trước merge (bắt buộc — human)

Xem **từng clip** trước khi ghép:

| # | Check | Fail → |
|---|-------|--------|
| 1 | Có **bất kỳ chữ** nào đọc được (kể cả ngoại ngữ)? | Blur post **hoặc** gen lại + tăng ANTI_TEXT_LOCK |
| 2 | Màn hình điện thoại/tablet có UI giả? | Overlay UI đúng **hoặc** gen lại với `blank gradient screens only` |
| 3 | Cùng cast với clip trước? (nếu scenario yêu cầu) | Gen lại bằng `reference` |
| 4 | Tỷ lệ 9:16, 8–10s? | Regenerate đúng `aspect_ratio` |
| 5 | **Audio T2V** có giọng nói / tiếng Thái / bất kỳ lời thoại? | **Mute** (`ffmpeg -an`) — không dùng audio model |
| 6 | Narration VI đã ghi — **chưa** burn vào T2V? | OK — sub + TTS chỉ ở bước post |

Ghi kết quả QC vào scenario:

```yaml
visual_qc:
  clip_1: pass | fail_text | fail_cast
  clip_2: pass
  notes: "blur 0:03-0:04 tablet corner"
```

---

## 8. Post-production (nơi xử lý lỗi thật)

| Lỗi T2V | Công cụ | Hành động |
|---------|---------|-----------|
| Chữ Thái / rác trên biển, áo | CapCut / Premiere / ffmpeg | Mosaic hoặc blur vùng |
| UI điện thoại sai | After Effects / CapCut | Track matte + `vms_mobile.png` |
| **Giọng Thái / lời thoại T2V** | ffmpeg `-an` mute → TTS nam VI | **Bắt buộc** mọi clip VMS |
| Ngôn ngữ đúng cho audience | TTS nam VI + burn-in sub | Bước 7 workflow VMS |
| Nhạc / pacing | Merge clips + BGM documentary nhẹ | Fade 0.3s giữa clip |

**Narration:** 3.5–4.5 chữ/giây · giọng **nam** · tiếng **Việt** · hashtag chỉ ở **caption**, không T2V.

---

## 9. Checklist agent (trước mỗi lần gọi API)

- [ ] `upload_images: true` — cast sheet URL public đã sẵn sàng
- [ ] Clip có NV lặp → **`mode: reference`** hoặc **keyframe** (không `text` thuần)
- [ ] Prompt **English only** — không tiếng Việt có dấu, không `"VMS"` / `"Thiên Long Trí"` trong prompt
- [ ] Cảnh có người → có **AUDIO_LOCK** (silent dialogue) — video T2V vẫn **mute post**
- [ ] Cuối prompt có **ANTI_TEXT_LOCK** (full hoặc rút gọn)
- [ ] Không clipboard / dashboard / signage / badge chữ trong SCENE
- [ ] Điện thoại = **blank gradient** hoặc **composite ref** `vms_mobile.png`; UI thật = **post overlay**
- [ ] `narration_vi` tách riêng — không nhét vào prompt
- [ ] Sau gen: **visual QC** mục 7 trước khi báo deliver

---

## 10. Ví dụ JSON scenario (1 clip — reference + upload)

```json
{
  "clip_id": "week10_clip1_manager",
  "generation_mode": {
    "upload_images": true,
    "mode": "reference"
  },
  "cast_assets": [
    "https://YOUR_HOST/cast_manager.png",
    "https://YOUR_HOST/cast_reception.png"
  ],
  "agnes": {
    "model": "agnes-video-2.5-flash",
    "mode": "reference",
    "seconds": "8",
    "size": "720P",
    "aspect_ratio": "9:16",
    "images": [
      "https://YOUR_HOST/cast_manager.png",
      "https://YOUR_HOST/cast_reception.png"
    ]
  },
  "narration_vi": "Anh quản lý, tháng này khách mới tăng 20%. Nhưng em xem lại — tỷ lệ quay lại chỉ 30% thôi ạ.",
  "t2v_prompt": "Use the person in <Picture 1> as the clinic manager and <Picture 2> as the receptionist. Vertical 9:16, 8 seconds, documentary veterinary clinic, warm window light. Manager sits at wooden desk listening. Receptionist gestures with open hands, silent body language, no papers no screen UI. Shallow focus. AUDIO LOCK: no dialogue no speech. ANTI-TEXT LOCK: no readable text, blank screens only, no Thai Chinese Japanese script, documentary only."
}
```

---

Mục 11 — Hướng dẫn sử dụng 6 image brand assets trong skill:

4 logo variants (logo.png, Logo-Darkmode, Logo-Lightmode, Logo-Lightmode-Blue Background) — cách chọn theo nền, overlay góc phải dưới, scale
2 VMS UI plates (vms_mobile.png, vms_pc.png) — pipeline overlay post-production, track màn hình, motion tracking
Pipeline đề xuất từ T2V → QC → post-production
Lưu ý: không dùng UI plate làm reference cho video, ưu tiên post overlay

Mục 11.1 — Brand Identity:
Primary: #0066CC — main brand
Secondary: #23C1F5 — accent
Quy tắc dùng màu trong post-production (overlay, gradient, text color)
Typography khuyến nghị cho burn-in sub / caption (Roboto, Inter, Be Vietnam Pro)
Mục 11.2 — Logo assets (giữ nguyên) 
Mục 11.3 — VMS UI plates (giữ nguyên) 
Mục 11.4 — Pipeline đề xuất (giữ nguyên) 
Mục 11.5 — Lưu ý quan trọng (giữ nguyên)
