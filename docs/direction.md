# OsbMpeg: hướng phát triển tiếp theo

Tài liệu này là nguồn cho **lý do**: vì sao bản v1 hỏng, osu! storyboard thực sự biểu diễn được gì,
không gian representation, chiến lược phân rã, các hướng đã bác bỏ kèm điều kiện đảo, prior art, và
lịch sử đo đạc. Nó không chứa bước triển khai. Ba tài liệu của dự án và vai trò của chúng:

| Tài liệu | Trả lời câu hỏi | Khi mâu thuẫn |
| --- | --- | --- |
| `docs/direction.md` (file này) | vì sao, sự thật về osu!, không gian lựa chọn, đã đo được gì | sự thật và số đo ở đây là nguồn |
| `docs/implementation-plan.md` | xây gì, theo thứ tự nào, với API và acceptance nào; mọi quyết định đã chốt (D1 đến D18), invariants, lộ trình từ Phase A tới M3.5 | quyết định thi công ở đó là nguồn; nếu nó đòi một sự thật khác với file này thì sửa file này, không override |
| `docs/feasibility-plan.md` | kiểm chứng gì trước khi viết code sản phẩm (spike ném đi, tiêu chí pass/fail chốt trước) | |

Người triển khai đọc `implementation-plan.md` trước, quay lại đây khi cần lý do. Gặp điểm tài liệu không
nói rõ và không suy ra được từ invariants: hỏi người dùng, đừng tự phát minh kiến trúc. Các điểm đánh
dấu `OQ-n` (mục 9) là câu hỏi mở đã có sẵn phạm vi spike an toàn. Lịch sử đo đạc nằm ở mục 11.

Ký hiệu phân loại quyết định dùng xuyên suốt:

* `MUST`: đủ bằng chứng, triển khai theo spec.
* `INVESTIGATE`: đáng nghiên cứu, chưa khóa, có phạm vi spike.
* `DEFER`: chỉ quyết sau khi có số liệu được nêu tên.
* `REJECT`: có bằng chứng cho thấy không đáng làm, kèm điều kiện đảo quyết định.

---

## 1. Bản cài đặt hiện tại hỏng ở đâu

Hai triệu chứng: output mờ, và output nặng. Cả hai đều có nguyên nhân xác định được trong code, không
phải chuyện tinh chỉnh tham số.

### 1.1 Vì sao mờ

Sàn chất lượng là số tương đối. `ParameterTuner.cs:55` đặt `TargetSlackDb = 1.0`, sàn được tính bằng
PSNR của combo mặc định trừ đi 1dB. Trên nội dung khó (bad_apple có baseline khoảng 24.9dB) sàn rơi
xuống 23.9dB, nghĩa là ảnh nát vẫn "đạt". Tệ hơn, tuner tối thiểu hóa bytes *xuống sát sàn đó*, nên
chất lượng thấp là mục tiêu tối ưu chứ không phải tai nạn.

Palette quantize không dither. `AssetStore.cs:232` gọi `OctreeQuantizer(MaxColors = Colors)` và tuner
từng chọn `Colors=16` cho short_animation, tạo posterization trên gradient.

Vòng lặp hở. `TileRunTracker` chỉ đóng run khi hash *đã lượng tử hóa* thay đổi, cộng thêm ngân sách
MAE của `TileTolerance`. Thay đổi chậm như dissolve, gradient trôi, hay ánh sáng đổi dần luôn nằm
dưới ngưỡng, nên tile đứng hình cũ vô hạn. Không có gì đo error trên frame thực tế ship ra: PSNR chỉ
được đo lúc tuning, trung bình trên một sample window ngắn, nên staleness cục bộ không bao giờ bị
bắt. `Metrics.Ssim` cũng là SSIM toàn frame, chính doc comment của nó đã ghi rằng một tile sai trong
một frame đẹp gần như không làm số đó nhúc nhích.

### 1.2 Vì sao nặng

Chỉ có đúng một primitive. `TileEncodeLoop.Emit` phát ra mỗi run thành một sprite kèm một lệnh `S`
tĩnh để đặt scale. Không có lệnh động nào (`M`, `F`, `C`, `R`) được dùng để mô tả nội dung. Camera
pan làm mọi tile đổi mỗi frame, và mỗi tile đổi lại phải re-emit ở độ phân giải nguồn. Con số thực
đo được trên Minecraft full clip 60fps: 79.612 sprites, 79.223 assets, 735,01 MB assets cộng 9,31 MB
`.osb`, tổng 744,32 MB, thời gian compile 542 phút.

Animation không dedupe được (format cần chuỗi file đánh số riêng cho mỗi object) và có thể lên tới
300 frame PNG full-res cho một object. Mọi frame đó nằm trong RAM cùng lúc khi osu! load.

Yêu cầu 60fps qua frame duplication nhân đôi mật độ biên run.

### 1.3 Những gì đang đúng và cần giữ

Decode ở độ phân giải nguồn, và `CanvasMapping` khớp đúng phép toán của `DrawableStoryboard`. Đây là
chiến lược sharp duy nhất (xem R2 ở mục 3), nên độ mờ hiện tại không đến từ resolution.

Tầng `OsbMpeg.Parsers` hoàn chỉnh: IR, `OsbWriter` (float, shorthand, loop), `CommandEvaluator` và
`EasingTable` phủ đủ easing 0 đến 34, các pass merge/drop/loop-extract.

`Compositor` có blend semantics đã verify khớp osu-framework: normal blend là straight alpha lerp,
additive là cộng.

`AssetStore` content-addressed XXH3-128, chế độ in-memory, dedupe xuyên scene. `FrameSource` streaming
kèm hwaccel, `MediaProbe`, `Metrics.Psnr`.

Scene detection của v1 (`ScenePrePass`, `SceneBounds`) có tín hiệu đúng (gần như mọi tile đóng run cùng
lúc) và ngưỡng cut đã đo (mục 11.6). Thiết kế mới không giữ nó như một pass riêng, vì closed loop tự
thấy hard cut bằng chính tín hiệu đó (gần như mọi block vi phạm cùng lúc); ngưỡng được tái dùng. Tương
tự, `.osbv`, CLI, tuner và shared decode nhiều object không còn trong sản phẩm (D4, D8 của
`implementation-plan.md`): sản phẩm là một thư viện .NET nhận một video và trả object storyboard.

---

## 2. Đích đến và ràng buộc

Coi OsbMpeg về lâu dài là một compiler đi tìm representation storyboard hiệu quả nhất cho một video,
dưới ràng buộc cứng về chất lượng và về tải renderer.

```text
Video
  ↓
Phân tích scene / thời gian
  ↓
Phân rã thành thành phần
  ↓
Sinh candidate representation
  ↓
Chọn representation
  ↓
Storyboard IR
  ↓
Dựng lại (reconstruction)
  ↓
Kiểm chứng chất lượng và workload
  ↓
Storyboard tối ưu
```

Đây là đích đến, không phải kiến trúc đã khóa. Không kỹ thuật cụ thể nào (object tracking, optical
flow, depth estimation, panorama, RDO) được giả định trước là đáp án.

### 2.1 Ba mục tiêu, đo bằng metric cụ thể

| Mục tiêu | Metric | Cách đo |
| --- | --- | --- |
| Giữ chất lượng | 100% frame đạt PSNR ≥ P và worst-tile SSIM ≥ S theo preset | verify pass: render toàn bộ output ở độ phân giải nguồn, so từng frame với source |
| Gọn nhẹ | asset bytes, `.osb` bytes, asset count | `WorkloadAnalyzer` |
| Không quá tải renderer | peak SB load, peak alive sprites, animation resident bytes, `.osb` line count | `WorkloadAnalyzer` quét timeline |

Khi các mục tiêu xung đột thì fidelity thắng. Ngân sách workload bị vượt: đổi representation (crossfade
dài hơn, merge patch, đổi encoding, keyframe thưa hơn), không bao giờ âm thầm hạ sàn. Vẫn vượt thì in
violation report kèm gợi ý hạ preset và để người dùng quyết.

### 2.2 Hàm mục tiêu và ràng buộc

Thứ cần tối thiểu hóa:

```text
minimize   AssetBytes + OsbBytes
```

Asset bytes trội hơn hẳn trong thực tế (735 MB so với 9,31 MB trên minecraft), nên trọng số giữa hai
số này không thành vấn đề.

Các ràng buộc cứng, không phải trọng số, vì giới hạn renderer là vách đá chứ không phải dốc:

```text
∀ frame:      PSNR ≥ preset.P  và  worst-tile SSIM ≥ preset.S     (fidelity, tối thượng)
∀ t:          SteadySbLoad(t) ≤ 4.0 viewport
              TransientSbLoad(t) ≤ +2.0 (cửa sổ fade)
              AliveSprites(t) ≤ 500
Σ animation:  ResidentBytes ≤ 200MB mỗi scene
∀ asset:      chiều ≤ 1024 (ưu tiên, để giữ atlas), ≤ 3840 (tuyệt đối)
.osb:         ≤ ~1M dòng, ≤ ~10MB mỗi phút video (mềm, chỉ cảnh báo)
```

Không cộng có trọng số các trục workload lại với nhau. Một storyboard "trung bình tốt" nhưng vượt
fill-rate ở một đoạn vẫn lag đúng ở đoạn đó, và peak mới là thứ người chơi cảm nhận.

### 2.3 Quality preset

Gate gồm PSNR tối thiểu mỗi frame và worst-tile SSIM (tile 64×64 luma). Global SSIM chỉ in trong
report, không gate, vì nó là aggregate kiểu trung bình, đúng loại mù cục bộ đã gây ra vấn đề ở 1.1.

| Preset | PSNR tối thiểu mỗi frame | Worst-tile SSIM |
| --- | --- | --- |
| `high` (mặc định) | 38 dB | 0,92 |
| `medium` | 34 dB | 0,88 |
| `low` | 30 dB | 0,80 |

Các số này là placeholder và được chốt bởi spike calibration M0.5 (`implementation-plan.md`), không phải đoán tay. Dữ
liệu lịch sử cảnh báo trước: bad_apple có baseline khoảng 24,9dB *trung bình*, nên PSNR *tối thiểu*
34dB trên nội dung nhiễu có thể đòi patch rate gần bằng full refresh. Spike sẽ đo đường cong sàn so
với patch rate trên từng fixture rồi đặt số, kể cả khả năng preset dùng tileSSIM làm gate chính và
PSNR thấp hơn nhiều so với trực giác.

Preset được chọn qua `CompileOptions.Quality`; không có CLI.

### 2.4 Ưu tiên nội dung và dự đoán theo loại nội dung

Ưu tiên trong 6 tháng tới là nội dung được author: motion graphics, UI, 2D animation (D14). Game render
và footage thật không được tệ đi (verify pass, bytes không tăng) nhưng không phải mục tiêu tối ưu.
Dự đoán theo loại nội dung, dùng để đặt fixture và keep-gate:

| Nội dung | Fixture | Dự đoán sau M2.5 | Lợi ích đến từ |
| --- | --- | --- | --- |
| Motion graphics, UI game, logo, text | tier `authored` (generator, Crello Animation, Animation2Code) | giảm sprite và asset 1 đến 2 bậc độ lớn; easing khớp chính xác; nhiều vùng 0 byte | layer Sim(2), easing rời rạc, canonical asset |
| 2D animation sạch | `medium/fish-short` | giảm mạnh ở phần rigid, phần biến dạng (mặt, tay) rơi về crossfade hoặc patch | layer, canonical reuse, crossfade |
| Game render 3D có camera | `hard/minecraft` | phần lớn lợi ích từ K=1 và panorama; foreground có parallax rơi về patch | global motion |
| Footage thật, nhiễu | `hard/birdbrain`, `medium/fish` | gần 0; nhiễu là ranh giới vật lý (mục 7) | không có |
| Silhouette nhấp nháy | `medium/badapple` | như Animation có cap | không có |

---

## 3. osu! storyboard thực sự biểu diễn được gì

Mười sự kiện quyết định thiết kế, trích từ osu!wiki cùng source `ppy/osu` và `ppy/osu-framework`.
Encoder bắt buộc tôn trọng.

**R1. Command nội suy liên tục.** Mỗi command trở thành một `Transform` của osu!framework, được đánh
giá như hàm liên tục của thời gian trong mỗi `Update()`, tại refresh rate thực của màn hình. Chuyển
động qua `M` hay `S` không cần một command mỗi frame; vài keyframe linear là mượt tuyệt đối, mượt hơn
cả fps của nguồn.

**R2. Scale là float tự do, lọc bilinear kèm mipmap (tối đa 3 mức).** Phát asset ở độ phân giải nguồn
rồi scale xuống không gian 640 cho ra hình sắc nét trên mọi màn hình. Tỷ lệ từ storyboard-space sang
screen là `screen_height/480`, tức 2,25× ở 1080p, 3× ở 1440p, 4,5× ở 2160p. Việc author asset ở độ
phân giải 640 rồi để osu! phóng lên mới là nguyên nhân mờ, không phải giới hạn của renderer.

**R3. Mô hình chi phí của renderer.** Chi phí mỗi frame xấp xỉ fill-rate và overdraw (diện tích các
sprite đang sống; texel trong suốt bên trong một quad opaque-alpha vẫn tốn) nhân với số texture bind.
Sprite nằm ngoài lifetime tốn gần như bằng không vì bị bỏ qua cả `Update` lẫn `Draw`. Sprite alpha=0
giữa lifetime thì vẫn sống và vẫn phải duyệt transform. Tổng số sprite và command trên toàn timeline
chủ yếu là chi phí parse và memory. Thiết kế time-disjoint một sprite mỗi vùng đã tối ưu chi phí vẽ.

**R4. Frame của Animation được preload toàn bộ, không stream.** N frame nghĩa là N texture nằm trong
memory cùng lúc. Frame count của Animation là trục đắt thật sự về memory, khác hẳn command count.

**R5. Command cùng property chồng thời gian được stable và lazer xử lý khác nhau, vĩnh viễn**
(ppy/osu#7257, maintainer xác nhận đây là thiết kế có chủ đích của lazer chứ không phải bug cần sửa
cho khớp stable). Mọi track theo property phải rời nhau (`next.start == prev.end`). Property thật sự
là X, Y, Scale, VectorScaleX, VectorScaleY, Rotation, Alpha, Colour, Blending, các flip. Theo lazer
(chuẩn verify, D1), `S` và `V` là hai property riêng và được nhân với nhau (`Scale × VectorScale`); chỉ
trên stable chúng mới xung đột, nên validator cảnh báo khi cả hai cùng sống trên một object thay vì
fail. `M` đụng `MX` và `MY`.

**R6. Atlas 1024×1024.** Asset từ 1024² trở xuống (trừ padding) được gộp chung atlas và batch được.
Lớn hơn vẫn render tới `MaxTextureSize` khoảng 4096 nhưng mất batching. Ranking criteria giới hạn
17 triệu px² mỗi ảnh.

**R7. Lifetime.** `StoryboardSprite.StartTime` tự suy ra từ điểm alpha lớn hơn 0 đầu tiên, nên đệm
alpha=0 ở đầu là miễn phí. Không có cơ chế đối xứng ở cuối, nên encoder phải kết thúc mọi sprite bằng
command chấm dứt đúng thời điểm nó hết nhiệm vụ.

**R8. Widescreen là key `WidescreenStoryboard: 1` trong file `.osu`, không phải trong `.osb`.** Một
compiler chỉ phát `.osb` không thể tự bật canvas rộng 853,33. Đây là yêu cầu đóng gói, phải ghi rõ
trong README và trong output note.

**R9. Không bao giờ phát `[Variables]`, và không dùng value-chaining shorthand.** Một case thật với
554 biến biến thời gian load từ khoảng 3 giây thành 45 giây. Value-chaining (một dòng command nhiều cặp
giá trị) chỉ stable expand; `LegacyStoryboardDecoder` của lazer đọc đúng cặp đầu và bỏ phần còn lại
[deepwiki], nên phát ra là sai trên chuẩn verify (D1).

**R10. Quy mô đã được chứng minh.** Một storyboard 34.847 sprite, 977 nghìn dòng, `.osb` 27 MB vẫn
load khoảng 3 giây và chạy được. Không có bức tường về số sprite; hiệu năng suy giảm mềm. `.osb` trên
10 MB bắt đầu có nhược điểm.

---

## 4. Không gian representation

### 4.1 Danh mục những gì storyboard biểu diễn được

| # | Representation | Cấu tạo | Bytes | Workload | Thắng trên |
| --- | --- | --- | --- | --- | --- |
| T1 | StaticPatch | 1 sprite, asset crop, span [t0,t1) | 1 asset | 1 alive, fill = diện tích | nội dung tĩnh |
| T2 | PatchSequence | chuỗi T1 rời nhau cùng vùng | N assets | 1 alive | fallback vạn năng (chính là bản hiện tại) |
| T3 | Crossfade | 2 sprite chồng, `F` 0→1 trên cái mới | 2 assets mỗi cặp | 2 alive trong fade | thay đổi chậm: dissolve, ánh sáng, biến dạng chậm |
| T4 | TransformTrack | 1 sprite kèm keyframe `MX`/`MY`/`S`/`V`/`R` | 1 asset cộng ít text | 1 alive | chuyển động rigid, nội suy liên tục theo R1 |
| T5 | MotionLayerSet | K sprite T4, xếp theo z | K assets | K alive | parallax, tách foreground khỏi nền |
| T6 | Panorama | asset lớn cắt thành strip, chung một track M | xấp xỉ hợp nội dung | strip alive, fill chỉ phần trên màn | camera pan hoặc scroll dài |
| T7 | Animation | flipbook, frameDelay | N frame, không dedupe | N texture resident (R4) | thrash thật sự |
| T8 | Photometric | `F` hoặc `C` trên sprite sẵn có | gần 0 | 0 mới | fade in/out, tint, đổi sáng cả vùng |
| T9 | AdditiveLayer | sprite kèm `P,A` | 1 asset | +1 alive | glow, flash, lửa; thứ duy nhất làm sáng thêm được vì `C` chỉ nhân tối đi |
| T10 | SpriteReuse | 1 asset, nhiều instance khác transform và thời gian | 1 asset | bằng số instance | motif lặp: particle giống nhau, UI, texture lặp |
| T11 | SolidColor | asset đơn sắc nhỏ scale to | gần 0 | fill theo vùng | letterbox, fade to black, nền phẳng |
| T12 | AlphaMatteSprite | PNG có alpha, mép feather | 1 asset | fill tính cả quad, kể cả texel trong suốt (R3) | mép mềm cho foreground layer, bắt buộc để layer không lộ viền chữ nhật |
| T13 | FlipReuse | `P,H` hoặc `P,V` | 0 | 0 | nội dung đối xứng, hiếm nhưng gần như miễn phí để thử |

### 4.2 Giới hạn cứng của không gian này

Phần tuyến tính của transform trong osu!framework là `R(θ)·diag(sx,sy)`, tức rotation sau scale, một
lần. Đó là 5 tham số và **thiếu shear** so với affine 6 tham số (phân tích SVD `A = R1·Σ·R2` cần
rotation cả trước lẫn sau scale). Hệ quả:

* Không có shear, không có perspective hay homography cho một sprite. Cách xấp xỉ là chia thành các
  strip, mỗi strip một similarity. Chi phí tuyến tính theo số strip, sai số cỡ bình phương chiều cao
  strip nhân độ cong của perspective.
* Không mask hay clip tùy ý. Wipe transition may mắn biểu diễn được bằng sprite của scene B trượt vào
  với mép cứng, nhưng iris hay wipe theo hình thì không.
* `C` là tint đều cho cả sprite, không có gradient tint. Gradient phải nằm sẵn trong asset.
* Không có primitive blur. Đổi focus phải crossfade sang asset đã blur sẵn, tức tốn thêm asset.
* Không có particle system. Mỗi hạt là một sprite; T10 giúp tiết kiệm asset chứ không giúp sprite count.
* `C` chỉ nhân nên chỉ làm tối. Muốn sáng hơn ảnh gốc phải dùng T9.
* Alpha theo từng pixel bên trong asset thì có (PNG), nhưng fill-rate vẫn tính cả texel trong suốt
  theo R3, nên rect phải bó sát và feather phải mỏng.

### 4.3 Các tổ hợp đáng chú ý

T3 lồng T4 (crossfade giữa hai TransformTrack) xấp xỉ được "ngoại hình đổi trong khi đang di chuyển",
tức morph thô. Ca này quan trọng vì biến dạng chậm cộng chuyển động là chuyện thường gặp, ví dụ người
đi bộ ở xa camera.

T6 làm layer đáy cho T5 phủ được ca pan kèm foreground độc lập. T8 áp lên mọi thứ nên fade cả scene
không tốn asset nào. T2 là tầng đáy phủ mọi sai số của các tầng trên, và chính điều đó biến mọi tầng
trên thành optimization thuần: chúng sai thì tốn thêm chứ không tạo artifact.

---

## 5. Chiến lược phân rã và tối ưu

### 5.1 Khung thống nhất: layered similarity motion

Một khung duy nhất bao trùm gần hết các hướng thường được đề xuất. Bài toán: gán mỗi block 16×16 của
mỗi frame vào một trong K motion model similarity, cộng phần dư. Thuật toán cổ điển, không cần ML:

```text
flow theo block (block matching pyramidal)
   → sequential RANSAC: fit model trội nhất, loại inlier, fit tiếp (K ≤ 4)
   → lọc theo tính liên thông không gian: layer phải là blob liền ≥ diện tích tối thiểu
   → thời gian: gán của frame trước làm prior cho frame sau
```

K=1 chính là global camera motion. "Entity" là layer có support nhỏ. Đây cũng là định nghĩa làm việc
duy nhất về "thực thể" trong dự án (D12): một *thay đổi liên kết*, tức vùng không-thời gian lớn nhất mà
một primitive trong ladder giải thích được trong sàn. Không có object ngữ nghĩa, không có prompt; sprite
là hệ quả của việc một track giải thích được một vùng, không phải đầu vào. Parallax là các layer khác biên
độ translation. Panorama là layer tích lũy theo thời gian. Occlusion là hệ quả của z-order giữa các
layer. Nghĩa là không phải chọn giữa object tracking, motion layers hay depth: chúng là một trục K và
một trục lifetime của cùng cơ chế. Vì vậy phần motion phải được xây dưới interface layer-set ngay từ
đầu, dù bản đầu chỉ dùng K=1.

### 5.2 So sánh các chiến lược phân rã

| Chiến lược | Thắng trên | Chi phí compute | Rủi ro | Kết luận |
| --- | --- | --- | --- | --- |
| Global motion (K=1) | pan và zoom camera, chiếm đa số footage game và cinematic | thấp | có gate hai tầng | MUST (M2) |
| Motion layers (K≤4) | parallax, tách foreground; nội dung được author vốn là nhiều object độc lập | trung bình | layer phân mảnh do noise; mép layer cần alpha matte T12 | MUST (M2.5) với keep-gate trên tier `authored` |
| Object tracking ngữ nghĩa (SAM 2/3 và tương đương) | không rõ | cao, GPU | output là mask hoặc box, storyboard chỉ tiêu được rect kèm similarity; tracker mất ID qua che khuất dài | REJECT làm xương sống (5.6, D12); chỉ còn là bộ tinh chỉnh biên tùy chọn trong sidecar, có gate (D13) |
| Depth estimation | thứ tự z cho layer | cao | z-order suy được từ chính occlusion của motion layers | REJECT cho core |
| Panorama, mosaic | pan dài, scroll | trung bình | mover nhiễm vào stitch (median cộng mask xử lý được); giới hạn texture; parallax trong nền phá stitch | DEFER tới checkpoint |
| Suy luận occlusion | không rõ | gần 0 | không | byproduct, không phải feature (5.3) |
| Photometric | fade, flash | gần 0 | ít | MUST (M1) |
| Residual patches | mọi thứ còn lại | không rõ | không | tầng đáy vĩnh viễn |

### 5.3 Occlusion là hệ quả, không phải feature

Với layers, z-order và closed loop, mọi yêu cầu về occlusion tự thỏa. "A che B nên B không cần update"
đúng theo nghĩa đen: closed loop đo error trên composite, vùng B bị che không đóng góp error nên
không patch nào được chi. "A biến mất, B vẫn tồn tại": sprite B có lifetime dài, và nội dung vùng vừa
lộ ra được verify ngay tại frame nó lộ, vì `ErrorMap` chạy sau `AdvanceTo`. Nội dung đúng thì tốn 0,
chưa từng thấy thì patch, đó là điều đúng phải làm.

Thứ duy nhất cần chủ động là kéo dài lifetime của layer bị che thay vì đóng nó, tức giữ sprite sống
qua occlusion nếu dự đoán sẽ lộ lại và nội dung còn đúng. Đó là một heuristic nhỏ trong bộ chọn
representation, không phải một hệ thống.

### 5.4 Định danh entity chuyển thành tái dùng asset chuẩn hóa

Lợi ích thật của "cùng một entity" là tái dùng asset, và có đường rẻ hơn tracking nhiều. `AssetStore`
content-hash đã là re-identification miễn phí cho nội dung trùng byte, xuyên scene và xuyên file. Mở
rộng tự nhiên là chuẩn hóa trước khi hash: đưa patch về pose chuẩn (scale về bậc lượng tử hóa, khử
rotation nếu dùng) rồi hash, còn instance thì đặt lại bằng `M`, `S`, `R` của sprite. Cùng một ngôi sao
ở 5 vị trí và 3 cỡ trở thành 1 asset kèm 5 instance, mà vẫn là exact-hash nên không cần fuzzy search.

Near-duplicate thật sự thì tra ứng viên bằng coarse hash rồi verify bằng SSIM so với sàn. Cách này
bounded và an toàn với sàn cứng, nhưng để dành tới checkpoint vì chỉ đáng khi attribution chỉ ra
bucket này lớn.

### 5.5 Panorama: số học khả thi

Pan 10 giây với 100 osu!px mỗi giây trên nền 1080p cho panorama khoảng 2920×1080. Vượt 1024 nên mất
atlas, nhưng vẫn dưới 4096. Cắt thành strip rộng tối đa 1024, tất cả strip chung một trajectory để
breakpoint đồng bộ. Strip ngoài màn bị GPU clip nên fill gần 0, chỉ tốn một ít alive count. Pan dài
tùy ý chỉ là thêm strip, không chạm `MaxTextureSize`.

Stitch bằng cách warp các frame về hệ panorama qua chính global transform đã có, tích lũy median theo
từng pixel để loại mover, kèm confidence mask. Pixel chưa từng thấy được lấp bằng khuếch tán, confidence 0, không bao giờ được tin; closed loop sẽ patch khi
nó lộ. Không để trong suốt, vì ở mép alpha mềm của layer phía trên lỗ sẽ lộ ra và vi phạm yêu cầu phủ kín. Asset bytes xấp xỉ hợp nội dung, tức cận dưới lý thuyết của mọi cách encode pan.
Điều kiện nhận: scene đã accept K=1 và residual thấp, nghĩa là nền thực sự rigid.

### 5.6 Vì sao không dùng semantic detection và learned model

Lý do là lệch độ phân giải quyết định. SAM, RAFT hay depth-anything cho mask, flow và depth chính xác
tới từng pixel, nhưng storyboard chỉ tiêu được rect kèm similarity kèm alpha. Phần chính xác vượt mức
rect là thông tin bị vứt đi, trong khi giá phải trả là dependency ONNX runtime, model hàng trăm MB,
inference mỗi frame và GPU. Block-level classical (block matching cộng RANSAC) khớp đúng độ phân giải
mà representation quyết định được.

Cửa quay lại, ghi rõ để khỏi tranh cãi lại từ đầu: nếu checkpoint cho thấy motion layers fail chủ yếu
vì chất lượng flow trên nội dung khó, thử DIS flow (classical, nhẹ) trước, learned flow sau cùng.

Tương tự, "camera cộng 4 motion layer cộng patches" thắng "camera cộng 37 object ngữ nghĩa", vì 37
object vẫn phải nắn về rect similarity trước khi emit, và khi đó chúng thành 37 layer chất lượng thấp
hơn 4 layer fit thẳng trên flow.

Chỗ duy nhất một model học còn có việc là *biên* của layer đã được motion đề xuất: alpha matte trong dải
biên vài pixel, nơi classical (guided filter, trapped-ball) yếu trên footage có texture. Vì vậy D13 đặt
nó dưới contract `ILayerBoundaryRefiner`, cài trong tiến trình Python riêng, chỉ xây khi attribution
(M3.5) đo thấy bucket "mép layer" đủ lớn, và chọn model theo thời điểm đó (SAM 2, SAM 3, MatAnyone hay
thứ kế tiếp) vì contract chỉ nói bằng mask và alpha, không nói bằng model. Trên nội dung ưu tiên (biên
màu phẳng của motion graphics và 2D animation) khả năng gate này không bao giờ mở là cao.

### 5.7 Bốn tầng tối ưu

Không gian tìm kiếm phân rã tự nhiên thành bốn tầng gần độc lập:

```text
(1) Phân rã mỗi scene:        chọn tập layer (K model, có panorama không, ranh giới thời gian)
(2) Representation mỗi vùng:  chọn T1..T13 cho từng dirty region
(3) Keyframe mỗi track:       nén trajectory và span crossfade
(4) Encoding mỗi asset:       PNG, PNG palette có dither, JPEG
```

| Tầng | Thuật toán | Vì sao không cần nặng hơn |
| --- | --- | --- |
| (1) | matching pursuit: thêm layer hoặc panorama khi lợi ích biên đo được lớn hơn chi phí biên | số ứng viên nhỏ (K ≤ 4, panorama có hoặc không); lợi ích đo rẻ trên bản downscale; sai thì closed loop trả bằng patch chứ không bằng artifact |
| (2) | ladder: thử từ rẻ tới đắt, nhận cái đầu tiên đạt sàn | fidelity là ràng buộc cứng nên bài toán là min cost có ràng buộc, không phải đánh đổi theo λ; chi phí các bậc cách nhau hàng bậc độ lớn nên greedy xấp xỉ tối ưu |
| (3) | RDP joint trong screen space cho track; mở rộng greedy cho span crossfade | error đơn điệu theo span nên greedy tối ưu |
| (4) | thử các encoding, verify SSIM, chọn bytes nhỏ nhất | độc lập hoàn toàn theo từng asset nên exhaustive là đúng nghĩa đen |

Bác bỏ (đến khi đo được greedy bỏ phí hơn 10 đến 15% trên corpus): global graph optimization, beam
search, DP toàn cục, RDO quét λ. Lý do chung: interaction giữa các quyết định đã bị chặn bởi kiến
trúc phân tầng, vì tầng patch đáy hấp thụ mọi hệ quả xấu thành *chi phí đo được*, nghĩa là greedy
nhìn thấy giá thật của mình ngay. Còn mọi phương pháp toàn cục đều cần cost model *dự đoán* thay vì
*đo*, và đó là nguồn sai mới. Bài học có sẵn: spike heatmap partitioning từng cho ước lượng đẹp nhưng
đo thật thua 2,6 đến 5,4 lần (mục 11.4).

Closed loop trong bức tranh này là cơ chế đúng đắn và là máy đo chi phí thật cho mọi tầng trên. Trí
tuệ nằm ở bốn tầng; closed loop bảo đảm mọi trí tuệ đó không thể sai thành artifact, chỉ có thể sai
thành tốn.


### 5.8 Vật thể di chuyển làm lộ nền: vì sao bắt đầu từ rect nhân quả

Ca điển hình: X đứng trước nền, che một phần, rồi đi chỗ khác làm lộ dần phần nền bị che. Trực giác gợi ý
phải tách X thành sprite RGBA rồi dựng lại nền. Nhưng phần "phục hồi nền vừa lộ" closed loop đã làm sẵn: vùng
X vừa rời hiển thị sai, bị đánh dấu bẩn và được patch cắt từ frame hiện tại, đúng cả khi X đi nhanh, bị che
hay đổi hình. Thứ closed loop *không* tự có là biểu diễn chính X bằng một sprite di chuyển.

| Cách | Asset cho X rộng 200 px đi 5 px mỗi frame trong 60 frame | Cần thêm gì |
| --- | --- | --- |
| Chỉ patch (M1) | khoảng 60 patch cỡ X cộng dải lộ | không |
| Rect nhân quả (M2.5a) | 1 sprite X cộng khoảng 60 patch dải hẹp | quỹ đạo Sim(2), display model có transform, chèn patch dưới X |
| Phân rã có dựng nền (M2.5b) | 1 nền cộng 1 X | pass full-res tích lũy, mask và alpha, khử lem màu |

Rect nhân quả lấy được phần lớn khoản tiết kiệm so với M1 với một phần nhỏ độ phức tạp. Phần còn lại, chuỗi
dải lộ và quầng nền bị kéo theo trong rect khi nền có texture, được đo bằng hai bucket `rect-halo` và
`reveal-strip`, và chỉ khi chúng lớn mới xây nấc b (I9). Rect không mask an toàn về chất lượng, vì phần nền
sai trong rect bị verify bắt; nó chỉ có thể lỗ về chi phí, và gate theo đoạn chặn điều đó.

Điểm dễ sai nhất là z-order: patch vá nền sinh sau sprite X nên mặc định nằm trên X; khi X quay lại, patch che
X và sinh vòng patch chồng patch. Patch vá nền phải được chèn dưới layer di chuyển, nên display model phải
cho phép chèn ở dưới ngay từ M1.

### 5.9 Solver CP, ILP, SAT: thước đo trước khi là công cụ

Chọn representation bằng solver toàn cục trong lúc encode không đáng làm lúc này, vì bốn lý do: chi phí chỉ
biết sau khi encode PNG, nên đưa vào solver nghĩa là trả trước chi phí mà ladder dừng sớm để tránh; fidelity
không cộng được khi candidate chồng nhau qua alpha, z-order và crossfade; quy mô hàng triệu biến buộc chia cửa
sổ, khi đó thành greedy có lookahead; và bài học dự đoán thua đo thật ở 11.4. Hai chỗ solver hợp: oracle
offline cho R3, giải bài toán phủ có trọng số trên candidate đã đo để biết ladder bỏ phí bao nhiêu, chính là
điều kiện đảo của dòng bác bỏ tối ưu toàn cục; và ngân sách workload ở M3, nơi ràng buộc là vách đá theo từng
lát thời gian và candidate đã được đo xong. Các bài toán con khác có thuật toán chuyên dụng tốt hơn: chia đoạn
easing là DP một chiều, z-order với K ≤ 4 duyệt hết 24 thứ tự, gán block vào layer là gán nhãn MRF bằng graph
cut, span crossfade đơn điệu nên greedy tối ưu.

---

## 6. Góc nhìn synthesis: truy hồi storyboard ngược

Có một cách nhìn khác về cùng kiến trúc: coi video đầu vào như thể nó từng được sinh ra bởi một
storyboard, và nhiệm vụ là truy hồi một chương trình storyboard giải thích được nó với chi phí thấp.
Góc nhìn này không thay kiến trúc; nó đặt tên đúng cho kiến trúc đang có và chỉ ra chỗ còn nghèo.

### 6.1 Encoder đã là một synthesizer

Vòng lặp closed-loop chính là một thể hiện của CEGIS (counterexample-guided inductive synthesis):
verify tìm phản ví dụ (dirty region), synthesizer đề xuất fix cục bộ (candidate), rồi verify lại.
Storyboard là một DSL:

```text
Program   := Object*                       (compose theo painter's algorithm: layer rồi thứ tự khai báo)
Object    := Sprite(asset, origin, x, y, Track*) | Animation(assets[], frameDelay, loopType, Track*)
Track     := thuộc 1 trong 9 property độc lập; mỗi track là chuỗi segment không chồng (R5)
Segment   := (t0, t1, v0, v1, easing ∈ 0..34)
Loop      := đường cú pháp, flatten được; Trigger nằm ngoài phạm vi vì phụ thuộc gameplay
```

Ba tính chất quyết định chiến lược. Thứ nhất, semantics đã executable: encoder composite trên một
display model riêng (rect căn pixel, track alpha và tint tuyến tính từng đoạn), còn verifier render IR
bằng renderer độc lập sau khi output đã đi qua vòng text; hai bản thật sự độc lập, và đó là điều kiện
tiên quyết của mọi synthesis-by-verification. Thứ hai, compose theo z là compositional: thêm
object chỉ ảnh hưởng vùng nó phủ, nên synthesis cục bộ hợp lệ. Thứ ba, mỗi track có số chiều thấp, nên
fit một segment là bài curve-fitting cổ điển và rẻ.

Nhưng không có identifiability toàn cục: vô số chương trình render ra cùng một video. Vì vậy "truy hồi
đúng chương trình gốc" là mục tiêu sai; "một chương trình tương đương trong sai số ε với chi phí nhỏ
nhất" mới là mục tiêu đúng, và đó chính là hàm mục tiêu ở 2.2.

### 6.2 Quan sát ánh xạ sang giả thuyết nào

| Chữ ký quan sát được (đo rẻ) | Giả thuyết đáng fit, theo thứ tự | Đã có provider? |
| --- | --- | --- |
| Δ mỗi pixel xấp xỉ 0 | tĩnh, không làm gì | có (không dirty) |
| Δ là phép nhân đều theo kênh trên cả sprite | `C` tint; `F` nếu nền dưới đen | Photometric |
| Δ là blend đơn điệu giữa hai ảnh ổn định | crossfade | Crossfade |
| flow đồng nhất một model similarity | track `M`, `S`, có thể `R` | TransformTrack (M2) |
| flow tách thành K model theo không gian | multi-layer (T5) | để dành tới checkpoint |
| track 1D có gia tốc hoặc giảm tốc mượt | segment easing thay nhiều keyframe linear | chưa có, xem 6.3 |
| có chu kỳ lặp (autocorrelation cao) | `L` loop; Animation LoopForever | một phần, qua LoopExtractor hậu kỳ |
| nội dung trùng nội dung đã thấy | tái dùng sprite hoặc asset kèm transform | để dành tới checkpoint |
| đổi mỗi frame, không có cấu trúc | Animation hoặc chuỗi patch | Patch, Animation (M3) |
| vùng bị che rồi lộ lại không đổi | kéo dài lifetime sprite dưới | hệ quả của closed loop |

Về z-order: nó chỉ quan sát được tại các sự kiện occlusion, tức khi hai layer giao nhau và một cái
thắng. Không có occlusion thì không suy ra được, nhưng cũng vô hại vì mọi thứ tự đều render đúng. Câu
trả lời đúng là truy hồi *một* thứ tự z nhất quán với mọi sự kiện occlusion quan sát được, bằng một
đồ thị ràng buộc nhỏ và topological sort. Có chu trình là bằng chứng phân rã sai, và khi đó phải hạ
giả thuyết xuống.

### 6.3 Fit hàm, không chỉ fit easing

Nguyên tắc rộng hơn: bất cứ biến đổi nào mô tả được bằng một hàm theo thời gian đều nên được xét dưới
dạng fit đường cong thay vì phát từng frame. Easing chỉ là một trường hợp.

Cơ chế cụ thể: RDP hiện tại chỉ biết segment linear. Mở rộng nó thành, với mỗi đoạn giữa hai
breakpoint ứng viên, thử fit *một* segment easing trước khi chấp nhận chèn thêm keyframe. Việc fit là
đánh giá một số hàm cố định trên track vô hướng, không có tham số tự do nào ngoài (v0, v1), nên rất
rẻ và kiểm tra residual có dạng đóng. Segment easing đạt epsilon thì một command thay được N.

Tập easing đáng thử không cần cả 35: chín hàm gồm Linear, In/Out/InOut Quad, In/Out/InOut Cubic,
In/Out Sine đã phủ hầu hết chuyển động được author. Elastic, Bounce và Back chỉ khớp motion graphics
rất đặc thù, thêm sau nếu dataset chỉ ra.

Thắng lớn trên hai loại nội dung: nội dung vốn được author bằng tween (storyboard thật, motion
graphics, UI game) vì khi đó khớp chính xác và N keyframe thu về 1 command; và gia tốc vật lý mượt vì
ease xấp xỉ hàm bậc hai còn RDP linear phải chèn dày ở đoạn cong. Vô ích trên trajectory nhiễu hay
giật, ví dụ footage cầm tay, vì ở đó keyframe linear mới đúng.

Áp cho mọi track fit được: x, y, scale, alpha, colour. Track alpha và colour hiện sinh từ provider
Photometric và Crossfade, nên dùng chung một fitter.

### 6.4 Chọn candidate đầu tiên hay candidate tốt nhất

Hiện tại ladder chọn candidate *đầu tiên* đạt sàn. Câu hỏi mở là có nên đánh giá mọi candidate đạt sàn
rồi chọn cái rẻ nhất. Phân tích: chi phí các bậc cách nhau hàng bậc độ lớn (photometric gần 0, track
vài KB, crossfade 2 asset, patch 1 asset, animation N asset), nên chọn cái đầu tiên xấp xỉ tối ưu ở
ranh giới giữa các bậc. Chỗ có thể sai nằm trong cùng một bậc và giữa hai bậc kề nhau: crossfade dài
so với hai patch, animation so với chuỗi patch, một command easing so với ba keyframe. Chênh lệch nhỏ,
nhưng tần suất thì chưa biết.

Cách biến câu hỏi thành số: chạy một chế độ đo trong đó mỗi region đánh giá mọi provider mà không dừng
sớm, rồi log cặp (ladder chọn gì, rẻ nhất là gì, chênh bao nhiêu bytes). Tổng chênh dưới khoảng 10%
thì giữ ladder và đóng câu hỏi bằng số; từ 10% trở lên thì nâng bộ chọn thành best-among-passing, một
thay đổi cục bộ trong vòng chọn, không đụng contract của provider.

### 6.5 Dataset có ground truth và metric tầng hai

Đây là chỗ duy nhất có ground truth ở mức *chương trình* chứ không chỉ mức pixel, và nó rẻ để xây vì
generator lẫn renderer đều đã có trong repo.

```text
tools/OsbMpeg.DatasetGen/            (project console riêng, không đụng Compiler)
dataset/case_XXX/
├── source.osb + assets/             sinh bởi generator theo template mỗi category, có seed
├── frames/ *.png                    render bằng SoftwareStoryboardRenderer (tier rẻ, lossless)
├── rendered.mp4                     ffmpeg từ frames, x264 CRF 16 (tier thực tế, vì nhiễu codec
│                                    cũng là một phần bài toán thật)
└── metadata.json                    category, seed, resolution, fps, duration, số sprite, số
                                     animation, số command, asset bytes, số layer
```

Các category, mỗi cái một template có tham số ngẫu nhiên: tĩnh, một transform đơn (`M`, `S`, `R` từng
cái), từng họ easing, fade, colour animation, crossfade, pan, zoom, nhiều sprite, layered motion kèm
occlusion, animation flipbook, loop, hỗn hợp, và một nhóm mập mờ gồm những case mà cùng một video sinh
ra được từ hai chương trình khác nhau trở lên.

Vấn đề vòng tròn phải ghi thẳng: tier rẻ render bằng chính renderer của mình, nên bug renderer nhiễm
cả dataset lẫn kết quả truy hồi. Giảm rủi ro bằng hai cách: golden test ở M0 đã ghim renderer so với
semantics của osu!, và một tier vàng gồm 5 đến 10 case render bằng osu! lazer thật (cách làm cụ thể
còn phải xác định lúc build). Sai lệch giữa tier rẻ và tier vàng trên cùng một case chính là bug
renderer, và bản thân nó đã là phát hiện có giá trị.

Không giả định truy hồi luôn khả thi. Nhóm case mập mờ và các case dùng đặc tính chưa hỗ trợ tồn tại
để đo giới hạn, không phải để pass 100%.

Về metric, có ba tầng với thẩm quyền rõ ràng: so sánh văn bản thô là vô nghĩa và chỉ dùng để debug;
so sánh cấu trúc chỉ có ý nghĩa sau khi chuẩn hóa và chỉ dùng cho research; tương đương khi render
(mọi frame, mọi tile đạt sàn) mới là thẩm quyền. Khi hai storyboard khác nhau nhưng render tương
đương thì theo mục tiêu sản phẩm đó là thành công, và metric tầng hai lúc ấy chỉ còn để hỏi cách của
ground truth có rẻ hơn cách đã truy hồi không.

Những thứ không suy ra được từ pixel, đừng đo đúng sai trên chúng: z-order ngoài các sự kiện occlusion,
phân chia giữa scale của asset và scale của command, phân rã khi các layer không bao giờ tách chuyển
động, và keyframe dày so với easing (render trùng nhau). Những thứ đo được có nghĩa: trajectory
composite theo vùng màn hình, nội dung asset sau khi chuẩn hóa, tập sự kiện occlusion, và độ phức tạp.

| Metric | Cách đo | Trả lời câu hỏi |
| --- | --- | --- |
| `complexity_ratio` | cost(truy hồi) chia cost(ground truth) theo từng trục | generator tốt tới đâu |
| `representation_confusion` | ghép tương ứng vùng theo thời gian (IoU screen-space ≥ 0,5, bipartite match) rồi lập ma trận primitive của ground truth so với primitive đã chọn | chọn sai ở đâu |
| `missing_primitive_attribution` | bytes ở vùng mà ground truth dùng primitive X còn ta dùng patch hoặc animation | còn thiếu primitive nào |
| `trajectory_rmse` | RMSE screen-space của trajectory composite trên các vùng đã ghép | chất lượng fit |
| `asset_recall` | tỷ lệ asset của ground truth có match (exact hoặc canonical hash) | tái dùng có hoạt động không |
| `occlusion_consistency` | tỷ lệ sự kiện occlusion tái tạo đúng thứ tự thắng | z-order |

Giữ các metric này riêng biệt. Không xây score tổng hợp cho tới khi có dữ liệu về tương quan, và không
metric tầng hai nào được làm gate.

---

## 7. Failure modes

| Nội dung | Cơ chế fail | Representation đỡ được | Worst case chấp nhận |
| --- | --- | --- | --- |
| Hard cut | mọi model chết cùng lúc | scene detection sẵn có, cộng I-frame | một asset full-frame mỗi cut, đúng giá |
| Chuyển động nhanh | flow không bắt kịp, crossfade sai công cụ | T7 animation có cap, hoặc T2 với patch rate cao | gần bằng full refresh cục bộ, sàn vẫn giữ |
| Camera xoay trong mặt phẳng | không | T4 có `R` | ổn |
| Perspective, xoay ngoài mặt phẳng | similarity thiếu shear | chia strip, hoặc T2 | patch trả, đo attribution rồi mới xây strip |
| Biến dạng chậm (mặt, vải) | không primitive nào biến dạng | T3 crossfade, tốt bất ngờ vì nội suy tuyến tính giữa hai pose xấp xỉ morph khi Δ nhỏ | T2 khi nhanh |
| Particle, khói, lửa | hàng trăm mover nhỏ | T10 khi các hạt giống nhau, cộng T9 vì lửa và glow vốn là additive | T7 hoặc T2 kèm cap ngân sách |
| Phản chiếu, specular | phi rigid và phi photometric | không | patch |
| Đổi ánh sáng toàn cục | không | T8, hoặc T9 khi cần sáng thêm | gần như miễn phí |
| Dissolve | không | T3 native | gần như miễn phí |
| Wipe | không | sprite của scene B kèm track M, mép cứng | T2 nếu wipe theo hình lạ |
| Nhiễu và grain | phá đồng thời motion coherence, dedupe và crossfade | không hướng nào cứu được | JPEG asset cộng preset thấp hơn |
| Occlusion | không | hệ quả của kiến trúc (5.3) | không |

Nhiễu là ranh giới vật lý của toàn bộ cách tiếp cận này. Dữ liệu cũ đã cho thấy: tuner bó tay trên
birdbrain, còn bad_apple ra hơn 10 nghìn sprite cho 5 giây. Nói rõ một lần trong tài liệu sản phẩm,
đừng để mỗi milestone lại "phát hiện" lại.

---

## 8. Bảng quyết định

Mỗi dòng một hướng, kèm điều kiện đảo quyết định. Không hướng nào bị khóa chết bằng ý kiến.

| Hướng | Bằng chứng | Lợi ích kỳ vọng | Độ phức tạp | Hợp với storyboard | Quyết định | Điều gì đảo nó |
| --- | --- | --- | --- | --- | --- | --- |
| Closed loop và sàn tuyệt đối | vòng hở là nguyên nhân gốc của mờ (1.1) | hết mờ do cấu trúc | trung bình | hoàn hảo, ở mức IR | MUST (M1) | không |
| Crossfade (T3) | blend lerp đã verify khớp shader; dissolve rất phổ biến | giảm mạnh refresh rate vùng đổi chậm | thấp | native (`F`) | MUST (M1) | không |
| Photometric (T8) | fade và tint là command native, gần 0 bytes | fade gần như miễn phí | thấp | native (`F`, `C`) | MUST (M1) | không |
| Global motion K=1 (T4) | R1 nội suy liên tục; 79 nghìn sprite là bằng chứng thiếu nó | thắng lớn nhất trên pan và zoom | trung bình | native (`M`, `S`) | MUST (M2), spike flow trước | S2.0 cho thấy flow không dùng được trên corpus thì thu hẹp về nội dung synthetic và game |
| Sub-segment theo motion | scene tĩnh rồi pan rồi tĩnh là ca thường | mỗi segment một representation đúng | thấp | không đổi | MUST (M2) | không |
| Layered motion K > 1, nấc rect nhân quả (T5, M2.5a) | nội dung được author là nhiều vật độc lập (D14); closed loop đã tự vá nền vừa lộ (5.8) | lớn trên `authored`, chưa rõ trên phần còn lại | trung bình | tốt (K sprite xếp z, patch vá chèn dưới) | MUST (M2.5a, sau R2) với keep-gate: trên `authored` bytes ≤ 0,5× và sprite ≤ 0,2× so với M2, verify 100%; trên medium và hard bytes không tăng quá 2% | trượt gate thì quay về DEFER tới M3.5 |
| Mask, alpha, dựng nền nhìn trước, khử lem màu cho layer (M2.5b) | chỉ xóa được phần dải lộ và quầng nền mà nấc a để lại (5.8) | bằng tổng hai bucket `rect-halo` và `reveal-strip` | cao, thêm một pass full-res | tốt (T12) | DEFER tới sau M2.5a | `rect-halo` hoặc `reveal-strip` ≥ 20% bytes trên `authored` |
| Panorama (T6) | toán khả thi (5.5); strip giải được giới hạn texture | cận dưới bytes cho pan dài | trung bình | tốt | DEFER tới M3.5 | bucket `pan-reveal` từ 25 tới 30% |
| Tái dùng asset chuẩn hóa | hash đã có; chuẩn hóa là mở rộng key (I10) | lớn nếu bucket near-dup lớn | thấp tới trung bình | hoàn hảo | DEFER tới M3.5 | bucket `near-duplicate-assets` từ khoảng 20% |
| Perspective strips | similarity thiếu shear, sai hệ thống và đo được | vừa, tùy nội dung | trung bình | chấp nhận được | DEFER tới M3.5 | bucket `perspective-residual` từ 25% |
| Suy luận occlusion riêng | là hệ quả của z-order cộng closed loop (5.3) | 0, vì đã có sẵn | không | không | REJECT (như một feature) | tìm ra ca mà closed loop không tự xử lý được |
| Object detection và tracking ngữ nghĩa làm xương sống (SAM 2/3) | lệch granularity (5.6); lợi ích identity đã có đường rẻ hơn (5.4); SAM 2 tự ghi nhận mất track qua che khuất dài, lẫn object giống nhau, không tự liệt kê object | thấp so với giá | cao, GPU, PyTorch | kém | REJECT (D12) | không |
| Model học tinh chỉnh biên layer (`ILayerBoundaryRefiner`: SAM, MatAnyone) | biên là chỗ duy nhất model học hơn classical | nhỏ trên `authored`, vừa trên footage | trung bình, nằm ngoài assembly (sidecar Python) | tốt, chỉ đổi alpha ở dải biên | DEFER tới sau M3.5 (D13) | bucket "mép layer" ≥ 20% bytes, hoặc verify fail tập trung ở biên layer trên 2D animation |
| Depth estimation model | z-order suy được từ dữ liệu occlusion (6.2) | thấp | cao | gián tiếp | REJECT | layered motion bế tắc riêng vì không suy được thứ tự z |
| Learned optical flow | classical khớp đúng granularity rect (5.6) | chỉ khi classical fail | cao (ONNX, model, GPU) | không đổi | REJECT ở giai đoạn này | S2.0 và cả DIS flow đều fail trên một bucket lớn |
| RDO Lagrangian, graph search hay solver toàn cục làm bộ chọn trong lúc encode | fidelity là ràng buộc cứng chứ không phải λ; bài học predict so với measure (11.4) | thấp | cao | không đổi | REJECT | đo được greedy ladder bỏ phí hơn 10 tới 15% trên corpus |
| DP đặt keyframe | RDP đủ theo mọi số đo hiện có | nhỏ | thấp | không đổi | DEFER | đo thấy RDP bỏ phí keyframe đáng kể |
| Fuzzy, pHash Hamming hay embedding cho near-dup, thay asset không verify pixel | O(N²) và rủi ro sàn; hai asset gần giống thay nhau giữa hai run kề sinh pop; canonical exact phủ phần lớn (5.4) | nhỏ so với canonical | cao | không đổi | REJECT | canonical reuse đã ship mà bucket near-dup vẫn lớn, và chỉ khi có verify SSIM so với sàn |
| Animation fallback (T7) | thrash có thật (bad_apple); R4 giới hạn memory | chặn nổ sprite count | thấp | native, đắt memory | MUST (M3), có cap | không |
| JPEG asset | dữ liệu DEFLATE cũ gợi ý ảnh chụp nén kém bằng PNG, nhưng một lần đo thật cho JPEG *tệ hơn* PNG: +6,5% trên anime, +62,6% trên bad_apple (doc comment `AssetStore.cs`) | chưa rõ | thấp | cần OQ-4 xác nhận | INVESTIGATE (M3), không MUST | đo trên `standard` của footage cho thấy giảm từ 30% mà không thủng gate, và OQ-4 pass |
| WebP hoặc AVIF | chưa có bằng chứng osu! load được | chưa rõ | thấp | chưa xác nhận | REJECT cho tới khi verify | xác nhận cả stable và lazer load được |
| Fit easing rời rạc trong TrajectoryFitter | lý thuyết vững, fit dạng đóng nên rẻ (6.3); nội dung ưu tiên được author bằng tween (D14) | lớn trên nội dung được author, gần 0 trên trajectory nhiễu | thấp | native (35 easing cố định, không tham số tự do) | MUST (R2, ngay sau M2) | keep-gate: ≥ 30% command bytes trên `authored`, 0 verify regression; trượt thì giữ RDP linear |
| Cubic Bézier, spring, curve tham số tự do trong track | `.osb` chỉ có 35 easing cố định; Bézier chỉ xấp xỉ được bằng chuỗi segment | âm (không phát được) | không đổi | không có | REJECT | osu! thêm easing tham số |
| Dataset ground truth và metric tầng hai | generator và renderer đã có; Crello Animation và Animation2Code có ground truth sẵn; mở khóa câu hỏi "chọn sai ở đâu" | là hạ tầng đánh giá | trung bình | là instrument | MUST (R1, ngay sau M1) | không cần gate; sai lệch tier rẻ so với tier vàng tự nó là phát hiện |
| Solver CP-SAT/ILP làm oracle offline (R3) | là cách duy nhất có nghiệm tối ưu để so; chạy trên candidate đã đo nên không dự đoán (5.9) | biết chắc greedy bỏ phí bao nhiêu | trung bình, nằm ngoài assembly | không đổi | MUST (R3, sau M3) | không cần gate, là instrument |
| Solver cho ngân sách workload (S3.1b) | ràng buộc theo lát thời gian liên kết nhau, greedy có thể vừa vi phạm vừa tốn (5.9) | vừa, chỉ trên đoạn vượt | trung bình | không đổi | INVESTIGATE (M3) | S3.1 greedy để lại vi phạm, hoặc R3 thấy gap ≥ 10% ở đoạn vượt |
| Bộ chọn best-among-passing | bậc chi phí cách nhau bậc độ lớn nên nghi ngờ gap nhỏ; chưa đo | nhỏ tới vừa | thấp | không đổi | DEFER tới R3 | gap từ 10% trên dataset và fixture |
| Synthesis nhận biết loop | `.osb` bytes là trục lợi duy nhất; LoopExtractor hậu kỳ đã có | nhỏ trừ khi nội dung có chu kỳ | trung bình | native (`L`) | DEFER (R4, `implementation-plan.md` mục 12) | R1 hoặc attribution thấy chu kỳ phổ biến |
| Prune nhiều độ phân giải trong Evaluate | chỉ loại chứ không nhận nên an toàn với I4 | chỉ về hiệu năng | thấp | không đổi | DEFER | profile cho thấy evaluate là nút cổ chai |
| Score truy hồi tổng hợp | chưa có dữ liệu về tương quan giữa metric và "truy hồi đúng" | chưa rõ | chưa rõ | không đổi | REJECT cho tới khi có dữ liệu | benchmark R1 cho thấy các metric rời tương quan mạnh và cần một số duy nhất |
| Mesh hoặc subdivision tổng quát | nổ sprite count; thiếu shear làm lưới thô sai hệ thống | chưa rõ | cao | kém (fill-rate) | REJECT | chưa thấy đường đảo |
| `[Variables]` để nén `.osb` | R9, 3 giây thành 45 giây | âm | không đổi | không đổi | REJECT vĩnh viễn | không |
| Value-chaining shorthand | lazer chỉ đọc cặp giá trị đầu (R9) | âm trên lazer | không đổi | sai trên chuẩn verify | REJECT vĩnh viễn (D1) | không |
| Nền động hoặc vùng lớn bằng `Animation` | R4: N frame full-res resident cùng lúc, 1080p × 300 frame xấp xỉ 2,4 GB | âm | không đổi | nổ memory | REJECT; `Animation` chỉ cho vùng nhỏ với cap resident bytes (M3) | không |
| Cost function dự đoán với trọng số tay (asset, command, overdraw) | predict thua measure 2,6 đến 5,4 lần (11.4); ba trục workload là vách đá chứ không phải dốc (2.2) | âm | không đổi | không đổi | REJECT; chi phí đo thật, ràng buộc cứng | không |
| Freeze gate so frame t+1 với frame t | chính là vòng hở gây mờ của v1 (1.1) | âm | không đổi | không đổi | REJECT; chỉ so với asset đang hiển thị trên composite | không |

---

## 9. Câu hỏi mở

| ID | Câu hỏi | Vì sao chưa quyết | Spike quyết định | Phạm vi an toàn |
| --- | --- | --- | --- | --- |
| OQ-1 | Chiến lược re-composite nào của `ReconstructionState` nhanh nhất | vấn đề hiệu năng, không phải tính đúng | benchmark 2 chiến lược trên fish 5 giây | chỉ đổi bên trong class, giữ test đối chiếu render đầy đủ |
| OQ-2 | Block matching có đủ tốt trên footage thật không, và trên motion graphics ít texture thì KLT trong support hay ECC trên pixel | chất lượng flow phụ thuộc nội dung; Zhang 2023 ghi flow neural lẫn SIFT và KLT kém trên motion graphics | S2.0 (`implementation-plan.md` mục 12) | code spike ném đi được, chỉ số liệu là deliverable |
| OQ-3 | Con số preset cuối cùng | phụ thuộc nội dung | M0.5 | chờ người dùng xem ảnh, không tự chốt |
| OQ-4 | osu! stable và lazer có load asset `.jpg` trong storyboard không | chưa verify thực tế | đầu S3.3 (`implementation-plan.md` mục 12) | khoảng 30 phút thủ công; fail thì bỏ JPEG, không tìm cách lách |
| OQ-5 | Ngữ nghĩa alive chính xác cho `WorkloadAnalyzer` (alpha lớn hơn 0 lần đầu so với span command) | ảnh hưởng độ chính xác report, không ảnh hưởng output | hỏi cụ thể qua deepwiki khi cần | dùng bản bảo thủ (span command) tới lúc đó |

---

## 10. Prior art đã đối chiếu

Mọi khối trong pipeline đều có người làm trước; giá trị của dự án nằm ở tầng *compile* (chọn
representation trong một DSL cố định, không khả vi, có ràng buộc cứng, dưới sàn fidelity được verify
bằng render, với chi phí đo thật), không ở decomposition. Bảng này tồn tại để không ai đề xuất lại một
khối như thể nó mới, và để biết lấy công cụ ở đâu.

| Khối | Prior art gần nhất | Lấy gì, bỏ gì |
| --- | --- | --- |
| Toàn bộ "nền tĩnh + global motion + object foreground" | MPEG-4 Part 2 sprite coding: static sprite, GME 8 tham số, VOP foreground. Hiệu quả gấp vài lần khi foreground dưới 10 đến 15% diện tích; chưa bao giờ có bản thương mại vì bước segmentation (Glantz 2010, Krutz 2008, Waseda R03) | cùng bài toán, target khác (DSL render được bởi osu!) |
| Object = canonical image + transform mỗi frame + z-index | Zhang, Ma, Wu, Ritchie, Agrawala, *Editing Motion Graphics Video via Motion Vectorization and Transformation*, SIGGRAPH Asia 2023: color clustering LAB + trapped-ball, differentiable compositing fit affine, greedy chọn mapping, canonical image cập nhật khi lộ lớn hơn; object che hoàn toàn mất ID; fail trên non-affine và che khuất nặng | lấy: trapped-ball cho biên trên nội dung phẳng, chiến lược canonical image; bỏ: affine (osu! không có shear, dùng Sim(2)) |
| Sprite = static texture + tham số, init bằng VOS | Suzuki, Kikuchi, Yamaguchi, *Fast Sprite Decomposition from Animated Graphics*, ECCV 2024; dataset Crello Animation (299 template, texture + affine + opacity mỗi frame, CDLA-Permissive-2.0, cần chấp nhận VistaCreate agreement, không tái phân phối) | lấy: dataset làm tier `authored`; texture prior chống artifact |
| Decomposition cartoon có tween và sprite tái dùng | Zhang, Wong, Liu, *Sprite-from-Sprite*, SIGGRAPH Asia 2022 (Pixel MLP) | tham khảo cho `fish-short`; cần số sprite do người chọn |
| Một texture + biến dạng theo frame | Ye et al., *Deformable Sprites*, CVPR 2022; Kasten et al., *Layered Neural Atlases*, SIGGRAPH Asia 2021 (atlas tĩnh, kém sau 100 frame) | chỉ làm detector cho fall-through; osu! không có primitive biến dạng |
| Object kèm effect (bóng, phản xạ, khói) | Lu et al., *Omnimatte*, CVPR 2021 (nền homography tĩnh); OmnimatteRF, ICCV 2023; Lee et al., *Generative Omnimatte*, CVPR 2025 | ngoài phạm vi; bóng đổ là nhân tối nên về lý thuyết map được sang `C` trên sprite phụ, chưa ai thử |
| Segmentation và tracking | SAM 2 (ICLR 2025; Appendix C: fail qua shot change, lẫn object giống nhau, mất track sau che khuất dài, mỗi object chạy riêng; 24 đến 47 fps mỗi object trên A100); SAM 3 (11/2025, prompt bằng khái niệm, detector + tracker); SAM 3.1 Object Multiplex (03/2026) | chỉ qua `ILayerBoundaryRefiner` có gate (D13) |
| Matting và khử lem màu | Yang et al., *MatAnyone*, CVPR 2025 (mask frame đầu, memory propagation); Germer et al., *Fast Multi-Level Foreground Estimation*, ICPR 2020; Forte và Pitié, *FBA Matting*, 2020 | thay mọi công thức trimap tự sinh hay chia cho α; chỉ trong sidecar |
| Nền amodal | Granados 2012 (camera động, graph cut); Zhou et al., *ProPainter*, ICCV 2023 (fail khi vùng che chưa từng lộ); Hudson và Smith, *Track Anything Behind Everything* (generative) | lấy: median trong hệ panorama, lỗ để trong suốt cho closed loop vá; bỏ: generative vì ảo giác |
| Base + α(t)·Light | Sunkavalli et al., *Factored Time-Lapse Video*, SIGGRAPH 2007; Ye et al., *Intrinsic Video*, SIGGRAPH 2014; về toán là NMF hạng 1 | chỉ đúng trong hệ sprite đã bù chuyển động; base theo percentile thấp, không min |
| Layer theo motion, không theo ngữ nghĩa | Wang và Adelson, *Representing Moving Images with Layers*, IEEE TIP 1994; object-centric learning (SAVi++, NeurIPS 2022) còn yếu trên video thật khi không có điều kiện | là khung của mục 5.1 |
| Keyframe và easing | Roberts, PhD 2018 (optimal keyframe selection giống animator); Nowicki, Cowan, Mann, GI 2019; Sawicki 2020 (Bézier tổng quát hóa easing); PELT (Killick 2012); PVLDB 2018 trajectory simplification study. Không có paper riêng cho nhận dạng easing từ quỹ đạo | R2 là contribution nhỏ nhưng thật, nếu có ground truth |
| De-rendering animation sang code | Ji et al., *Animation2Code*, NeurIPS 2026: 1.069 web animation, VLM đạt appearance ~0,8 nhưng temporal similarity ~0,3 kể cả sau fine-tune; Liang et al., *VAnim*, ICML 2026 | bằng chứng bài toán chưa bị mô hình lớn giải; dataset dùng cho tier `authored` (kiểm license trước) |

---

## 11. Lịch sử đo đạc của bản v1

Phần này giữ lại các kết quả đo và các hướng đã thử từ tài liệu research cũ, để không ai phải suy diễn
lại từ đầu hay đề xuất lại thứ đã bị bác bỏ có bằng chứng.

### 11.1 Motion và region compensation (đã bác bỏ)

`GlobalMotionEstimator`, `RegionSegmenter`, `RegionTracker`, `TransformEstimator`, `TrajectoryFitter`,
`OcclusionAnalyzer` từng bị bác bỏ vì lý do cấu trúc chứ không phải chưa tinh chỉnh. `.osb` không có
predictive hay residual coding, mỗi Sprite là một texture đầy đủ và tự chứa. Một motion vector chỉ
dùng để *dự đoán* tile lưới cố định tại (x,y) từ frame trước không mua được gì: bytes của asset vẫn
đến từ việc crop vùng (x,y) cố định của frame *hiện tại*, mà vùng đó đổi mỗi frame dưới một cú pan bất
kể vector nói gì. Muốn asset trùng byte qua các frame khi có chuyển động thật thì phải dịch lưới *lấy
mẫu* theo nội dung, nhưng khi đó lưới *xuất* cũng phải dịch đúng như vậy để dựng lại cho đúng, và điều
đó mở ra khoảng trống ở một mép cùng phần thừa ở mép kia.

Điểm khác biệt của hướng mới ở mục 5: asset là sprite bền di chuyển bằng lệnh `M` và `S`, không
re-crop mỗi frame, nên motion vector trực tiếp thành command. Coverage gap ở mép pan là có chủ đích và
được closed loop xử lý như mọi vùng dirty khác. Tiền đề của lần bác bỏ cũ không còn.

Một nhánh chưa chết nhưng chưa từng được đo: residual coding có dấu bằng một Sprite thứ hai blend
normal lên trên một base, với dấu và độ lớn nướng vào RGBA từng pixel. Đã xác nhận biểu diễn được từ
fragment shader của osu-framework (`sh_Masking.h`: `return v_Colour * texel;`, alpha của texel thật sự
tham gia blend). Cái chặn nó: normal blend là `lerp(prediction, overlay, alpha)` theo từng kênh, nên
tái tạo chính xác một target bất kỳ cần một màu overlay theo pixel, mà điều đó tương đương suy lại
chính ảnh target, tức không dedupe được gì trừ khi màu overlay suy biến về một palette dấu nhỏ (ví dụ
đen và trắng), và khi đó một alpha phải dùng chung cho R, G, B nên chỉ đúng ở nơi cả ba kênh có delta
cùng dấu. Chưa bao giờ đo: trên footage thật, tỷ lệ pixel có `sign(ΔR)=sign(ΔG)=sign(ΔB)` là bao
nhiêu, và cặp (sign map, alpha map) có dedupe tốt hơn tile thô không. Ngoài ra base sprite không thể
đóng run khi còn residual phụ thuộc vào nó, tức gấp đôi số asset cho mỗi tile được phủ.

### 11.2 AssetTrimmer (đã bác bỏ)

Cơ chế nén thì hoạt động: một phép thử DEFLATE (ffmpeg, RGB và alpha bị zero dưới một mask tổng hợp,
khoảng 14 mẫu) cho thấy asset lớn hơn 3KB giảm được 25 tới 74% (trung bình khoảng 45%), còn asset nhỏ
hơn 1,5KB thì *tệ đi*, vì header và chunk của PNG chiếm ưu thế ở cỡ đó. Đây là điểm giao đo được,
không phải phỏng đoán.

Nhưng cả hai biến thể (mask alpha giữ nguyên footprint, và crop thật) đều dính một lỗi đúng đắn: các
sprite tại một vị trí tile có lifetime không chồng nhau theo thiết kế, run B bắt đầu đúng lúc run A
kết thúc. Nếu asset của run B làm trong suốt các pixel "không đổi so với run A" để tiết kiệm bytes thì
trong suốt lifetime của run B chúng render thành canvas trống, vì sprite của run A đã bị gỡ từ trước.
Muốn quay lại hướng này thì cần một mô hình lifetime hoàn toàn khác, kiểu sprite không bao giờ bị gỡ
và được vá dần.

Bài học đó đã được tôn trọng trong thiết kế crossfade ở 9.3: sprite cũ được giữ sống cho tới khi
sprite mới opaque hoàn toàn.

### 11.3 Tầng RDO (đã bác bỏ)

Trước khi xây bất kỳ framework nào, một phép đo được đặt làm điều kiện: quét
`--min-animation-uniqueness` (ngưỡng heuristic chọn Sprite so với Animation sẵn có) xem một lần tinh
chỉnh mặc định đơn giản có bắt được các ca mà framework theo chi phí lẽ ra cần cho không. Đã quét hai
lần, trước và sau một sửa lỗi không liên quan ở `QuadtreeMerger`, và output trùng byte cả hai lần trên
fixture test, trước cũng như sau. Cả hai lần thắng thật về nén trong phiên đó đều truy về việc sửa bug
cơ học ở *phía trên* tầng chọn candidate (một lỗi làm tròn và một lỗi tiêu chí merge), không phải do
bản thân heuristic chọn sai. Không xây; tiền đề (một ca mà ngưỡng uniqueness về cấu trúc không thể
biểu đạt lựa chọn đúng) vẫn chưa được chứng minh trên mọi fixture đã thử.

### 11.4 Phân hoạch tile theo heatmap (spike, không nhận)

Ý tưởng: thay vì lưới tile đều cố định, dựng heatmap tần suất thay đổi từ một lần quét mịn rồi phân
hoạch canvas từ trên xuống, vùng lớn ở nơi nội dung tĩnh và vùng nhỏ ở nơi nội dung thay đổi liên tục.

Lần thử đầu có một lỗi phương pháp thật, bị bắt trước khi tin vào số: quyết định split dùng tần suất
thay đổi *trung bình* của mọi tile mịn trong vùng ứng viên. Trung bình che đúng ca cần tránh: một góc
nhỏ biến động (ví dụ một vật thể di chuyển chiếm một phần vùng lớn) ép hash tổng hợp của *cả* vùng đổi
gần như mỗi frame bất kể phần còn lại tĩnh tới đâu, nhưng trung bình toàn vùng vẫn thấp nên không bao
giờ kích hoạt split. Kết quả: cả canvas 1920×1080 của fish_spinning giữ nguyên một vùng, re-encode PNG
full-frame khoảng 600 lần, 202MB cho cửa sổ 10 giây. Cùng nguyên nhân gốc với bug merge một frame của
`QuadtreeMerger`: một tín hiệu tổng hợp bỏ sót biến động cục bộ. Sửa bằng cách đổi tiêu chí split sang
tần suất thay đổi *lớn nhất* của bất kỳ tile mịn nào trong vùng.

Spike sau khi sửa, đo chính xác theo byte trên ba fixture, dùng `ContentHasher` thật và `AssetStore`
in-memory để có chi phí PNG thật, so với output có tuning theo scene của bản hiện tại:

| Fixture | Window | Adaptive assetBytes | Tuned assetBytes | Tỷ lệ |
| --- | --- | --- | --- | --- |
| fish_spinning | 10s | 201.209.266 | 76.154.338 | tệ hơn 2,6 lần |
| minecraft | 10s | 375.766.670 | 69.576.841 | tệ hơn 5,4 lần |
| short_animation | 16,5s | 3.714.496 | 1.442.462 | tệ hơn 2,6 lần |

Thua cả ba, kể cả fixture nội dung sạch mà ý tưởng lẽ ra được lợi nhất (short_animation: 2294 vùng
phát ra so với 116 sprite của hệ thống thật trên cùng nội dung).

Hai điểm gây nhiễu đã biết, tức đây không phải cuộc so hoàn toàn công bằng: spike hardcode Colors=0
(không lượng tử hóa palette PNG) trong khi combo đã tune của short_animation dùng Colors=16; và ngưỡng
split (tần suất thay đổi lớn nhất trên 0,15) chỉ được thử đúng một lần, không quét. Nhưng có tín hiệu
thật bên dưới: bộ tuner theo scene *tự chọn* TileSize=256, tức vùng lớn và đều, chấp nhận chi phí
re-emit, cho đúng các fixture chuyển động mạnh mà ý tưởng này nhắm tới, ngược hẳn với dự đoán của phân
hoạch mịn theo nội dung.

### 11.5 Những thứ đã sửa và thắng đo được

**Bug merge một frame của `QuadtreeMerger` (commit `cdb5820`).** Tiêu chí merge duy nhất của
`TryMergeBlock` là `run.StartMs == first.StartMs && run.EndMs == first.EndMs`, tức sub-tile merge với
nhau bất cứ khi nào chúng đóng cùng thời điểm, không kiểm tra nội dung tương tự chút nào. Ở vùng
chuyển động nhanh nơi mọi tile kề nhau đổi mỗi frame, cả bốn đóng cùng nhịp thuần túy vì cùng biến
động mạnh, thỏa mãn phép thử thời gian mà không chia sẻ chút nội dung nào. Tìm ra qua một ca cụ thể:
vị trí tile chiếm 399 trên 425 sprite của `fish_spin_test` hóa ra là một block 640×640 do
`QuadtreeMerger` gộp (xác nhận bằng `ffprobe` trên PNG thật, không phải tile gốc 320×320), 100% khác
nhau giữa các frame, và vĩnh viễn không đủ điều kiện để `AnimationDetector` nâng cấp. Sửa: từ chối
merge khi khoảng `[StartMs,EndMs]` chung chỉ dài một frame, vì merge dài một frame theo định nghĩa
không bao giờ biểu diễn được nội dung tĩnh dùng chung thật sự. Thắng đo được trên `fish_spin_test`:
sprite từ 720 xuống 344 (giảm 52%), command từ 834 xuống 445 (giảm 47%), `.osb` từ 79,06KB xuống
44,63KB (giảm 44%), PSNR và SSIM *tăng* (35,12 lên 37,81 dB; 0,9687 lên 0,9847).

Một bug nhỏ hơn sửa cùng lúc: phép kiểm `isSingleFrame` của `AnimationDetector` dùng dung sai 0,5ms so
với thời lượng frame thật (phân số). Ở 30fps (33,333ms mỗi frame), biên run bị lượng tử hóa về ms nên
xen kẽ 33ms và 34ms theo tỷ lệ khoảng 2:1, mà dung sai 0,5ms chỉ chấp nhận được một phía, phân loại
sai khoảng 29% run một frame thật thành "ổn định". Đã nới lên 1,0ms.

**`FrameSource` treo vĩnh viễn với input không decode được.** Reader của named pipe (`StreamPipeSink`)
chặn vô hạn nếu ffmpeg không bao giờ mở phía output, ví dụ nó thoát ngay vì file input không tồn tại
nên không bao giờ tới bước mở pipe. Đã xác minh tiến trình thoát sạch (khoảng 1,3 giây với một lần
chạy ffmpeg thật trên file không tồn tại): treo hoàn toàn nằm ở phía .NET, `NamedPipeServerStream` chờ
một kết nối không bao giờ tới, và việc tiến trình đã thoát không tự gỡ được cái chờ đó.

Ba lần sửa hụt, mỗi lần được đo là sai chứ không phải phỏng đoán: (1) hủy khi task của
`FFMpegArguments.ProcessAsynchronously()` hoàn thành, nhưng task đó không trả về cho tới khi I/O của
sink xong, tức nó đang kẹt đúng chỗ đó; (2) chỉ giới hạn `ReadAsync` đầu tiên trong callback của sink,
nhưng bắt tay kết nối xảy ra *trước* khi delegate của sink được gọi trong trường hợp client không bao
giờ kết nối; (3) `CancellableThrough(token)` để giết tiến trình ffmpeg, vô nghĩa khi tiến trình đã tự
thoát.

Cách thật sự hiệu quả: chỉ giới hạn phần chờ channel của *chính mình*
(`ChannelReader.WaitToReadAsync(token)`, hoàn toàn nằm trong tầm kiểm soát của phương thức đang gọi,
độc lập với nội bộ FFMpegCore) cho riêng item đầu tiên. Khi dữ liệu thật đã chảy một lần thì nhịp
decode trở lại bình thường và không giới hạn. `StartupTimeout` ban đầu là 20 giây, sau nới lên 60 giây
khi tuning theo scene chạy song song khiến vài lần decode ffmpeg có thể tranh CPU cùng lúc; input hỏng
thật vẫn fail trong khoảng 1,3 giây nên nới timeout không che lỗi thật.

**`VideoId` từng là bộ đếm theo document, không phải danh tính ổn định.**
`VideoSourcePlanner.PlanAsync` gán `VideoId = i.ToString("x")`, tức thứ tự gặp trong document của một
lần compile. Vì `VideoId` đặt tên thư mục con của cache asset, hai project `.osbv` tham chiếu cùng một
video nhưng khác số lượng hoặc thứ tự nguồn lại rơi vào *hai* thư mục asset khác nhau cho cùng một
file, không chia sẻ gì. Đã sửa: hash ổn định của `(NormalizedPath, EffectiveFps)` qua `XxHash128`. Xác
minh thật: compile một document `.osbv` hình dạng khác tham chiếu cùng video ở vị trí nguồn thứ hai
vào cùng thư mục asset, thì `scenes.json` và file asset của video dùng chung được đọc thẳng từ cache
(không có dòng log tune lại nào), và tên thư mục `VideoId` trùng byte với document ban đầu.

**`Colors` bị cố định lúc dựng `AssetStore` thay vì theo từng lần gọi.** Phát hiện giữa lúc làm tuning
theo scene, tức là một bug mà tính năng đó *sẽ tạo ra* nếu không bắt kịp: `AssetStore.GetOrAdd` và
`WriteAnimation` hash trên bytes RGB thô, còn `Colors` cố định lúc dựng. Ổn khi một combo phủ cả
video, hỏng ngay khi hai *scene* dùng chung một `AssetStore` mà chọn `Colors` khác nhau cho cùng pixel
thô, vì scene thứ hai sẽ âm thầm dùng lại thứ scene đầu đã ghi (chế độ hexNaming bỏ qua encode lại khi
đường dẫn đích đã tồn tại), sai cả mức lượng tử hóa. Đã sửa bằng cách gấp `Colors` vào content hash
làm seed của XXH3.

### 11.6 Những thứ đã ship và số liệu của chúng

**Cache asset content-addressed bền (XXH3-128, tên theo hex).** Thay SHA-256 cắt ngắn (64 bit) bằng
XXH3-128, tốt hơn hẳn ở cả hai trục chứ không phải đánh đổi. Đường dẫn ở chế độ `hexNaming` suy từ
content hash (`s/{hash:x32}.png`) nên `SavePng` bỏ qua encode lại khi file đích đã tồn tại, biến thư
mục asset thành cache thật giữa các lần chạy. Xác minh trực tiếp: compile cùng fixture hai lần, lần
lạnh ghi 1102 file trong 25,1 giây, lần ấm không chạm file nào trong 19,1 giây (mtime từ `stat` giống
hệt trước và sau), output `.osb` trùng byte. Trên fixture nhiều animation (tỷ lệ chi phí encode PNG so
với decode cao hơn), tăng tốc nhờ cache ấm hơn 2 lần.

Điều được xác lập ở đây và có ảnh hưởng tới mọi thứ phía sau: thay đổi `TileSize`, `HashQuantLevels`
hay `Colors` *không* trung tính với cache chỉ vì một giá trị khác thì sinh nội dung khác một cách hợp
lệ. Hash phủ cả buffer tile, nên đổi `TileSize` đổi luôn độ dài hash của *mọi* tile, không chỉ những
tile có nội dung nhìn thấy thay đổi. Một lần chạy ở `TileSize=160` không có lấy một cache hit nào so
với cache của lần chạy trước ở `TileSize=320`, ngay cả trên cùng footage. Điều này *đúng* (tile cỡ
khác thì đúng là asset khác) nhưng nghĩa là mọi bộ auto-tune phải hội tụ về một giá trị ổn định cho
mỗi nội dung và giữ nguyên qua các lần compile lại.

**Auto-tune toàn cục.** Luôn bật, không có flag CLI. Sàn chất lượng tự hiệu chuẩn: probe một lần ở
mặc định hardcode trên một cửa sổ mẫu ngắn, lấy PSNR đó *trừ* một khoảng nới nhỏ (1dB) làm sàn cho mọi
candidate. Tìm kiếm theo coordinate descent thay vì lưới 4 chiều (13 tới 15 probe thay vì 81 tới 625):
`TileSize` trước (đòn bẩy cấu trúc lớn nhất), rồi `Colors`, rồi `HashQuantLevels`, rồi `TileTolerance`
cuối cùng. Tín hiệu chi phí là asset bytes *cộng* ước lượng bytes text của `.osb`
(`commandCount * ~100`), không phải asset bytes đơn thuần.

Thắng đo được trên cả 5 fixture, một cửa sổ 10 giây mỗi fixture, combo đã tune so với mặc định
hardcode cũ:

| Fixture | Nội dung | Combo đã tune | Δ bytes | Δ sprites |
| --- | --- | --- | --- | --- |
| short_animation | animation sạch | 256/32/8/16 | -25,2% | 90 trên 911 |
| birdbrain | footage thật, nhiễu | 64/32/8/0 (bằng mặc định) | 0,0% | 8549 trên 8549 |
| bad_apple | nhấp nháy đen trắng cực đoan | 64/16/8/0 | -4,4% | 10007 trên 12398 |
| fish_spinning | FHD, nhiễu vừa | 256/32/8/0 | -7,7% | 875 trên 6705 |
| minecraft | game render, sạch | 256/16/8/0 | -28,9% | 842 trên 14535 |

Không fixture nào tệ đi. Footage thật nhiễu rơi đúng vào mặc định hardcode, tức sàn an toàn hoạt động
như thiết kế. Số sprite và command giảm mạnh hơn hẳn số bytes (Minecraft: bytes giảm 28,9% nhưng sprite
giảm 94%), nghĩa là engine storyboard của osu! xử lý ít object hơn nhiều so với những gì con số bytes
gợi ý. Đây cũng là bằng chứng sớm rằng mô hình chi phí ở 2.2 tách bytes khỏi workload là đúng.

Chi phí tuning đã được đo và cắt dần, mỗi bước đo trước khi sang bước sau: 5 phút 31 giây (lần chạy
thật đầu tiên, 14 probe, mẫu 4000ms), xuống 2 phút 8 giây (thu cửa sổ mẫu còn 1500ms, vì thời gian
render mới là chi phí trội chứ không phải số lần decode hay số lần spawn ffmpeg, tìm ra bằng đo từng
probe chứ không phải phỏng đoán), xuống 1 phút 25 giây (song song hóa các candidate trong cùng một
trục), xuống 30 giây (bỏ probe trùng khi mang seed sang trục sau).

**Tuning theo từng scene.** Động cơ đến từ hai thí nghiệm ném đi trước khi quyết xây: một fixture hai
scene có hard cut (giảm 8,9% bytes so với một combo toàn cục) và một fixture ghép 5 fixture (giảm
26,7%). Cảnh báo được ghi ngay lúc đó: 26,7% là trần, vì đó là 5 nguồn khác nhau tối đa bị cắt mỗi 5
giây, ca thuận lợi nhất mà tuning theo scene có thể gặp. Đo trên code đã ship thật (ép cùng một đường
code production đi qua nhánh "1 scene" so với "N scene" để so sánh tương đương): **giảm 7,86%** trên
một fixture 8 scene thật, khớp với cảnh báo "đừng kỳ vọng trần".

Phát hiện scene boundary (`ScenePrePass`, chỉ decode, không PNG, không QuadtreeMerger, không
AnimationDetector) tìm hard cut qua một tín hiệu duy nhất: một frame mà tỷ lệ rất cao các vị trí tile
đóng run đồng thời, dưới *bất kỳ* bộ tham số nào, phát hiện thẳng từ giá trị trả về của
`TileRunTracker.Advance`.

Một lỗi hiệu chuẩn thật, tìm ra qua chính phần verification của plan chứ không phải bỏ qua bằng giả
định: ngưỡng đoán ban đầu `CutThreshold=0.8` bỏ sót một cut thật đo được ở 0,721, trong khi một frame
chuyển động mạnh bình thường (không phải cut) đo được 0,662; đã hiệu chuẩn lại về 0,7, nằm đúng giữa
hai số đo thật. Quy tắc chặn merge ban đầu ("từ chối mọi ứng viên trong vòng 2000ms kể từ cut được
chấp nhận gần nhất") nuốt mất một cut thứ hai thật chỉ cách cut đầu 1,8 giây; đã thay bằng thuật toán
hai pass: gom cụm các frame vượt ngưỡng gần nhau thành một ứng viên trước (dung sai khoảng cách
300ms), rồi chỉ từ chối một ứng viên nếu nó tạo ra một scene dài gần bằng 0 ngay sau scene trước
(1000ms, chỉ để chặn ca suy biến).

**Tuning lười theo scene.** Bản ship ban đầu tune *mọi* scene phát hiện được trên cả file ngay sau
prepass. Một truy vấn `.osbv` cửa sổ 10 giây trên file 255 giây có nhiều cut tự nhiên phải trả khoảng
100 tới 170 giây tuning *mỗi scene*, cho những scene chẳng liên quan gì tới cửa sổ được yêu cầu. Sửa:
chỉ tune một scene khi `VideoCompiler` sắp thật sự encode nó.

**I/O đĩa của probe chiếm khoảng 75% wall time mỗi probe** (đo trên 52 probe thật, không phải ước
lượng): mỗi probe ghi PNG thật ra một thư mục tạm chỉ để biết kích thước bytes rồi đọc lại để so PSNR,
tức thuần overhead. Đã sửa bằng chế độ `AssetStore` in-memory (vẫn encode PNG, nhưng vào `MemoryStream`
thay vì file) và một renderer đọc thẳng asset probe từ đó. Kiểm tra regression trên cùng scene xác
nhận output trùng byte và tuning nhanh hơn khoảng 2,4 lần trên một cặp scene thật (213 giây xuống 88
giây).

**Chia ba hệ thống, detection giới hạn theo window, asset store toàn cục.** Cấu trúc phẳng
`Analysis`/`Encoder`/`Media`/`Osb`/`Render`/`Evaluation`/`Tuning` của `Compiler` được gom lại thành 3
thư mục hệ thống (`Detection`, `Tuning`, `Encode`) cộng `Shared` và `Compilation`. Thuần di chuyển và
đổi namespace, không đổi hành vi, xác minh bằng cách grep toàn bộ `using` và tham chiếu đủ điều kiện
trước khi di chuyển.

Cùng lượt đó có một thay đổi hành vi: `ScenePrePass.ScanAsync` trước kia decode toàn bộ file nguồn để
tìm mọi cut, dù một lần compile `.osbv` chỉ cần các scene giao với cửa sổ của chính nó. Đã giới hạn
`ScanAsync` vào `[windowStartMs, windowEndMs)`.

`VideoSourcePlan` từng mang `VideoId` đặt tên thư mục con của asset store, mỗi plan một instance
`AssetStore`. Điều đó chặn hai cơ hội dedupe thật: cùng một file encode lại ở fps khác, và hai file
*khác nhau* có tile sinh ra nội dung trùng byte. Đã sửa triệt để hơn một `VideoId` tốt hơn: bỏ hẳn
trường đó. Một instance `AssetStore` phẳng, content-addressed (`s/{hash}.png`, `a/{hash}/f{n}.png`)
dùng chung cho *cả* compile. Phạm vi đã chốt: chỉ dedupe theo hash chính xác, không so khớp gần đúng
hay theo cảm nhận, không nội suy frame giữa các fps để tạo thêm hash hit. `AssetStore` phải được dựng
một lần mỗi lần gọi `CompileAsync` và truyền xuống, không phải mỗi plan một cái, vì `FileCount` và
`TotalBytes` sẽ đếm trùng một cache hit xuyên plan mà instance kia không biết.

**Phương pháp mẫu train và eval cho tuning.** Thiết kế ban đầu probe một cửa sổ mẫu khoảng 1,5 giây
cho mỗi candidate, nên một candidate có thể trông như thắng thuần túy vì nội dung mà đúng đoạn clip đó
chứa. Thiết kế mới, chốt qua đối thoại: mục đích của eval là *từ chối* chứ không chỉ đo, nên một
candidate phải vượt sàn PSNR trên *cả* probe train lẫn một probe eval mà nó chưa từng thấy khi chọn;
mẫu trải đều theo thời gian; kích thước ban đầu là 3 đoạn train cộng 1 đoạn eval, mỗi đoạn 500ms.

Một bug tìm ra qua regression thật (`badapple8`), unit test không bắt được: bản cài đặt đầu tiên trải
3 cửa sổ train và 1 cửa sổ eval ra 4 phần *tách rời, cách xa nhau* của scene. Trên fixture bad_apple
thật, cách này đo baseline PSNR ở 28,02dB thay vì baseline một mẫu đã biết khoảng 24,90dB, vì lát eval
xa rơi vào nội dung dễ nén hơn, đẩy sàn tự hiệu chuẩn lên khoảng 3dB và khiến mọi trục rơi về baseline.
Hậu quả: tổng bytes tăng 18,9% (34,3MB so với 28,9MB đã biết là tốt) và thời gian tuning gấp khoảng 2
lần. Nguyên nhân gốc: một phép chia train/eval mà các mẫu rải khắp scene đang đo "combo này tổng quát
hóa tốt tới đâu trên toàn scene", một câu hỏi khác và chặt hơn "một mẫu 3 giây cục bộ có đại diện
không".

Đã sửa bằng cách thay bố cục bốn phần tư rải rác bằng một *khối cục bộ* `RequiredSampleMs=3000ms` (căn
giữa nếu là segment đầu tiên được tune, neo ở đầu segment nếu không), chia thành 4 khối liền kề 750ms,
lấy khối 0, 1, 3 làm train và khối 2 làm eval. Xác minh trên cùng fixture thật (`badapple9`): output
trùng byte với baseline tốt trước khi regression (28.793.996 bytes tổng), cùng combo được chọn cho mỗi
scene.

Một tinh chỉnh cuối, từ nhận xét của người dùng: một scene không dài hơn `RequiredSampleMs` không phải
là mẫu của cái gì lớn hơn, nó *chính là* toàn bộ sản phẩm cho scene đó, nên tune nó với một lát eval
giữ lại của chính nó chẳng đạt được gì (không còn "vật liệu chưa thấy" nào để tổng quát hóa tới, và
overfit vào 100% dữ liệu output của chính mình là mục tiêu chứ không phải rủi ro). `BuildSampleWindows`
giờ rẽ nhánh: segment ngắn hơn hoặc bằng `RequiredSampleMs` trả về toàn bộ span của nó làm cả train
lẫn eval (probe một lần, dùng lại kết quả), segment dài hơn giữ nguyên cách chia 4 khối. Xác minh trên
`badapple10`: với scene ngắn 2083ms trong fixture đó, mọi dòng probe in `trainPSNR` bằng đúng
`evalPSNR`, xác nhận là dùng lại chứ không phải probe lần hai; scene dài hơn vẫn chia bình thường; các
số đếm object (`sprites=6897 animations=1467 commands=8364 assets=27706`) khớp baseline tốt đã biết ở
mọi chiều.

**Decode dùng chung cho các cửa sổ mẫu, và bỏ qua vòng PNG trong probe.** `ParameterTuner.TuneAsync`
decode trước các cửa sổ mẫu cố định đúng một lần rồi replay qua `TileEncodeLoop.Options.PreDecodedFrames`.
Số lần spawn ffmpeg mỗi lượt tuning: 40 xuống 4. Thời gian chờ frame: 54,9 giây xuống 13ms. Bản
in-memory của `AssetStore` giữ pixel sau lượng tử hóa (`GetMemoryPixels`) và
`SoftwareStoryboardRenderer.LoadAsset` đọc thẳng từ đó, nên vòng decode PNG biến mất khỏi đường probe.
Đường compile ghi đĩa không đổi.

Một cảnh báo về tính đúng đắn mà bản audit phát hiện, vì tuyên bố ban đầu quá mạnh: output *không*
trùng byte so với đường cũ, và bản thân đường cũ cũng chưa bao giờ có tính chất đó, vì decode theo
từng lần spawn ffmpeg dao động ±1 frame ở biên cửa sổ. Trong một lần chạy đường *cũ*, cùng một bộ
(64,32,8,0) được probe ở hai trục khác nhau cho ra chi phí lệch **9%** (3.226.823 so với 3.530.775) và
eval PSNR lệch 0,01dB. Đường mới so đường cũ giữa các lần chạy: train 29,53 so với 29,56dB, eval 23,76
so với 23,67dB, chi phí lệch trong 6,5%, cùng bậc với dao động vốn có đó. Quyết định giữ nguyên
(64/32/8/0 ở cả hai đường). Đường dùng chung *loại bỏ* dao động trong một lần chạy, vì một lần decode
mỗi cửa sổ nghĩa là mọi candidate so với cùng một bộ frame tham chiếu cố định.

Một bug thật xuất hiện khi bản dùng chung được viết lại bằng fan-out theo channel
(`WindowFrameSource`, `FanOutFrameBuffer`): producer ghi tới *mọi* consumer vô điều kiện trên mỗi
frame, nên bất kỳ consumer nào mà probe không đang đọc sẽ đầy buffer 8 frame và **chặn producer**,
khiến nó không thể nuôi các consumer đang được đọc, tức **deadlock ở concurrency từ 4 trở lên**. Ở
concurrency 1 thì không deadlock nhưng **giao thiếu frame** (chỉ khoảng 8 frame đã buffer tới được
reader duy nhất), nên probe render vài frame rồi báo "thành công" nhanh giả tạo khoảng 56 giây kèm một
PSNR trông hợp lý, tức công việc dở dang giả dạng thành thắng lợi. Đã sửa bằng cách quay lại bản replay
in-memory đã được chứng minh (commit `8de86aa`): decode mỗi cửa sổ mẫu *một lần* vào `List<byte[]>` rồi
replay qua `PreDecodedFrames`, không có channel nào để deadlock. Xác minh chạy được ở cả concurrency 1
và 4, báo 4 cửa sổ đã decode, RAM đỉnh 1,96 GB, combo 64/32/8/0.

A/B trên máy có GPU decode (CUDA), bad_apple scene 5 giây ở giữa, concurrency 1:

| | đường cũ (`--no-shared`) | đường mới (dùng chung) | thay đổi |
| --- | ---: | ---: | --- |
| Số lần spawn ffmpeg | 40 | 4 | ít hơn 10 lần |
| Chờ frame (ffmpeg) | 43.239ms (9,7%) | 12ms (0,0%) | loại bỏ |
| Công CPU khi encode | 444.300ms | 351.050ms | giảm 21% |
| Wall time | 358.129ms | 360.881ms | ngang nhau (+0,8%) |
| RAM đỉnh | 1,71 GB | 1,96 GB | tăng 0,25 GB (giữ frame đã decode) |
| Combo | 64/32/8/0 | 64/32/8/0 | ổn định |

Trên một máy khác chỉ decode bằng CPU, cùng code và cùng thiết kế, wall time của đường mới *tệ hơn*
(526 so với 336 giây). Khác biệt nằm ở đường decode của máy: với decode CPU, đường cũ chạy 40 lần
decode ffmpeg *bất đồng bộ* phía sau phần việc CPU nên mua được khoảng 2,5 lần chồng lấn pipeline, còn
đường mới thuần CPU sau lần decode một lần nên chỉ có khoảng 1,0 lần chồng lấn. Với GPU decode, lợi
thế chồng lấn đó biến mất và cả hai đều bị chặn bởi CPU của software renderer. Kết luận: khoảng cách
wall time là do máy chứ không phải regression của code; số đo độc lập với tải là tổng công CPU, và nó
giảm một nửa.

**Số liệu của một lần compile đầy đủ.** Minecraft full clip 60fps, 1920x804, decode bằng RTX 3060: 542
phút (khoảng 9 tiếng), `.osb` 9,31 MB cộng asset 735,01 MB, tổng 744,32 MB, 79.612 sprite và 79.223
asset, 36 scene, 35 cut, timeline 60fps nhân bản từ nguồn 23,976fps. Đây là con số nền để so mọi cải
tiến sau này.
