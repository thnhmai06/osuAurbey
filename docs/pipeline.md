# OsbMpeg: pipeline trích tách video thành storyboard osu!

## 0. Về tài liệu này

Đây là tài liệu thiết kế duy nhất của OsbMpeg. Nó mô tả pipeline từ đầu tới cuối: bài toán, những gì osu!
storyboard biểu diễn được, chiến lược, các quyết định đã chốt, kiến trúc, từng bước xử lý trong một lần
compile, cách đo, các hướng đã bác bỏ kèm điều kiện đảo, và prior art. Tài liệu không chứa bước triển khai,
commit hay danh sách test: **plan triển khai được sinh ra sau, từ tài liệu này**.

Code hiện có trong repo (bản v1, codec lưới tile) chỉ là **kho thành phần để tái sử dụng** (mục 18). Nó
không phải mốc so sánh: không acceptance nào trong tài liệu này được đo bằng "tốt hơn v1". Mọi điều kiện
nhận đều đo trên chính pipeline mới, so với sàn chất lượng tuyệt đối hoặc so với nấc năng lực liền trước
(mục 12). Lịch sử đo của v1 chỉ được giữ ở dạng bài học đã trả giá (mục 17).

Gặp điểm tài liệu không nói rõ và không suy ra được từ invariants (mục 6.2): hỏi người dùng, đừng tự phát
minh kiến trúc.

Ký hiệu dùng xuyên suốt:

| Ký hiệu | Nghĩa |
| --- | --- |
| `D1`…`D19` | quyết định đã chốt với người dùng (mục 6.1) |
| `I1`…`I18` | invariant, vi phạm là bug kiến trúc (mục 6.2) |
| `R1`…`R10` | sự thật về osu! mà thiết kế phải tôn trọng (mục 2) |
| `T1`…`T13` | representation storyboard biểu diễn được (mục 3) |
| `M0`…`M3.5`, `DS`, `EZ`, `OR` | nấc năng lực của pipeline (mục 12) |
| `OQ-n` | câu hỏi mở có phạm vi spike an toàn (mục 15) |
| `MUST` | đủ bằng chứng, xây theo thiết kế |
| `INVESTIGATE` | đáng nghiên cứu, chưa khóa, có phạm vi spike |
| `DEFER` | chỉ quyết sau khi có số liệu được nêu tên |
| `REJECT` | có bằng chứng cho thấy không đáng làm, kèm điều kiện đảo |

---

## 1. Bài toán và sản phẩm

### 1.1 Sản phẩm

OsbMpeg là **một thư viện .NET 10**. Lõi là assembly `OsbMpeg`: nó trích tách **một video** thành tập object
storyboard osu! biểu diễn bằng object model của OsuParsers, ghi kết quả vào **các channel do caller đưa,
tách theo loại** (object, asset, tiến độ), mỗi channel đúng trình tự thời gian và object đúng thứ tự trong file
storyboard, và ghi asset PNG lossless vào nơi lưu do caller chỉ định. Kết quả cuối là một `Task`. Lõi **không ghi `.osb`** và không biết
cú pháp văn bản của nó; việc đó thuộc project ngoài `OsbMpeg.Osb`.

Kết quả phải đạt năm điều:

1. **Chất lượng:** mọi frame, tại fps yêu cầu, đạt sàn tuyệt đối của preset, được kiểm chứng độc lập với
   encoder (mục 9.11).
2. **Asset lossless:** mọi asset giải mã lại trùng từng byte với pixel nguồn đã cắt. Không lượng tử hóa,
   không palette xấp xỉ, không dither, không encoding lossy dưới bất kỳ hình thức nào (D19). Sàn chất lượng
   chỉ áp cho *cách biểu diễn* (crossfade, tint, transform), không bao giờ cho asset.
3. **Cache asset theo nội dung:** tên asset chỉ phụ thuộc pixel, nằm trong một thư mục dành riêng do caller
   chỉ định, dùng lại được xuyên lần gọi, xuyên video và xuyên tiến trình.
4. **An toàn đồng thời:** nhiều compile song song trong một tiến trình, hoặc nhiều tiến trình dùng chung một
   thư mục asset, không bao giờ ghi đè nhau và không đọc phải file dở.
5. **Không phụ thuộc nơi lưu hay format ghi:** object ghi qua channel, asset đi qua `IAssetStorage`. Trong
   memory chỉ giữ metadata; pixel được nạp lười và có giới hạn.

Không có CLI, không có format `.osbv`, không có auto-tuner, không có scene pre-pass riêng.

### 1.2 Ba mục tiêu, đo bằng metric cụ thể

| Mục tiêu | Metric | Cách đo |
| --- | --- | --- |
| Giữ chất lượng | 100% frame đạt PSNR ≥ P và worst-tile SSIM ≥ S theo preset | verify pass: render toàn bộ output ở độ phân giải nguồn, so từng frame với source |
| Gọn nhẹ | asset bytes, `.osb` bytes, asset count | `WorkloadAnalyzer` |
| Không quá tải renderer | peak SB load, peak alive sprites, animation resident bytes, `.osb` line count | `WorkloadAnalyzer` quét timeline |

Khi các mục tiêu xung đột thì fidelity thắng. Ngân sách workload bị vượt: đổi representation (crossfade
dài hơn, merge patch, đổi encoding, keyframe thưa hơn), không bao giờ âm thầm hạ sàn. Vẫn vượt thì in
violation report kèm gợi ý hạ preset và để người dùng quyết.

### 1.3 Hàm mục tiêu và ràng buộc

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

### 1.4 Quality preset

Gate gồm PSNR tối thiểu mỗi frame và worst-tile SSIM (tile 64×64 luma). Global SSIM chỉ in trong
report, không gate, vì nó là aggregate kiểu trung bình, đúng loại mù cục bộ đã gây ra vấn đề ở 17.1.

| Preset | PSNR tối thiểu mỗi frame | Worst-tile SSIM |
| --- | --- | --- |
| `high` (mặc định) | 38 dB | 0,92 |
| `medium` | 34 dB | 0,88 |
| `low` | 30 dB | 0,80 |

Các số này là placeholder và được chốt bởi hiệu chuẩn M0.5 (mục 11.1), không phải đoán tay. Dữ
liệu lịch sử cảnh báo trước: bad_apple có baseline khoảng 24,9dB *trung bình*, nên PSNR *tối thiểu*
34dB trên nội dung nhiễu có thể đòi patch rate gần bằng full refresh. Spike sẽ đo đường cong sàn so
với patch rate trên từng fixture rồi đặt số, kể cả khả năng preset dùng tileSSIM làm gate chính và
PSNR thấp hơn nhiều so với trực giác.

Preset được chọn qua `CompileOptions.Quality`; không có CLI.

### 1.5 Ưu tiên nội dung và dự đoán theo loại nội dung

Ưu tiên là nội dung được author: motion graphics, UI, 2D animation (D14). Game render
và footage thật không được tệ đi (verify pass, bytes không tăng) nhưng không phải mục tiêu tối ưu.
Dự đoán theo loại nội dung, dùng để đặt fixture và keep-gate:

| Nội dung | Fixture | Dự đoán sau M2.5 | Lợi ích đến từ |
| --- | --- | --- | --- |
| Motion graphics, UI game, logo, text | tier `authored` (generator, Crello Animation, Animation2Code) | giảm sprite và asset 1 đến 2 bậc độ lớn; easing khớp chính xác; nhiều vùng 0 byte | layer Sim(2), easing rời rạc, canonical asset |
| 2D animation sạch | `medium/fish-short` | giảm mạnh ở phần rigid, phần biến dạng (mặt, tay) rơi về crossfade hoặc patch | layer, canonical reuse, crossfade |
| Game render 3D có camera | `hard/minecraft` | phần lớn lợi ích từ K=1 và panorama; foreground có parallax rơi về patch | global motion |
| Footage thật, nhiễu | `hard/birdbrain`, `medium/fish` | gần 0; nhiễu là ranh giới vật lý (mục 13) | không có |
| Silhouette nhấp nháy | `medium/badapple` | như Animation có cap | không có |

---

## 2. osu! storyboard thực sự biểu diễn được gì

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

## 3. Không gian representation

### 3.1 Danh mục những gì storyboard biểu diễn được

| # | Representation | Cấu tạo | Bytes | Workload | Thắng trên |
| --- | --- | --- | --- | --- | --- |
| T1 | StaticPatch | 1 sprite, asset crop, span [t0,t1) | 1 asset | 1 alive, fill = diện tích | nội dung tĩnh |
| T2 | PatchSequence | chuỗi T1 rời nhau cùng vùng | N assets | 1 alive | fallback vạn năng (chính là cách v1 làm) |
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

### 3.2 Giới hạn cứng của không gian này

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

### 3.3 Các tổ hợp đáng chú ý

T3 lồng T4 (crossfade giữa hai TransformTrack) xấp xỉ được "ngoại hình đổi trong khi đang di chuyển",
tức morph thô. Ca này quan trọng vì biến dạng chậm cộng chuyển động là chuyện thường gặp, ví dụ người
đi bộ ở xa camera.

T6 làm layer đáy cho T5 phủ được ca pan kèm foreground độc lập. T8 áp lên mọi thứ nên fade cả scene
không tốn asset nào. T2 là tầng đáy phủ mọi sai số của các tầng trên, và chính điều đó biến mọi tầng
trên thành optimization thuần: chúng sai thì tốn thêm chứ không tạo artifact.

---

## 4. Chiến lược phân rã và tối ưu

### 4.1 Khung thống nhất: layered similarity motion

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

### 4.2 So sánh các chiến lược phân rã

| Chiến lược | Thắng trên | Chi phí compute | Rủi ro | Kết luận |
| --- | --- | --- | --- | --- |
| Global motion (K=1) | pan và zoom camera, chiếm đa số footage game và cinematic | thấp | có gate hai tầng | MUST (M2) |
| Motion layers (K≤4) | parallax, tách foreground; nội dung được author vốn là nhiều object độc lập | trung bình | layer phân mảnh do noise; mép layer cần alpha matte T12 | MUST (M2.5) với keep-gate trên tier `authored` |
| Object tracking ngữ nghĩa (SAM 2/3 và tương đương) | không rõ | cao, GPU | output là mask hoặc box, storyboard chỉ tiêu được rect kèm similarity; tracker mất ID qua che khuất dài | REJECT làm xương sống (5.6, D12); chỉ còn là bộ tinh chỉnh biên tùy chọn trong sidecar, có gate (D13) |
| Depth estimation | thứ tự z cho layer | cao | z-order suy được từ chính occlusion của motion layers | REJECT cho core |
| Panorama, mosaic | pan dài, scroll | trung bình | mover nhiễm vào stitch (median cộng mask xử lý được); giới hạn texture; parallax trong nền phá stitch | DEFER tới checkpoint |
| Suy luận occlusion | không rõ | gần 0 | không | byproduct, không phải feature (4.3) |
| Photometric | fade, flash | gần 0 | ít | MUST (M1) |
| Residual patches | mọi thứ còn lại | không rõ | không | tầng đáy vĩnh viễn |

### 4.3 Occlusion là hệ quả, không phải feature

Với layers, z-order và closed loop, mọi yêu cầu về occlusion tự thỏa. "A che B nên B không cần update"
đúng theo nghĩa đen: closed loop đo error trên composite, vùng B bị che không đóng góp error nên
không patch nào được chi. "A biến mất, B vẫn tồn tại": sprite B có lifetime dài, và nội dung vùng vừa
lộ ra được verify ngay tại frame nó lộ, vì `ErrorMap` chạy sau `AdvanceTo`. Nội dung đúng thì tốn 0,
chưa từng thấy thì patch, đó là điều đúng phải làm.

Thứ duy nhất cần chủ động là kéo dài lifetime của layer bị che thay vì đóng nó, tức giữ sprite sống
qua occlusion nếu dự đoán sẽ lộ lại và nội dung còn đúng. Đó là một heuristic nhỏ trong bộ chọn
representation, không phải một hệ thống.

### 4.4 Định danh entity chuyển thành tái dùng asset chuẩn hóa

Lợi ích thật của "cùng một entity" là tái dùng asset, và có đường rẻ hơn tracking nhiều. `AssetStore`
content-hash đã là re-identification miễn phí cho nội dung trùng byte, xuyên scene và xuyên file. Mở
rộng tự nhiên là chuẩn hóa trước khi hash: đưa patch về pose chuẩn (scale về bậc lượng tử hóa, khử
rotation nếu dùng) rồi hash, còn instance thì đặt lại bằng `M`, `S`, `R` của sprite. Cùng một ngôi sao
ở 5 vị trí và 3 cỡ trở thành 1 asset kèm 5 instance, mà vẫn là exact-hash nên không cần fuzzy search.

Near-duplicate thật sự thì tra ứng viên bằng coarse hash rồi verify bằng SSIM so với sàn. Cách này
bounded và an toàn với sàn cứng, nhưng để dành tới checkpoint vì chỉ đáng khi attribution chỉ ra
bucket này lớn.

### 4.5 Panorama: số học khả thi

Pan 10 giây với 100 osu!px mỗi giây trên nền 1080p cho panorama khoảng 2920×1080. Vượt 1024 nên mất
atlas, nhưng vẫn dưới 4096. Cắt thành strip rộng tối đa 1024, tất cả strip chung một trajectory để
breakpoint đồng bộ. Strip ngoài màn bị GPU clip nên fill gần 0, chỉ tốn một ít alive count. Pan dài
tùy ý chỉ là thêm strip, không chạm `MaxTextureSize`.

Stitch bằng cách warp các frame về hệ panorama qua chính global transform đã có, tích lũy median theo
từng pixel để loại mover, kèm confidence mask. Pixel chưa từng thấy được lấp bằng khuếch tán, confidence 0, không bao giờ được tin; closed loop sẽ patch khi
nó lộ. Không để trong suốt, vì ở mép alpha mềm của layer phía trên lỗ sẽ lộ ra và vi phạm yêu cầu phủ kín. Asset bytes xấp xỉ hợp nội dung, tức cận dưới lý thuyết của mọi cách encode pan.
Điều kiện nhận: scene đã accept K=1 và residual thấp, nghĩa là nền thực sự rigid.

### 4.6 Vì sao không dùng semantic detection và learned model

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

### 4.7 Bốn tầng tối ưu

Không gian tìm kiếm phân rã tự nhiên thành bốn tầng gần độc lập:

```text
(1) Phân rã mỗi scene:        chọn tập layer (K model, có panorama không, ranh giới thời gian)
(2) Representation mỗi vùng:  chọn T1..T13 cho từng dirty region
(3) Keyframe mỗi track:       nén trajectory và span crossfade
(4) Nén mỗi asset:           PNG lossless, chọn cách nén nhỏ nhất
```

| Tầng | Thuật toán | Vì sao không cần nặng hơn |
| --- | --- | --- |
| (1) | matching pursuit: thêm layer hoặc panorama khi lợi ích biên đo được lớn hơn chi phí biên | số ứng viên nhỏ (K ≤ 4, panorama có hoặc không); lợi ích đo rẻ trên bản downscale; sai thì closed loop trả bằng patch chứ không bằng artifact |
| (2) | ladder: thử từ rẻ tới đắt, nhận cái đầu tiên đạt sàn | fidelity là ràng buộc cứng nên bài toán là min cost có ràng buộc, không phải đánh đổi theo λ; chi phí các bậc cách nhau hàng bậc độ lớn nên greedy xấp xỉ tối ưu |
| (3) | RDP joint trong screen space cho track; mở rộng greedy cho span crossfade | error đơn điệu theo span nên greedy tối ưu |
| (4) | thử các cách ghi PNG lossless, kiểm giải mã trùng từng byte, chọn bytes nhỏ nhất | độc lập hoàn toàn theo từng asset nên exhaustive là đúng nghĩa đen; asset không bao giờ bị biến đổi pixel (D19) |

Bác bỏ (đến khi đo được greedy bỏ phí hơn 10 đến 15% trên corpus): global graph optimization, beam
search, DP toàn cục, RDO quét λ. Lý do chung: interaction giữa các quyết định đã bị chặn bởi kiến
trúc phân tầng, vì tầng patch đáy hấp thụ mọi hệ quả xấu thành *chi phí đo được*, nghĩa là greedy
nhìn thấy giá thật của mình ngay. Còn mọi phương pháp toàn cục đều cần cost model *dự đoán* thay vì
*đo*, và đó là nguồn sai mới. Bài học có sẵn: spike heatmap partitioning từng cho ước lượng đẹp nhưng
đo thật thua 2,6 đến 5,4 lần (mục 17.5).

Closed loop trong bức tranh này là cơ chế đúng đắn và là máy đo chi phí thật cho mọi tầng trên. Trí
tuệ nằm ở bốn tầng; closed loop bảo đảm mọi trí tuệ đó không thể sai thành artifact, chỉ có thể sai
thành tốn.

### 4.8 Vật thể di chuyển làm lộ nền: vì sao bắt đầu từ rect nhân quả

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

### 4.9 Solver CP, ILP, SAT: thước đo trước khi là công cụ

Chọn representation bằng solver toàn cục trong lúc encode không đáng làm lúc này, vì bốn lý do: chi phí chỉ
biết sau khi encode PNG, nên đưa vào solver nghĩa là trả trước chi phí mà ladder dừng sớm để tránh; fidelity
không cộng được khi candidate chồng nhau qua alpha, z-order và crossfade; quy mô hàng triệu biến buộc chia cửa
sổ, khi đó thành greedy có lookahead; và bài học dự đoán thua đo thật ở 17.5. Hai chỗ solver hợp: oracle
offline cho `OR`, giải bài toán phủ có trọng số trên candidate đã đo để biết ladder bỏ phí bao nhiêu, chính là
điều kiện đảo của dòng bác bỏ tối ưu toàn cục; và ngân sách workload ở M3, nơi ràng buộc là vách đá theo từng
lát thời gian và candidate đã được đo xong. Các bài toán con khác có thuật toán chuyên dụng tốt hơn: chia đoạn
easing là DP một chiều, z-order với K ≤ 4 duyệt hết 24 thứ tự, gán block vào layer là gán nhãn MRF bằng graph
cut, span crossfade đơn điệu nên greedy tối ưu.

---

## 5. Góc nhìn synthesis: truy hồi storyboard ngược

Có một cách nhìn khác về cùng kiến trúc: coi video đầu vào như thể nó từng được sinh ra bởi một
storyboard, và nhiệm vụ là truy hồi một chương trình storyboard giải thích được nó với chi phí thấp.
Góc nhìn này không thay kiến trúc; nó đặt tên đúng cho kiến trúc đang có và chỉ ra chỗ còn nghèo.

### 5.1 Encoder đã là một synthesizer

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
bằng renderer độc lập trên chính object đã ghi ra, sau khi đổi qua lại với OsuParsers; hai bản thật sự độc lập, và đó là điều kiện
tiên quyết của mọi synthesis-by-verification. Thứ hai, compose theo z là compositional: thêm
object chỉ ảnh hưởng vùng nó phủ, nên synthesis cục bộ hợp lệ. Thứ ba, mỗi track có số chiều thấp, nên
fit một segment là bài curve-fitting cổ điển và rẻ.

Nhưng không có identifiability toàn cục: vô số chương trình render ra cùng một video. Vì vậy "truy hồi
đúng chương trình gốc" là mục tiêu sai; "một chương trình tương đương trong sai số ε với chi phí nhỏ
nhất" mới là mục tiêu đúng, và đó chính là hàm mục tiêu ở 1.3.

### 5.2 Quan sát ánh xạ sang giả thuyết nào

| Chữ ký quan sát được (đo rẻ) | Giả thuyết đáng fit, theo thứ tự | Đã có provider? |
| --- | --- | --- |
| Δ mỗi pixel xấp xỉ 0 | tĩnh, không làm gì | có (không dirty) |
| Δ là phép nhân đều theo kênh trên cả sprite | `C` tint; `F` nếu nền dưới đen | Photometric |
| Δ là blend đơn điệu giữa hai ảnh ổn định | crossfade | Crossfade |
| flow đồng nhất một model similarity | track `M`, `S`, có thể `R` | TransformTrack (M2) |
| flow tách thành K model theo không gian | multi-layer (T5) | để dành tới checkpoint |
| track 1D có gia tốc hoặc giảm tốc mượt | segment easing thay nhiều keyframe linear | chưa có, xem 5.3 |
| có chu kỳ lặp (autocorrelation cao) | `L` loop; Animation LoopForever | một phần, qua LoopExtractor hậu kỳ |
| nội dung trùng nội dung đã thấy | tái dùng sprite hoặc asset kèm transform | để dành tới checkpoint |
| đổi mỗi frame, không có cấu trúc | Animation hoặc chuỗi patch | Patch, Animation (M3) |
| vùng bị che rồi lộ lại không đổi | kéo dài lifetime sprite dưới | hệ quả của closed loop |

Về z-order: nó chỉ quan sát được tại các sự kiện occlusion, tức khi hai layer giao nhau và một cái
thắng. Không có occlusion thì không suy ra được, nhưng cũng vô hại vì mọi thứ tự đều render đúng. Câu
trả lời đúng là truy hồi *một* thứ tự z nhất quán với mọi sự kiện occlusion quan sát được, bằng một
đồ thị ràng buộc nhỏ và topological sort. Có chu trình là bằng chứng phân rã sai, và khi đó phải hạ
giả thuyết xuống.

### 5.3 Fit hàm, không chỉ fit easing

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

### 5.4 Chọn candidate đầu tiên hay candidate tốt nhất

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

### 5.5 Dataset có ground truth và metric tầng hai

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

## 6. Quyết định và invariants

### 6.1 Quyết định đã chốt

Chốt với người dùng ngày 2026-09-28, 2026-10-29 và 2026-11-01.

| ID | Quyết định | Hệ quả thi công |
| --- | --- | --- |
| D1 | osu!lazer là chuẩn verify, stable ở mức best-effort | Không phát bất cứ thứ gì đã biết là lazer đọc sai. Value-chaining bị REJECT. `S` và `V` là hai property riêng (lazer nhân `Scale × VectorScale` [deepwiki]). Thứ chỉ rủi ro trên stable thì cảnh báo, không fail. |
| D2 | Sàn chất lượng cứng là mặc định; library trả report, không throw khi vi phạm | `VerificationReport.Passed = false` là kết quả hợp lệ, không phải exception. |
| D3 | Public của lõi gồm: API compile một video ghi vào **các channel do caller đưa, tách theo loại** (object, asset, tiến độ) và trả `Task<CompileSummary>` cho kết quả cuối; helper gom thành kết quả; renderer, verifier, storage. Encoder core để internal. Lõi không có writer `.osb`. | Xem mục 7.2 và 9.10. `ICandidateProvider` để internal, chưa mở plugin. Mỗi channel FIFO theo trình tự thời gian; channel object theo đúng thứ tự trong file storyboard (`Order` tăng dần), nên lớp ngoài ghi file nối tiếp từ đầu tới cuối mà không sắp lại. Backpressure do caller quyết bằng loại channel họ tạo. |
| D4 | Bỏ `.osbv`. Method chính chỉ xử lý **một** `VideoAnimation`. Command lồng trong video dùng object model của OsuParsers. Cache nằm ngay trong thư mục asset, chỉ theo nội dung. Caller chỉ định thư mục asset riêng để không lẫn với asset storyboard có sẵn. | Ranh giới public (vào và ra) là OsuParsers. IR nội bộ (`SbDocument`, …) chuyển thành `internal`. Đây là cách hòa giải D3 với D4: "IR public" ở D3 được hiểu là object model OsuParsers. OsuParsers object không mang layer hay thứ tự, nên mỗi object ghi ra kèm `Layer` và `Order`; channel object đã theo thứ tự file, kết quả gom có `AddTo(Storyboard)`. Thời gian: lõi nhận `StartOffset` (mốc ms trên storyboard mà frame tại `From` xuất hiện) và mọi thời gian trong object trả ra đã cộng sẵn offset đó (mục 9.2). |
| D5 | Trừu tượng hóa nơi lưu tối đa. Lõi không ghi file văn bản nào. Asset quản lý kiểu database: index trong memory, nội dung trên storage, pixel nạp lười hoặc async, không nạp dồn. | `IAssetStorage` là public (mục 8). `AssetIndex` chỉ giữ metadata. Renderer và verifier nạp pixel lười với cache LRU có trần theo byte. |
| D6 | Chỉ .NET 10 | `net10.0`, không multi-target. |
| D7 | `Fps` = số frame chương trình xử lý và biểu diễn mỗi giây. Ép video về fps đó rồi xử lý tiếp; frame bị loại được frame trước đại diện. | Resample bằng filter `fps=` của ffmpeg (đã có trong `FrameSource`). `null` nghĩa là fps nguồn. Fps lớn hơn nguồn chỉ tốn thời gian, không đổi output, nên không cần xử lý riêng. |
| D8 | Không CLI; lõi chỉ trích tách, không ghi format | Xóa `OsbMpeg.Cli` và test của nó. Ghi `.osb` nằm trong project ngoài `OsbMpeg.Osb` tham chiếu lõi, gồm writer và test round-trip văn bản; lõi không tham chiếu ngược. Harness đo nằm trong test project (mục 11). |
| D9 | Được đổi format, nhưng quy tắc đặt tên asset phải nhất quán và **chỉ phụ thuộc nội dung pixel**. Tối ưu truy vấn, không tạo file hoặc folder vô nghĩa. | Key = XXH3-128 trên (tag phiên bản, format pixel, kích thước, pixel cuối cùng được lưu). Bỏ seed `colors`. Animation lưu phẳng, không mỗi animation một folder. Không file index, không file phụ. |
| D10 | Một tài liệu thiết kế duy nhất; plan triển khai được sinh từ nó. Code v1 chỉ là kho thành phần, không phải mốc so sánh | Tài liệu không chứa bước, commit hay test. Mỗi nấc năng lực có điều kiện nhận đo trên pipeline mới (mục 12); không acceptance nào so với v1 (mục 18). |
| D11 | Nhiều tiến trình OsbMpeg chạy song song trên cùng thư mục asset phải an toàn | Hợp đồng "tạo nếu chưa có" nguyên tử (mục 8.1). Ghi file tạm rồi rename, không bao giờ ghi đè. |
| D12 | **Không có khái niệm object hay thực thể ngữ nghĩa.** Đơn vị làm việc là *thay đổi liên kết* (coherent change): vùng không-thời gian lớn nhất mà một primitive trong ladder giải thích được trong sàn. "Sprite" là hệ quả của việc một track Sim(2) giải thích được một vùng, không phải đầu vào. | Không có kênh prompt, không có bước "định danh object". Layer được *đề xuất* bằng tín hiệu motion (block flow, sequential RANSAC Sim(2), liên thông không gian, prior thời gian) và được *nhận* chỉ khi closed loop đo thấy rẻ hơn. Re-identification xuyên che khuất và xuyên scene là canonical hash của asset (I10), không phải ID của tracker. |
| D13 | **Lõi là .NET thuần; model học chỉ ở sidecar tùy chọn.** SAM 2/3, MatAnyone hoặc tương đương chạy trong tiến trình Python riêng, chỉ để *tinh chỉnh biên* (alpha matte) của layer đã được đề xuất bằng motion. Không có sidecar thì pipeline vẫn chạy đầy đủ. | Contract `ILayerBoundaryRefiner` (mục 10.2, sidecar ở mục 12). Sidecar chỉ được xây sau khi attribution (M3.5) chỉ ra bucket "mép layer" đủ lớn. Không thêm ONNX hay PyTorch vào assembly `OsbMpeg`. |
| D14 | **Ưu tiên nội dung được author**: motion graphics, UI, 2D animation. Game và footage vẫn phải không tệ đi (verify pass, bytes không tăng), nhưng không là mục tiêu tối ưu trong 6 tháng tới. | Thêm tier fixture `authored` (mục 11.2 và `tests/fixtures/README.md`). R1 (dataset có ground truth chương trình) đi ngay sau M1, R2 (easing rời rạc) ngay sau M2, motion layer K>1 là M2.5 với keep-gate đo trên tier `authored` (mục 12). |
| D15 | **Thứ tự: bộ đo trước, vòng kín sau, provider sau cùng.** Verifier, renderer và metric (M0) rồi hiệu chuẩn preset (M0.5) đi trước vòng kín (M1); mọi provider khác đi sau vòng kín. | Mọi số đo về layer, easing, additive đều là số đo *sau* verify. Không spike provider nào trên code v1. |
| D16 | **Các hướng đã bác bỏ có bằng chứng, không đề xuất lại** (bảng quyết định, mục 14): tracking ngữ nghĩa làm xương sống; freeze gate so với frame trước (vòng hở, chính là lỗi mờ của v1); nền động hay vùng lớn bằng `Animation` (R4); cubic Bézier và curve tham số tự do (không có trong `.osb`); dedupe theo pHash hay fuzzy không verify pixel (pop); cost function dự đoán với trọng số tay (đo thật thua 2,6 đến 5,4 lần); value-chaining và `[Variables]`. | Không viết code cho các mục này; muốn đảo thì phải có số đo nêu trong cột "điều gì đảo nó" của bảng đó. Những kỹ thuật được giữ cho M2.5: Sim(2) thay affine; `C` đóng dạng với master là frame sáng nhất; additive layer `Base + α(t)·Light` với base theo percentile, chỉ trong hệ sprite đã bù chuyển động; easing rời rạc + piecewise + chọn theo error + λ·complexity; canonical asset + instance; nền dựng từ pixel thật đã thấy (tích lũy theo mask, không ảo giác), pixel chưa từng thấy được lấp bằng khuếch tán và không bao giờ được tin (mục 12.6). |
| D17 | **Vật thể di chuyển làm lộ nền được xử lý theo hai nấc.** Nấc a (M2.5a): theo dõi vùng của X bằng quỹ đạo Sim(2), biểu diễn X bằng *một* sprite cắt theo bounding rect kèm track, chạy nhân quả; nền vừa lộ ra *không* được dựng riêng mà để closed loop vá bằng patch như mọi vùng bẩn khác, và patch đó được chèn **dưới** X. Nấc b (M2.5b): mask và alpha cho X, dựng nền nhìn trước cả đoạn, khử lem màu ở mép; chỉ xây khi số đo của nấc a cho thấy bucket `rect-halo` hoặc `reveal-strip` đủ lớn. | Không có pass tích lũy, không dựng nền, không alpha ở M2.5a. Cần ba thay đổi nền tảng ở M2 (7.3, 9.4) và chèn-dưới trong `ReconstructionState`/`EmitContext` (7.3). Gate theo đoạn bằng kiểm thử khô; bị từ chối thì output trùng byte với bước trước. |
| D18 | **Solver (CP-SAT, ILP) là thước đo trước khi là công cụ.** Dùng làm oracle offline (`OR`, 11.4) để đo ladder greedy bỏ phí bao nhiêu; chỉ được vào lõi ở bài toán ngân sách workload M3 (multiple-choice knapsack theo lát thời gian) khi oracle hoặc số vi phạm ngân sách đòi. Không dùng làm bộ chọn representation toàn cục trong lúc encode. | `Google.OrTools` chỉ được thêm vào project đo hoặc test, không vào assembly `OsbMpeg` cho tới khi S3.1b được duyệt. Các bài toán con có cấu trúc tốt hơn giữ thuật toán chuyên dụng: chia đoạn easing dùng DP, z-order K ≤ 4 duyệt hết, gán block vào layer dùng graph cut nếu cần, span crossfade dùng greedy. |
| D19 | **Asset luôn lossless.** Asset là pixel nguồn cắt nguyên vẹn; PNG giải mã lại phải trùng từng byte. Cấm lượng tử hóa, palette xấp xỉ, dither, JPEG, WebP hay AVIF lossy, giảm độ phân giải. Sàn chất lượng của preset chỉ áp cho cách biểu diễn, không cho asset. | Tối ưu kích thước chỉ bằng cách lossless: mức nén và filter PNG, bỏ kênh alpha khi ảnh đặc, ghi grayscale khi R = G = B ở mọi pixel, PNG indexed chỉ khi ảnh có không quá 256 màu phân biệt; mọi biến thể đều có test giải mã trùng từng byte (9.9). Asset luôn ở độ phân giải nguồn (R2). |

### 6.2 Invariants

Mọi code, hiện tại và tương lai, phải giữ. Vi phạm là bug kiến trúc, không phải lựa chọn phong cách.

- **I1.** IR nội bộ (`SbDocument`, `SbObject`, `SbCommand`) là ngôn ngữ chung duy nhất *bên trong*.
  Mọi representation, hiện có hay thêm sau, phát ra `SbSprite` hoặc `SbAnimation` cùng command chuẩn
  vào `SbDocument`; không có kênh phụ, không có "gợi ý cho renderer" nằm ngoài IR. Ranh giới public là
  OsuParsers, và chỉ có đúng một converter hai chiều (`OsuParsersConverter`).
- **I2.** Verification độc lập là thẩm quyền cuối. Verifier render đúng object OsuParsers đã ghi ra, đổi
  lại sang IR qua converter, tức đã qua làm tròn thời gian; vòng văn bản `.osb` thuộc test của
  `OsbMpeg.Osb`. Verifier verifier không dùng
  `ReconstructionState` in-memory của encoder. Encoder dùng display model riêng. Dùng chung thư viện
  tầng thấp (`Compositor`, `CommandEvaluator`) thì được, khác nhau ở orchestration và nguồn dữ liệu.
- **I3.** Chuỗi fallback luôn kết thúc ở `PatchProvider`, và provider cuối này không bao giờ fail vì nó
  chỉ crop từ frame nguồn. Mọi optimization fail thì rơi xuống bậc dưới, cùng lắm là patch. Đắt thì
  được, sai thì không.
- **I4.** Confidence và heuristic không bao giờ thay pixel check. Motion confidence, hash equality, RDP
  error, tracking score, mask của bất kỳ model nào chỉ dùng để *đề xuất* candidate. Mọi commit phải qua
  pixel-level evaluate đạt sàn trước. Candidate sống dài hơn `FrameWindow` (track của M2, layer của
  M2.5) không thể được kiểm trên frame tương lai trước khi commit: với chúng, kiểm thử khô trên bản
  downscale của cả đoạn thay cho kiểm trước commit, còn closed loop vẫn kiểm mọi frame full-res khi
  frame tới và vá chỗ sai (9.4). Không frame nào được ship mà chưa qua pixel check.
- **I5.** Mỗi property có một người ghi duy nhất, và `PropertyOverlapValidator` gate mọi output. Property
  theo lazer, tính theo từng `SbCommandKind` sau khi IR đã tách `M` thành `MoveX`/`MoveY` và `V` thành
  `VectorScaleX`/`VectorScaleY`; `S` và `V` là hai property riêng, validator chỉ cảnh báo khi cả hai
  cùng sống trên một object. Người ghi duy nhất cho command của output là `EmitContext.Finalize` (kể
  cả khi đi qua baker). Validator chạy sau khi flatten loop và làm compile fail khi phát hiện chồng lấn.
- **I6.** Sàn tuyệt đối theo từng frame và từng tile. Vượt ngân sách thì đổi representation hoặc báo
  violation, không bao giờ âm thầm hạ sàn.
- **I7.** Thêm representation mới nghĩa là thêm `ICandidateProvider` (và nếu cần, một analyzer đề xuất
  giả thuyết), không sửa lõi `ClosedLoopEncoder`, `ReconstructionState` hay verifier. Danh sách provider
  kèm rank là bắt buộc từ M1; ladder viết thành chuỗi if là vi phạm.
- **I8.** Mọi số tạm phải được calibrate bằng đo trước khi trở thành gate.
- **I9.** Chạy attribution sau mỗi milestone lớn; feature mới chỉ sinh ra từ dữ liệu bucket, không từ
  tên feature nghe hay.
- **I10.** Dedupe duy nhất là key nội dung của `AssetIndex` (mục 8.3). Mở rộng (chuẩn hóa pose, biến thể
  encoding) phải là mở rộng của key, không phải một cơ chế song song.
- **I11.** Library không bao giờ xóa hay ghi đè một asset đã tồn tại. Ngoại lệ duy nhất là file tạm của
  chính nó đã quá hạn (mục 8.2).
- **I12.** Lõi không có state toàn cục thay đổi được; lock duy nhất của OsuParsers nằm ở `OsbMpeg.Osb`. Cấu hình ffmpeg đi theo từng
  lần gọi, không dùng `GlobalFFOptions.Configure`.
- **I13.** Public surface bị khóa bằng snapshot test. Muốn thêm type hay member public thì phải sửa file
  approved một cách có chủ đích.
- **I14 (phủ kín).** Từ `StartOffset` tới hết video, mọi pixel trong rect của video phải được phủ bởi ít
  nhất một item opaque (alpha nội dung bằng 1). Trong library, video nằm trên storyboard của caller:
  chỗ nào không được phủ thì layer bên dưới hoặc background beatmap sẽ lộ ra. Những thứ tồn tại để giữ
  I14:
  - I-frame ở frame 0 và quy tắc cut (9.5). Agent **không được** "tối ưu" I-frame bằng cách bỏ qua các
    block trông như đã sạch.
  - `BlockOwnership` (9.7) chỉ đóng item khi block của nó đã có chủ mới.
  - Nền của `ReconstructionState` và của verifier là xám `(128,128,128)`, không phải đen. Nhờ vậy lỗ
    hổng trên vùng nguồn màu đen (letterbox, cảnh tối) trở thành lỗi đo được.

  Có test cấu trúc: sau frame 0, tại mọi frame, mọi block đều có chủ, kể cả với input có letterbox.
- **I15.** Lõi `OsbMpeg` không phụ thuộc model học, PyTorch hay ONNX (D13). Thứ cần model chạy trong
  tiến trình riêng và giao tiếp bằng file mask và alpha qua một contract internal; thiếu nó, pipeline
  vẫn chạy đầy đủ với provider classical.
- **I16 (lossless).** Mọi asset ghi qua `AssetIndex` giải mã lại trùng từng byte với pixel đã đưa vào. Có
  test kiểm điều này cho mọi biến thể nén. Không đường code nào được biến đổi pixel asset (D19).
- **I17 (không format).** Lõi không tham chiếu `OsbMpeg.Osb` và không chứa code sinh hay đọc văn bản `.osb`.
- **I18 (trình tự ghi).** Mỗi channel FIFO theo trình tự thời gian. Channel object ghi `Order` tăng ngặt, đúng
  thứ tự trong file storyboard; một object chỉ được ghi khi nó đã xong và mọi object có `Order` nhỏ hơn đã được
  ghi. Chèn ở dưới luôn cấp khóa lớn hơn mọi khóa đã ghi. Asset của một object luôn được ghi trước object đó.
  Tiến độ không bao giờ đi lùi.

### 6.3 Kiểm tra ngược: kiến trúc có mở không

Câu hỏi kiểm tra: tuần sau phát hiện một representation mới tốt hơn mọi cái hiện có, có thêm được như
một candidate mà không phải viết lại `ClosedLoopEncoder`, `ReconstructionState` hay verifier không? Có,
vì bốn lý do. Lõi encoder chỉ biết `ICandidateProvider` và `IRepresentationCandidate`, nên thêm nghĩa là
thêm class rồi đăng ký vào danh sách provider kèm rank. `ReconstructionState` composite từ *object IR*
trong display list chứ không từ "loại representation", nên nó render được candidate mới mà không cần
biết đó là gì; điều kiện duy nhất là primitive động mới nằm trong tập command mà `CommandEvaluator`
đánh giá được, và tập đó chính là toàn bộ command space của `.osb`. Verifier render từ object đã ghi ra nên
theo định nghĩa không biết representation nào sinh ra nó. Giả thuyết phân rã mới (layer
set, panorama, sau này là bất cứ gì) chỉ là một analyzer mới ghi thêm vào `SceneMotionInfo` cộng một
provider mới tiêu thụ nó.

---

## 7. Kiến trúc

### 7.1 Assembly và module

```text
OsbMpeg.slnx
src/OsbMpeg/OsbMpeg.csproj              net10.0, lõi trích tách; không biết cú pháp .osb
  *.cs (root, namespace OsbMpeg)        PUBLIC API (mục 7.2)
  Storage/                              public: IAssetStorage, FileSystemAssetStorage
                                        internal: AssetIndex, AssetKey, AssetEntry, PngLosslessWriter
  Ir/                                   internal: SbDocument, SbObject, SbCommand…, CanvasMapping,
                                        LoopFlattener, Utilities (IsEqual)
  Ir/Passes/                            internal: MergeAdjacentCommands, DropNoOpCommands, LoopExtractor
  Conversion/                           internal: OsuParsersConverter (IR ↔ object OsuParsers)
  Rendering/                            public: StoryboardRenderer
                                        internal: CommandEvaluator, EasingTable, Compositor, Canvas,
                                        SoftwareStoryboardRenderer, AssetPixelCache
  Media/                                internal: FrameSource, MediaProbe, MediaInfo, VideoFrame
  Analysis/                             internal: FlowField, MotionModelFitter, MotionSegmenter,
                                        LayerProposer, SegmentGate, TrajectoryFitter, EasingFitter
  Compilation/                          internal: GroupTransformBaker, CompilePipeline
  Encoder/                              internal: ClosedLoopEncoder, ReconstructionState, DisplayItem,
                                        BlockOwnership, FrameWindow, EmitContext, ThrashPlanner,
                                        WorkloadBudgets, Candidates/*
  Verification/                         internal: Metrics, ErrorMap, DirtyRects, QualityVerifier,
                                        WorkloadAnalyzer, PropertyOverlapValidator, CalibrationSimulator
src/OsbMpeg.Osb/OsbMpeg.Osb.csproj      net10.0, lớp ngoài, tham chiếu OsbMpeg
  *.cs (namespace OsbMpeg.Osb)          public: StoryboardWriter
                                        internal: OsbWriter, OsbReader, StoryboardDecoderGate
tests/OsbMpeg.Tests/                    một project test, tham chiếu cả hai project
tests/fixtures/                         fixture, xem tests/fixtures/README.md
tools/OsbMpeg.DatasetGen/               generator dataset có ground truth (mục 5.5, 11.2)
```

Không đặt namespace tên `OsbMpeg.Encoding`, vì nó che `System.Text.Encoding` trong mọi file thuộc namespace
`OsbMpeg.*`. Dùng `OsbMpeg.Encoder`. `ChannelWriter<T>` lấy từ `System.Threading.Channels` của BCL. `ILogger` lấy từ `Microsoft.Extensions.Logging.Abstractions`; `null` thì
dùng `NullLogger`. Ngoài các type ở mục 7.2, mọi type là `internal`, và public surface của cả hai assembly bị
khóa bằng snapshot test (I13). Lõi có `InternalsVisibleTo("OsbMpeg.Osb")` để lớp ngoài dùng converter và IR;
chiều ngược lại bị cấm (I17).

### 7.2 Public API

```csharp
namespace OsbMpeg;

// Một video cần compile. Tương đương "Compile(video, at, from, to, ...)".
public sealed record VideoAnimation
{
    public required string VideoPath { get; init; }        // đường dẫn tương đối resolve theo CWD, giống File API của .NET
    public required double StartOffset { get; init; }      // mốc ms trên storyboard mà frame tại From xuất hiện; mọi thời gian trả ra đã cộng sẵn
    public TimeSpan From { get; init; } = TimeSpan.Zero;   // điểm bắt đầu trong video
    public TimeSpan? To { get; init; }                     // null nghĩa là tới hết video
    public double? Fps { get; init; }                      // null nghĩa là avg fps của nguồn (D7)
    public StoryboardLayer Layer { get; init; } = StoryboardLayer.Background;
    public Vector2 Position { get; init; } = new(320, 240); // tâm video trong storyboard space, auto-cover; cũng là pivot của group transform
    public CommandGroup? Commands { get; init; }           // group transform (OsuParsers), thời gian tuyệt đối trên storyboard; null = không có
}

public sealed record CompileOptions
{
    public required IAssetStorage Storage { get; init; }    // gốc = thư mục beatmap (nơi osu! resolve đường dẫn)
    public required string AssetDirectory { get; init; }    // ví dụ "sb/osbmpeg"; dành riêng cho OsbMpeg; là tiền tố đường dẫn asset trong object
    public QualityPreset Quality { get; init; } = QualityPreset.High;
    public bool Verify { get; init; } = true;
    public FfmpegSettings Ffmpeg { get; init; } = new();
}

public sealed record FfmpegSettings(string? BinaryFolder = null, string? HardwareAcceleration = null);

// Số giữ chỗ; số thật chốt ở M0.5 (mục 11.1).
public sealed record QualityPreset(double PsnrFloorDb, double TileSsimFloor, double HeadroomDb = 2.0,
    double TileSsimHeadroom = 0.02, int HysteresisFrames = 2)
{
    public static QualityPreset High { get; }
    public static QualityPreset Medium { get; }
    public static QualityPreset Low { get; }
}

// Thread-safe: nhiều CompileAsync chạy đồng thời trên cùng một instance là hợp lệ.
public sealed class OsbMpegCompiler
{
    public OsbMpegCompiler(CompileOptions options, ILogger? logger = null);   // validate options ngay (mục 8.7)
    // Ghi object, asset, tiến độ vào các writer của caller (9.10); Task hoàn tất sau verify.
    public Task<CompileSummary> CompileAsync(VideoAnimation video, CompileOutputs outputs,
        CancellationToken cancellationToken = default);
    // Helper: tự tạo channel unbounded, gom object theo thứ tự file, chuyển tiến độ qua IProgress.
    public Task<VideoCompileResult> CompileToResultAsync(VideoAnimation video,
        IProgress<CompileProgress>? progress = null, CancellationToken cancellationToken = default);
    public Task<VerificationReport> VerifyAsync(VideoAnimation video, IReadOnlyList<CompiledObject> objects,
        ChannelWriter<CompileProgress>? progress = null, CancellationToken cancellationToken = default);
}

// Các channel do caller tạo và đưa writer vào; caller giữ reader. Caller chọn bounded hay unbounded,
// tức chọn encoder có phải chờ khi họ đọc chậm hay không. Lõi ghi mỗi channel từ đúng một luồng.
public sealed record CompileOutputs
{
    public required ChannelWriter<CompiledObject> Objects { get; init; }  // Order tăng ngặt = thứ tự trong file
    public ChannelWriter<StoredAsset>? Assets { get; init; }              // FIFO theo thời điểm asset được lưu
    public ChannelWriter<CompileProgress>? Progress { get; init; }        // FIFO theo thời gian, không đi lùi
    public bool CompleteWriters { get; init; } = true;                    // lõi gọi Complete() hoặc Complete(ex)
}

// Một object đã xong. Order là thứ tự khai báo = z-order trong layer = thứ tự trong file, và channel object
// ghi đúng theo nó. Order là khóa có khoảng trống, chỉ dùng để so sánh.
public sealed record CompiledObject(IStoryboardObject Object, StoryboardLayer Layer, long Order);

// Một asset được lưu hoặc dùng lại trong lần compile này; mỗi path ghi đúng một lần mỗi lần compile.
public sealed record StoredAsset(string Path, int Width, int Height, int FrameCount, long Length, bool Created);

public sealed record CompileSummary(CompileStatistics Statistics, WorkloadReport? Workload,
    VerificationReport? Verification);                                    // Verification null khi Verify = false

// Kết quả gom của CompileToResultAsync.
public sealed record VideoCompileResult(
    StoryboardLayer Layer,
    IReadOnlyList<CompiledObject> Objects,      // theo thứ tự file
    CompileStatistics Statistics,
    WorkloadReport? Workload,
    VerificationReport? Verification)           // null khi Verify = false
{
    public void AddTo(Storyboard storyboard);   // storyboard.GetLayer(Layer).AddRange(Objects)
}

public sealed record CompileStatistics(int FrameCount, double Fps, int SpriteCount, int AnimationCount,
    int CommandCount, int AssetsCreated, int AssetsReused, long AssetBytesCreated, long AssetBytesReferenced,
    TimeSpan EncodeElapsed, TimeSpan VerifyElapsed);

public enum CompilePhase { Probing, Encoding, Verifying }
public readonly record struct CompileProgress(CompilePhase Phase, int FramesDone, int FramesTotal,
    int SpriteCount, int AssetsCreated, int AssetsReused);

// 6
public sealed record VerificationReport(bool Passed, int FrameCount,
    double MinPsnr, double P1Psnr, double MeanPsnr,
    double MinTileSsim, double P1TileSsim, double MeanTileSsim,
    int ViolationCount, IReadOnlyList<QualityViolation> Violations);   // Violations giữ tối đa 1000 bản ghi đầu
public sealed record QualityViolation(int FrameIndex, double StoryboardTimeMs, double Psnr, double TileSsim,
    int WorstTileX, int WorstTileY);

// 5
public sealed record WorkloadReport(double PeakSteadySbLoad, double PeakTransientSbLoad, int PeakAliveSprites,
    long AnimationResidentBytes, long TotalAssetBytes, int OsbLineCount,
    IReadOnlyList<WorkloadSample> Timeline);   // mẫu mỗi 100 ms
public readonly record struct WorkloadSample(double TimeMs, int AliveSprites, double SteadySbLoad, double TransientSbLoad);

public class OsbMpegException : Exception { /* ctor chuẩn */ }
public sealed class VideoDecodeException : OsbMpegException { /* ctor chuẩn */ }

// ---- Project ngoài OsbMpeg.Osb, không thuộc lõi ----
namespace OsbMpeg.Osb;
public static class StoryboardWriter
{
    // Ghi 5 layer hình và SamplesLayer. Throw NotSupportedException nếu storyboard.Variables khác rỗng:
    // không âm thầm làm mất dữ liệu của caller, và R9 cấm tự phát [Variables].
    public static void Write(Storyboard storyboard, TextWriter writer);
}

namespace OsbMpeg.Rendering;
public sealed class StoryboardRenderer
{
    public StoryboardRenderer(Storyboard storyboard, IAssetStorage storage, int width, int height, bool passing = true);
    public void Render(double timeMs, Span<byte> rgb24);    // rgb24.Length == width*height*3
}

namespace OsbMpeg.Storage;
public interface IAssetStorage { /* mục 8.1 */ }
public sealed class FileSystemAssetStorage : IAssetStorage { public FileSystemAssetStorage(string rootDirectory); }
```

`ILogger` lấy từ `Microsoft.Extensions.Logging.Abstractions` (bản 10.0.x ổn định mới nhất), `null` thì
dùng `NullLogger`. Ngoài danh sách trên, mọi type khác là `internal` (I13).

### 7.3 Contract nội bộ

Chữ ký được phép chỉnh cú pháp, không được đổi ngữ nghĩa.

**Candidate và provider.** Lõi encoder chỉ biết hai interface này; thêm representation là thêm class (I7).

```csharp
internal sealed record DirtyRegion(PixelRect Rect, int ViolationStartFrame, int NowFrame);
internal sealed class EncodeContext { FrameWindow Window; ReconstructionState Reconstruction; QualityPreset Preset;
    bool DisjointMode; double Fps; AssetIndex Assets; ErrorMap Scratch; SegmentPlan Segment; }
internal interface IRepresentationCandidate
{
    CandidateKind Kind { get; }
    long EstimatedCostBytes { get; }                    // chỉ để sắp thứ tự trong một bậc
    int FirstAffectedFrame { get; }                     // frame đầu mà candidate làm đổi hiển thị
    void RenderPreview(int frameIndex, Span<byte> regionScratch);   // vùng Rect tại frame đó; không side effect
    Task<CommitResult> CommitAsync(EmitContext emit, CancellationToken ct);
}
internal interface ICandidateProvider
{
    int LadderRank { get; }
    IEnumerable<IRepresentationCandidate> Propose(DirtyRegion region, EncodeContext ctx);  // rỗng = không áp dụng
}
```

Rank: `Photometric = 10`, `TransformTrack = 20`, `MovingRect = 25`, `Crossfade = 30`, `Patch = int.MaxValue − 1`.
Danh sách provider đăng ký trong constructor của `ClosedLoopEncoder`, sắp theo rank; viết thành chuỗi `if` là
vi phạm I7. Provider ném exception thì được log và coi như trả rỗng, trừ `PatchProvider`: exception của
provider cuối không bao giờ bị nuốt, vì nó fail nghĩa là bug thật.

**Display model.** Trạng thái hiển thị mô phỏng là working truth của encoder, không phải thẩm quyền (I2).

```csharp
internal sealed class DisplayItem
{
    long Order;                               // khóa z có khoảng trống, so sánh được, cho phép chèn ở dưới
    PixelRect Rect;                           // footprint gốc trong không gian video
    byte[] Pixels;                            // RGBA, alpha theo pixel (mặc định 255)
    double StartMs; double EndMs;             // EndMs = +∞ khi còn mở
    PiecewiseLinear Alpha;                    // alpha toàn cục theo thời gian, mặc định 1
    PiecewiseLinearRgb Tint;                  // mặc định (255,255,255)
    PiecewiseSim2? Transform;                 // (x, y, s, θ) quanh tâm; null = căn pixel, không transform
    bool Additive;
}
internal sealed class ReconstructionState
{
    ReconstructionState(int width, int height);
    ReadOnlySpan<byte> Canvas { get; }                         // RGB24, nền xám (128,128,128) theo I14
    double TimeMs { get; }
    void AdvanceTo(double tMs);
    void Add(DisplayItem item, DisplayItem? below = null);     // chèn trên cùng, hoặc ngay dưới một item
    void Close(DisplayItem item, double endMs);
    void AppendTrack(DisplayItem item, …);                     // photometric, crossfade, transform
    IReadOnlyList<DisplayItem> Query(PixelRect r);             // lưới bucket 128 px
    void RenderRegion(PixelRect r, double tMs, IReadOnlyList<DisplayItem> extra, Span<byte> scratch);
    void Evict(double beforeMs);                               // bỏ item đã đóng quá cửa sổ, trả pixel
}
```

Công thức vẽ: `pixel = src × tint / 255`, alpha tổng `= itemAlpha × texelAlpha / 255`, rồi normal
`dst·(1−a) + pixel·a` hoặc additive `dst + pixel·a`, làm tròn half-away-from-zero về byte. Item có transform
được lấy mẫu bilinear tại tâm texel, clamp ở mép, giống renderer (mục 9.11). Item vẽ khi `StartMs ≤ t < EndMs`,
theo `Order` tăng dần. `AdvanceTo(t)` chỉ composite lại hợp các vùng của item "động": có track alpha, tint hay
transform đang nội suy, hoặc có `StartMs`, `EndMs` trong `(TimeMs, t]`; vùng của item di chuyển là hợp footprint
trước và sau. Invariant quan trọng nhất của encoder: sau mọi chuỗi `Add`, `Close`, `AppendTrack`, `AdvanceTo`,
canvas phải bằng từng byte với bản render lại toàn bộ display list tại `TimeMs` (cho phép ±1 ở item có
transform so với renderer IR).

**Bản đồ lỗi.**

```csharp
internal enum BlockState : byte { Clean, NearFloor, Violating }
internal sealed class ErrorMap
{
    ErrorMap(int width, int height, QualityPreset preset);   // block 16, tile 64, lưới giống TileSsim
    void Compute(ReadOnlySpan<byte> reconRgb, ReadOnlySpan<byte> sourceRgb);
    ReadOnlySpan<BlockState> Blocks { get; }
    int BlocksX { get; } int BlocksY { get; }
    double ViolatingFraction { get; }
}
```

**Phân tích đoạn và layer.**

```text
SegmentPlan            := [(t0, t1, CameraModel?, LayerProposal[], Gate kết quả)]
LayerProposal          := { supportMask(t) mức block, Sim(2) track, footprint(t), zHint }
ILayerProposer         := frames downscale → LayerProposal[]
ILayerBoundaryRefiner  := (frames, supportMask(t)) → alpha(t) trong dải biên   (chỉ ở M2.5b, mục 10.2)
```

**Người ghi duy nhất.** `EmitContext` là nơi duy nhất sinh sprite và command của output (I5). Nó giữ một
`OpenSprite` cho mỗi `DisplayItem` (asset, footprint, các track) và theo dõi điểm kết thúc của từng cặp
object và property, nên từ chối ngay command chồng lấn lúc ghi; validator cuối chỉ là lưới thứ hai.

### 7.4 Ai sở hữu state nào

| State | Chủ sở hữu | Vòng đời | Ghi chú |
| --- | --- | --- | --- |
| `AssetIndex` (metadata của asset) | `OsbMpegCompiler` instance | đời instance | dùng chung giữa các compile đồng thời; không giữ pixel |
| File asset | `IAssetStorage` | vĩnh viễn | library không bao giờ xóa hay ghi đè |
| `ReconstructionState`, `FrameWindow`, `EmitContext`, `BlockOwnership` | một lần `CompileAsync` | một video | không chia sẻ |
| IR nội bộ | `CompilePipeline` | một lần gọi | không lộ ra ngoài |
| `CompiledObject` đã ghi ra | caller | ngay khi ghi vào channel | lõi chỉ giữ bản IR riêng, không giữ tham chiếu tới object OsuParsers |
| Hàng đợi object đã xong nhưng chưa tới lượt | `EmitContext` | tới khi mọi `Order` nhỏ hơn đã ghi | chỉ object OsuParsers và IR, không pixel (9.10) |
| Channel và reader | caller | do caller quản | lõi chỉ giữ writer trong lúc compile, gọi `Complete` khi xong nếu `CompleteWriters` |
| Cache pixel của renderer | một renderer instance | một lần render hoặc verify | LRU có trần byte |
| Lock `StoryboardDecoderGate` | static trong `OsbMpeg.Osb` | process | chỉ ở lớp ngoài; decoder của OsuParsers không reentrant. Lõi không dùng decoder |

---

## 8. Asset storage và cache

### 8.1 Hợp đồng `IAssetStorage`

```csharp
public interface IAssetStorage
{
    // Độ dài byte nếu asset tồn tại (đã publish đầy đủ), null nếu chưa có. Đồng thời là phép kiểm Exists.
    ValueTask<long?> GetLengthAsync(string path, CancellationToken cancellationToken = default);

    // Mở để đọc. Throw FileNotFoundException nếu không có.
    ValueTask<Stream> OpenReadAsync(string path, CancellationToken cancellationToken = default);

    // Tạo nếu chưa có, NGUYÊN TỬ. Trả về true nếu lần gọi này tạo ra asset; false nếu asset đã tồn tại,
    // khi đó content KHÔNG được ghi và file cũ giữ nguyên. Reader không bao giờ được thấy asset dở dang.
    ValueTask<bool> TryCreateAsync(string path, ReadOnlyMemory<byte> content, CancellationToken cancellationToken = default);
}
```

`path` là đường dẫn tương đối, phân cách bằng `/`, không rooted, không có đoạn `.` hay `..`, không có
`\`. Mọi implementation phải validate và throw `ArgumentException` khi sai. Ghi rõ trong doc comment
của interface: implementation tùy biến *bắt buộc* thỏa tính nguyên tử và không ghi đè, vì cache xuyên
tiến trình dựa trên đúng điều đó.

### 8.2 `FileSystemAssetStorage`: thuật toán

- `GetLengthAsync`: dùng `FileInfo(full).Exists ? Length : null`.
- `OpenReadAsync`: `new FileStream(full, Open, Read, FileShare.Read | FileShare.Delete, 64 KiB, useAsync: true)`.
- `TryCreateAsync`:
  1. Nếu file đích đã tồn tại thì trả `false` ngay.
  2. `Directory.CreateDirectory(dir)`.
  3. Tạo file tạm cùng thư mục, tên `.osbmpeg-{Guid:N}.tmp`, ghi content, `Flush(true)`, đóng.
  4. `File.Move(tmp, final, overwrite: false)`.
  5. Nếu bước 4 gặp `IOException`: nếu `File.Exists(final)` thì xóa tmp và trả `false` (tiến trình khác
     thắng race, nội dung tương đương theo key). Nếu không, đó là sharing violation do antivirus hoặc
     indexer: retry tối đa 5 lần với delay 50·2^k ms, rồi throw.
  6. Trong `finally`: nếu tmp còn tồn tại (bị cancel, lỗi, hoặc thua race) thì xóa, nuốt lỗi khi xóa.
- **Dọn file tạm mồ côi** (tiến trình chết giữa bước 3): lần đầu một instance ghi vào một thư mục thì
  xóa các `.osbmpeg-*.tmp` có `LastWriteTimeUtc` cũ hơn 24 giờ, nuốt lỗi. Ngưỡng 24 giờ đủ xa để không
  bao giờ xóa file tạm đang ghi dở của một tiến trình khác còn sống.

### 8.3 Key và đường dẫn asset (D9)

- **Key** = XXH3-128 (`System.IO.Hashing.XxHash128`, seed 0) trên chuỗi byte sau:
  `"OSBMPEG-ASSET-V1"` (ASCII) ‖ kind (1 byte: 0 = sprite, 1 = animation) ‖ pixel format (1 byte:
  0 = Rgb24) ‖ width (int32 LE) ‖ height (int32 LE) ‖ frameCount (int32 LE, sprite = 1) ‖ pixel của từng
  frame.
- **Pixel được hash là pixel crop từ frame**, cũng chính là pixel file lưu, vì asset luôn lossless (D19).
  Cách nén PNG không nằm trong key: hai cách nén lossless của cùng pixel là cùng một asset.
- Kích thước nằm trong key để sửa bug hiện có: `AssetStore.GetOrAdd` hash mỗi byte pixel, nên hai crop
  đơn sắc khác kích thước (ví dụ 10×20 và 20×10 cùng màu đen) trùng key và dùng nhầm file.
- Tag phiên bản `V1` là hằng số cho mọi video. Đổi cách encode trong tương lai thì tăng `V2`; cache cũ
  không bị dùng nhầm.
- **Sprite:** `{AssetDirectory}/s/{key:x32}.png`.
- **Animation** (chỉ concept, M3): `{AssetDirectory}/a/{key:x32}.png` là base path ghi vào `.osb`, còn
  frame i nằm ở `{AssetDirectory}/a/{key:x32}{i}.png`. osu! suy frame bằng `Path.Replace(".", $"{i}.")`
  trên *toàn* đường dẫn; key hex có độ dài cố định 32 ký tự, nên dạng phẳng không va chạm. Không tạo
  folder riêng cho mỗi animation. Điều kiện là `AssetDirectory` không chứa dấu `.` (mục 8.7). Một
  animation được coi là tồn tại khi *mọi* frame tồn tại; frame nào thiếu (tiến trình chết giữa chừng) thì
  ghi bù frame đó.
- Không tạo file index hay manifest: storage *chính là* index, vì tên file suy trực tiếp từ key. Truy
  vấn là một `GetLengthAsync`, O(1) trên NTFS.
- Asset theo layout cũ (`s/{hash}.png` seed bằng `colors`) không được dùng lại và không bị xóa (I11).
  Người dùng tự xóa thư mục cũ nếu muốn.

### 8.4 `AssetIndex` (trong tiến trình)

```csharp
internal sealed class AssetIndex(IAssetStorage storage, string assetDirectory)
{
    // Trả entry cùng cờ Created: true nghĩa là CHÍNH lần gọi này đã publish file (tính vào AssetsCreated).
    Task<(AssetEntry Entry, bool Created)> GetOrCreateSpriteAsync(ReadOnlyMemory<byte> rgb24, int width, int height, CancellationToken ct);
    Task<(AssetEntry Entry, bool Created)> GetOrCreateAnimationAsync(IReadOnlyList<ReadOnlyMemory<byte>> frames, int width, int height, CancellationToken ct); // M3
    bool TryGet(string path, out AssetEntry entry);   // cho WorkloadAnalyzer và renderer (tra kích thước)
}
internal readonly record struct AssetEntry(string Path, int Width, int Height, long Length);
```

- Bảng là `ConcurrentDictionary<UInt128, Lazy<Task<AssetEntry>>>` (`LazyThreadSafetyMode.ExecutionAndPublication`).
  Luồng đầu tiên với một key làm việc thật: `GetLengthAsync`; nếu chưa có thì encode PNG
  (ImageSharp, `PngCompressionLevel` 6) rồi `TryCreateAsync`. Luồng đến sau với cùng key await chung
  task đó.
- Cờ `Created` chỉ đúng cho luồng đã thực sự tạo. Hiện thực bằng `TaskCompletionSource` riêng cho
  người tạo, hoặc bằng việc trả `(entry, createdByMe)` từ delegate của `Lazy`, rồi luồng khác nhận
  `Created = false`.
- Task bị lỗi hoặc cancel phải bị gỡ khỏi bảng (`TryRemove(KeyValuePair)`), để một lỗi tạm thời không
  bị cache vĩnh viễn.
- Bảng chỉ giữ metadata, khoảng 100 byte mỗi entry. Không giữ pixel, không giữ byte PNG.

### 8.5 Ma trận đồng thời

| Tình huống | Hành vi đúng | Cơ chế |
| --- | --- | --- |
| Hai `CompileAsync` đồng thời trên một `OsbMpegCompiler`, trùng nội dung | Encode và ghi đúng một lần | `Lazy<Task>` trong `AssetIndex` |
| Hai tiến trình (hoặc hai instance) cùng ghi một key | Một bên publish; bên kia nhận `false`, file không bị ghi đè | Temp rồi `File.Move(overwrite:false)` |
| Tiến trình B đọc đúng lúc A đang ghi | B thấy "chưa có" hoặc thấy file đủ, không bao giờ thấy file dở | Chỉ tên cuối cùng mới được đọc; rename là nguyên tử trên cùng volume |
| Tiến trình chết giữa lúc ghi | Chỉ còn lại file tạm mồ côi | Dọn sau 24 giờ (mục 8.2) |
| Cancel giữa compile | Không để lại file tạm, không để lại file dở | `finally` xóa tmp |
| File asset bị xóa từ bên ngoài giữa chừng | Index vẫn tưởng file còn (stale) | Verifier đọc qua storage nên lộ lỗi ngay; ghi rõ trong doc comment. Không tự sửa. |
| Antivirus giữ file mới tạo | Retry có backoff | Mục 5.2, bước 5 |

### 8.6 Quy tắc bộ nhớ

- `AssetIndex` chỉ giữ metadata.
- `ReconstructionState` giữ pixel của các `DisplayItem` *còn sống*. Pixel được copy từ frame lúc
  commit; item đóng quá một cửa sổ thì giải phóng.
- `FrameWindow` có trần theo byte, mặc định `min(2 s, 512 MiB)`, tối thiểu 3 frame.
- Renderer và verifier nạp asset lười qua `IAssetStorage.OpenReadAsync` rồi decode ImageSharp, với cache
  LRU trần theo byte decode, mặc định 256 MiB. Asset được nạp khi object lần đầu cần vẽ và bị loại theo
  LRU. Cache vô hạn `_assetCache` của renderer v1 không được giữ lại.
- Không API nào nạp toàn bộ asset của một video vào memory cùng lúc.

### 8.7 Validate `AssetDirectory`

Không rỗng. Chỉ gồm các đoạn `[A-Za-z0-9_-]+` nối bằng `/`. Không có dấu `.` (vì cách osu! đặt tên frame
animation), không có `..`, không có `/` đầu hoặc cuối, không có `\`. Sai thì `ArgumentException` ngay
trong constructor của `OsbMpegCompiler`. Doc comment ghi rõ: thư mục này dành riêng cho OsbMpeg; library
chỉ ghi vào `{AssetDirectory}/s/` (sau này thêm `/a/`), không đụng tới phần nào khác của storage.

---

## 9. Pipeline chi tiết, theo thời gian của một lần compile

Tổng quan:

```text
validate đầu vào, probe video                                              [9.1, 9.2]
group commands: flatten, sort, kiểm chồng lấn, quyết chế độ rời nhau       [9.3]
mỗi đoạn:
  phân tích downscale: flow, camera, layer, kiểm thử khô, gate             [9.4]
  frame đầu của đoạn: patch toàn khung, đặt track và layer đã nhận
  mỗi frame: AdvanceTo, ErrorMap, DirtyRects, ladder, commit               [9.5, 9.6, 9.7]
  quỹ đạo của track: RDP joint, easing rời rạc                             [9.8]
sau mỗi cảnh: gom thrash thành Animation, ngân sách workload               [9.9]
ghi object theo thứ tự file, asset và tiến độ theo thời gian, vào channel   [9.10]
verifier độc lập, workload, sự kiện cuối                                   [9.11]
```

### 9.1 Kiểm tra đầu vào

Sai thì `ArgumentException` hoặc `ArgumentOutOfRangeException` với thông báo nêu rõ trường, trừ khi ghi khác.

- `VideoPath` rỗng hoặc file không tồn tại: `FileNotFoundException`. File không có video stream:
  `VideoDecodeException`.
- `From < 0`; `To ≤ From`; `From ≥ Duration`. `To > Duration + 1 frame` thì clamp về `Duration` và log cảnh báo,
  không throw, vì duration của ffprobe thường lệch vài ms.
- `Fps` ≤ 0, lớn hơn 1000, hoặc NaN. `StartOffset` phải hữu hạn; âm được phép, vì osu! chấp nhận thời gian âm.
- Group commands: chồng lấn property là lỗi (9.3); trigger chứa lệnh ngoài `F`, `C`, `P,A` thì
  `NotSupportedException`; loop flatten ra hơn 1 000 000 command là lỗi, để chặn nổ bộ nhớ.
- `AssetDirectory` sai quy tắc ở 8.7: lỗi ngay trong constructor của `OsbMpegCompiler`.

Probe bằng `MediaProbe` với cấu hình ffmpeg theo từng lần gọi (I12) cho ra `fpsEff`, `W`, `H`, duration.

### 9.2 Thời gian, fps và làm tròn

- `fpsEff = video.Fps ?? MediaInfo.SourceFps`. Validate `0 < fpsEff ≤ 1000`. `StartOffset` phải hữu hạn;
  âm được phép, vì osu! chấp nhận thời gian âm.
- Frame i được lấy bằng `FrameSource` với filter `fps`, seek `From`, duration `To - From`.
- **Ngữ nghĩa bắt buộc (D7):** frame output i là frame nguồn *mới nhất* có `pts ≤ From + i/fpsEff`. Frame
  bị loại được frame trước đó đại diện, không bao giờ bởi frame sau.
  - Mặc định `round=near` của ffmpeg có thể lấy frame *sau* khi tỉ lệ
    không nguyên, ví dụ 60→24 hay 23,976→30.
  - Giá trị `round` (`zero`, `inf`, `down`, `up`, `near`, xem tài liệu filter `fps`) bằng
    test thực nghiệm. Tạo video mà frame k có mức xám k (ffmpeg `lavfi` + `geq` theo `N`), resample
    60→24, 60→30, 24→60 và 23,976→30, rồi khẳng định chỉ số frame nguồn đúng theo ngữ nghĩa trên. Ghi giá
    trị đã chọn và lý do vào doc comment của `FrameSource`.
- Thời gian local của frame i là `localMs(i) = i · 1000 / fpsEff` (double). Thời gian storyboard là
  `StartOffset + localMs(i)`. Frame cuối hiển thị tới `StartOffset + FrameCount · 1000 / fpsEff`, với
  `FrameCount` là số frame decode thực nhận được.
- IR giữ double. Làm tròn chỉ xảy ra **ở một chỗ**: `OsuParsersConverter.ToOsuParsers`, theo quy tắc
  `(int)Math.Round(ms, MidpointRounding.AwayFromZero)`. Làm tròn
  đơn điệu nên track rời nhau vẫn rời nhau. Verifier chạy trên object *đã* làm tròn.
- Verifier lấy mẫu tại **giữa frame**: `t_i = StartOffset + (i + 0.5) · 1000 / fpsEff`. Ranh giới làm tròn
  lệch tối đa 0,5 ms, còn điểm giữa frame cách ranh giới ít nhất 0,5 ms (tại fps ≤ 1000), nên không
  bao giờ lấy mẫu trúng ranh giới.
- Encoder đánh giá tại `localMs(i)`, dùng đúng frame i, không lệch.

### 9.3 Group transform và chế độ rời nhau

`VideoAnimation.Commands` là nhóm command áp cho cả video, như thể video là một sprite. Nó được đổi sang IR,
flatten loop, sắp theo thời gian, rồi qua `PropertyOverlapValidator`: với mỗi property có giá trị (`Fade`,
`MoveX`, `MoveY`, `Scale`, `VectorScaleX`, `VectorScaleY`, `Rotate`, `Colour`), hai command chồng nhau quá
0,5 ms là lỗi, command tức thời nằm bên trong khoảng của command khác là lỗi, hai command tức thời cùng thời
điểm là lỗi. `Scale` và `VectorScale*` cùng sống chỉ là cảnh báo (rủi ro trên stable, D1). Lỗi ở input là
`ArgumentException`, vì stable và lazer sẽ hiển thị khác nhau. Cùng validator chạy lại trên output trước khi
ghi ra (9.10); lỗi ở đó là bug nội bộ.

Mọi sprite sinh ra đều mang group transform qua `GroupTransformBaker`, theo đúng một người ghi (I5): track nội
dung và track của group được ghép thành một track mỗi property, không bao giờ thành hai track cạnh nhau.
Group không có `F` thì track alpha nội dung đi thẳng thành `F`; group không có `C` thì track tint đi thẳng thành
`C`; group có `C` thì lấy mẫu từng frame và nhân `groupColour × contentTint / 255`. Track transform của nội
dung (camera, layer) được ghép với group bằng cách lấy mẫu: với mỗi frame t, lấy state của group tại t, compose
với transform nội dung, ra chuỗi state cuối, rồi nén bằng RDP joint một lần (9.8). Lấy mẫu thay vì đại số vì
tích của hai track tuyến tính là bậc hai.

**Chế độ rời nhau** bật khi group có `F` khác 1 ở bất kỳ đâu hoặc có `P,A`. Khi đó mọi sprite sinh ra mang
fade hoặc blend cộng của group; hai sprite chồng nhau lúc alpha nhỏ hơn 1 bị phơi sáng hai lần, sprite cộng thì
cộng dồn. Trong chế độ này: gom vùng bẩn thay bằng lưới tile 128 px cố định, mỗi tile đúng một sprite sống;
crossfade trả rỗng. Chế độ được log kèm lý do.

### 9.4 Phân tích đoạn

Chạy trên một lượt decode downscale (luma, cạnh lớn nhất tối đa 256 px), trước vòng lặp full-res của đoạn đó.
Scene không có chuyển động đáng kể chỉ cần một lượt decode.

1. **Flow.** Block matching 16×16, SAD trong bán kính ±12 px, thêm 3 mức pyramid hoặc DIS flow nếu spike flow
   (OQ-2) đòi, theo đúng thứ tự đó; learned flow bị bác (4.6). Trên nội dung ít texture, Sim(2) của một vùng
   được ước lượng bằng ECC trực tiếp trên pixel thay vì từ flow, khởi tạo bằng flow block và RANSAC.
2. **Camera.** RANSAC similarity `(tx, ty, s)`, 2 điểm mỗi mẫu, inlier là residual dưới 1 px ở full-res, 64 vòng.
   Interface trả một layer-set ngay từ đầu (4.1). Mô hình chỉ được nhận khi cả hai tầng đạt: ít nhất 60% block
   là inlier ở ít nhất 70% số frame, và diện tích phần camera không giải thích được trung bình không quá 25%
   khung. Tầng thứ hai tồn tại vì riêng inlier ratio có thể nhận một nền mà 40% frame vẫn phải patch mỗi frame
   chồng lên một sprite nền cũng đang trả tiền. Nguồn có cạnh vượt 3840 thì không nhận, vì sprite nền ở độ phân
   giải nguồn chạm `MaxTextureSize`.
3. **Segment.** Đoạn có tốc độ nhỏ trong ít nhất 500 ms là đoạn tĩnh; chuyển trạng thái có độ trễ 300 ms. Hard cut
   là ranh giới cứng: ở vòng full-res, frame có từ 70% block vi phạm cùng lúc là cut (9.5).
4. **Layer** (M2.5a, mục 10.1). Sequential RANSAC Sim(2) tìm tối đa K ≤ 4 mô hình, lọc theo liên thông không gian
   với diện tích tối thiểu, dùng gán nhãn của frame trước làm prior cho frame sau. Số layer quyết bằng matching
   pursuit: thêm layer khi lợi ích biên đo trên bản downscale lớn hơn chi phí biên.
5. **Gate theo đoạn, quy tắc commit trước, vá sau.** Candidate sống lâu hơn cửa sổ frame (track camera, layer)
   không thể được kiểm trên frame tương lai trước khi commit (I4). Với chúng, kiểm thử khô thay thế: render
   composite dự kiến ở bản downscale cho mọi frame của đoạn, đo diện tích phần dư, chỉ nhận khi lợi ích ròng
   ước lượng dương và phần dư dưới ngưỡng. Sau commit, vòng full-res vẫn kiểm từng frame và vá chỗ sai, nên không
   frame nào được ship mà chưa qua kiểm pixel. Nếu bytes vá thực tế của đoạn vượt bytes ước lượng của phương án
   không có candidate đó, ghi vào report để hiệu chỉnh gate; không quay lui trong đoạn. Đoạn bị từ chối mọi
   candidate dài hạn cho output trùng byte với khi không có tầng phân tích.

### 9.5 Vòng lặp từng frame

```text
frame 0 của đoạn: mọi block là Violating (I-frame), chỉ PatchProvider; rồi đặt track và layer đã nhận ở 9.4
với mỗi frame i:
    Window.Push(frame i); R.AdvanceTo(localMs(i)); E.Compute(R.Canvas, frame i)
    nếu E.ViolatingFraction ≥ 0.7: cut
        → mọi block thành dirty, chỉ PatchProvider, không lùi thời điểm
    ngược lại:
        dirty = Violating                                 # xử lý ngay, không debounce
        ripe  = NearFloor liên tục đủ HysteresisFrames frame
                → dirty, ViolationStartFrame = frame NearFloor đầu tiên
    rects = DirtyRects(dirty) (hoặc lưới tile 128 trong chế độ rời nhau), sắp diện tích giảm dần
    mỗi rect: ladder (9.6), commit candidate đầu tiên đạt sàn (9.7)
    debug: tính lại ErrorMap trên các rect vừa sửa, không còn Violating
    R.Evict(localMs(i) − cửa sổ); ghi tiến độ mỗi 10 frame
cuối video: EmitContext.FinalizeAll
```

- **FrameWindow.** Ring buffer frame thuê từ pool, dung lượng `max(3, min(ceil(2·fps), floor(512 MiB / (W·H·3))))`.
  Frame ngoài cửa sổ không truy cập được; đó cũng là trần tự nhiên của span crossfade.
- **ErrorMap.** `blockPSNR` từ MSE RGB của block 16×16, khớp `Metrics.Psnr`. Block dưới sàn `P` là `Violating`,
  dưới `P + HeadroomDb` là `NearFloor`. Theo tile 64×64 trên luma: SSIM dưới `S` thì mọi block của tile thành
  `Violating`; dưới `S + TileSsimHeadroom` thì block đang `Clean` của tile thành `NearFloor`. Block mép tính trên
  phần nằm trong frame.
- **Độ trễ chỉ áp cho headroom.** Sàn tuyệt đối nghĩa là không frame nào được ship dưới nó, kể cả trong cửa sổ
  chờ; vì vậy block vi phạm luôn được xử lý ngay. Block cận sàn chờ đủ hai frame để chống nhấp nháy, và khi đó
  patch lùi thời điểm bắt đầu về frame cận sàn đầu tiên, frame còn trong cửa sổ.
- **DirtyRects.** Thành phần liên thông 4 hướng trên block bẩn, lấy bbox; bbox có tỷ lệ lấp dưới 0,5 và diện tích
  trên 64 block thì tách một lần theo hàng hoặc cột trống dài nhất; mỗi cạnh tối đa 1024 px, vượt thì chia đều
  căn bội 16; rect nhỏ hơn 32×32 px được gộp vào rect kề cách dưới 32 px, vì overhead PNG khoảng 1,5 KB mỗi
  asset; cắt theo biên frame.

### 9.6 Ladder và các provider

**Quy tắc đánh giá (I4).** Với mọi frame f trong `[FirstAffectedFrame, NowFrame]`: `RenderPreview`, rồi tính
`blockPSNR` và `TileSsim` trên đúng vùng đó so với nguồn của frame f. Mọi frame đạt sàn thật thì candidate đạt.
Provider thử theo rank tăng dần; trong một provider, candidate thử theo chi phí ước lượng; nhận cái đầu tiên đạt.
Provider được tự quyết không propose dựa trên gợi ý rẻ, ví dụ crossfade thấy vùng đổi liên tục thì trả rỗng ngay
để khỏi tốn evaluate. Bỏ propose chỉ có thể làm chậm hoặc tốn hơn, không thể làm sai.

**Photometric (rank 10, T8).** Chỉ khi `Query(region)` trả đúng một item X phủ ≥ 90% region, region phủ ≥ 90%
footprint của X, X đặc, và không item động nào khác X giao region trong khoảng đang xét. Fit theo từng kênh:
`c = 255·Σ(A·T)/Σ(A²)` với A là pixel gốc của X, T là nguồn trên footprint của X. Kênh nào cần `c > 255` thì trả
rỗng, vì `C` chỉ làm tối được. Candidate là track tint đi tuyến tính từ tint hiện tại tại `ViolationStartFrame`
tới `c` tại `NowFrame`. Chỉ dùng `C`, không dùng `F`: video nằm trên nội dung khác của storyboard, nên `F` làm lộ
phần bên dưới thay vì cho ra màu tối như nguồn. Không tạo asset mới.

**TransformTrack (rank 20, T4).** Chỉ propose khi đoạn có mô hình camera đã nhận và region thuộc vùng nền chưa có
base đúng. Sprite nền cắt tại keyframe đầu đoạn, origin Centre, track `MX`, `MY`, `S` giải từ
`screen = p(t) + s(t)·(x − c)`, ghép group qua baker (9.3). Mép lộ ra khi pan không thuộc nền, đi theo ladder
thường và thành patch, đúng như mong đợi. Khi phần nền không bị patch đè suy thoái tới cận sàn trên hơn 40% diện
tích thì mở keyframe nền mới, chuyển bằng crossfade toàn khung, tức tái dùng cơ chế sẵn có.

**MovingRect (rank 25, T5).** Sprite di chuyển cho một layer đã nhận ở 9.4; chi tiết ở mục 10.1.

**Crossfade (rank 30, T3).** Chỉ khi không ở chế độ rời nhau, region đã có item (không phải I-frame), region không
vi phạm liên tục trong `HysteresisFrames + 2` frame gần nhất (liên tục là thrash), và không item động nào giao
region trong cửa sổ.

```text
A = canvas hiện tại trên region (tĩnh trong cửa sổ, theo điều kiện trên)
B = crop nguồn tại NowFrame
thử t_s ∈ {Now−w, Now−w/2, Now−w/4, …}, w = Now − Window.OldestIndex, tối thiểu 2 frame,
     không sớm hơn Start của các item đang hiển thị region
    mọi f ∈ [t_s, Now]: lerp(A, B, (f − t_s)/(Now − t_s)) đạt sàn so với nguồn f?
    đạt đầu tiên → item B khai báo sau, alpha 0 tới 1 trên [t_s, Now]; chủ cũ đóng tại Now
không span nào đạt → rỗng
```

Kiểm lại quá khứ là hợp lệ: các frame trong `(t_s, Now)` trước đó đã đạt với A tĩnh, và composite mới chỉ được
nhận khi cũng đạt. Quyết định chỉ dựa trên frame thật trong cửa sổ, không bao giờ dựa trên mẫu. Overdraw của
crossfade tính vào ngân sách transient riêng: một dissolve toàn khung đưa cả frame vào hai lớp cùng lúc là hành vi
đúng.

**Patch (rank cuối, T1, T2).** Propose theo thứ tự: bản lùi thời điểm (chỉ khi `ViolationStartFrame < NowFrame`:
crop tại `NowFrame`, bắt đầu từ `ViolationStartFrame`), rồi bản không lùi (crop tại `NowFrame`, bắt đầu tại
`NowFrame`). Bản sau luôn đạt vì preview bằng nguồn từng byte, nên chuỗi fallback luôn kết thúc (I3). Item mở
với `EndMs = +∞`, đóng bởi quyền sở hữu (9.7) hoặc ở cuối video.

### 9.7 Commit

1. **Asset.** Pixel crop đi qua `AssetIndex.GetOrCreateSpriteAsync` (mục 8): tra key, chưa có thì ghi PNG lossless
   nhỏ nhất qua `TryCreateAsync` nguyên tử, rồi ghi `StoredAsset` vào channel asset nếu path đó chưa được
   ghi trong lần compile này.
2. **Ghi.** `EmitContext` mở `OpenSprite` mới hoặc nối track vào sprite có sẵn; chồng lấn property bị từ chối ngay.
3. **Thứ tự z.** Item mới mặc định nằm trên cùng. Patch vá nền có rect không giao footprint của bất kỳ layer di
   chuyển nào trong khoảng sống của nó được chèn ngay dưới layer thấp nhất (`Add(item, below)`); patch giao
   footprint thì nằm trên. Nếu không, patch vá nền sẽ che layer khi layer quay lại và sinh vòng patch chồng patch.
4. **Quyền sở hữu block.** `owner[block]` là item trên cùng, đặc, phủ trọn phần trong frame của block. Commit patch
   đặc: mỗi block nó phủ trọn chuyển chủ, chủ cũ hết block thì `Close` tại thời điểm bắt đầu của patch (đúng cả
   khi patch được lùi thời điểm, vì patch phủ kín chủ cũ). Commit crossfade: chủ cũ hết block đóng tại thời điểm
   kết thúc fade. Photometric không đổi quyền sở hữu. Item có transform hoặc alpha theo pixel không bao giờ làm
   chủ block và không bao giờ bị đóng bởi quyền sở hữu; nó đóng ở cuối đoạn hoặc khi gate gỡ nó, và item tĩnh bị
   nó che một phần không bị đóng. Item bị che một phần vẫn sống; đỉnh overdraw có trong báo cáo workload. Đây là
   cơ chế giữ I14: item chỉ đóng khi block của nó đã có chủ mới.
5. **Áp dụng.** `ReconstructionState.Add` hoặc `AppendTrack`, composite ngay vùng bị ảnh hưởng.

### 9.8 Quỹ đạo: RDP joint và easing rời rạc

**RDP joint.** Điểm là `(tx, ty, s)` (thêm θ khi có xoay) mỗi frame; khoảng cách là dịch chuyển lớn nhất của
bốn góc khung dưới transform đã compose, so giữa đường nội suy và giá trị thật, đo bằng osu!px, epsilon tạm 0,3.
Không chạy RDP riêng từng thành phần với epsilon theo đơn vị vị trí: sai số scale Δs tốn `Δs·|x − c|` px, lớn nhất
ở góc ảnh và gần như vô hình nếu đo trên (tx, ty). Simplify joint cũng giữ breakpoint của `MX`, `MY`, `S` đồng
bộ. Với `p(u) = p₀ + uΔp` và `s(u) = s₀ + uΔs`, vị trí màn hình của một điểm ảnh cố định x là
`y(u) = p₀ + s₀(x−c) + u·[Δp + Δs(x−c)]`, tuyến tính theo u trên mỗi đoạn, nên nội suy tuyến tính đồng thời `M`
và `S` là đúng; chỉ phép đo sai số phải nằm trong screen space.

**Easing rời rạc (EZ).** Tập hàm là 35 easing của osu! (0 tới 34), không tham số tự do ngoài `(v0, v1)` của
segment; cubic Bézier, spring hay curve tùy ý không tồn tại trong `.osb`. Bắt đầu với chín hàm: Linear, Quad và
Cubic In/Out/InOut, Sine In/Out; thêm Elastic, Back, Bounce khi dataset chỉ ra. Với mỗi đoạn giữa hai breakpoint
ứng viên, đánh giá mọi easing với residual dạng đóng trong screen space (cùng metric bốn góc); nhận easing đạt
epsilon thay vì chèn keyframe. Chia đoạn bằng changepoint tối ưu (PELT hoặc DP trên số sample mỗi track; cost là
residual của easing tốt nhất trên đoạn, penalty là một command), chọn theo `Error + λ·Complexity` với fidelity là
ràng buộc cứng. Không easing nào đạt thì giữ RDP tuyến tính; trên quỹ đạo nhiễu kết quả trùng RDP. Áp cho mọi
track vô hướng: x, y, scale, alpha, colour (Photometric và Crossfade dùng chung fitter). Lý do và giới hạn ở 5.3.

### 9.9 Sau mỗi cảnh

**Animation cho thrash (T7).** Pass hậu kỳ mỗi cảnh, không nằm trong vòng frame. Quét lịch sử commit theo vùng:
vùng có ít nhất 4 hard swap liên tiếp cách nhau đúng một frame, tổng span tối đa khoảng 120 frame, diện tích không
quá tile 256 px, tổng bộ nhớ resident không quá 200 MB mỗi cảnh, và ít nhất 80% frame khác nội dung, thì gom chuỗi
patch thành một `Animation` LoopOnce và thay các sprite patch tương ứng. Vùng chỉ nhấp nháy giữa hai nội dung
không thành animation, vì dùng lại sprite rẻ hơn. Animation lưu phẳng `a/{key}{i}.png` (8.3). Không bao giờ dùng
`Animation` cho nền động hay vùng lớn (R4).

**Ngân sách workload.** Bộ đếm cập nhật tại mỗi commit: timeline tải steady và transient, số sprite sống, bộ nhớ
animation resident. `Allow(candidate)` kiểm dự phóng sau commit so với ngân sách ở 1.3; không đạt thì thử candidate
hoặc provider tiếp theo, tức đổi representation chứ không hạ sàn (I6). Patch rời nhau theo thời gian nên hiếm khi
bị chặn; nếu vẫn bị, vẫn commit và ghi violation, vì fidelity thắng.

**Giải ngân sách bằng solver (INVESTIGATE, D18).** Khi một đoạn vượt, chọn đổi representation ở vùng nào là
multiple-choice knapsack với ràng buộc dung lượng theo từng lát thời gian: biến `x[r,k]` chọn candidate k *đã đo*
cho vùng r, `Σk x[r,k] = 1`, tối thiểu tổng bytes, với mọi lát t: `AliveSprites(t) ≤ 500`, `SbLoad(t) ≤ 4.0`,
resident ≤ trần. Chỉ giải trên đoạn vượt nên tập nhỏ. Chỉ xây khi greedy để lại vi phạm hoặc oracle (11.4) cho
thấy greedy ở đoạn vượt bỏ phí từ 10%.

**Nén PNG lossless (D19).** `AssetIndex` thử các cách ghi lossless cho mỗi asset mới và giữ bản nhỏ nhất: mức nén
zlib và chiến lược filter PNG; RGB khi mọi alpha là 255; grayscale khi R = G = B ở mọi pixel; indexed khi ảnh có
không quá 256 màu phân biệt, với bảng màu chính xác. Mỗi biến thể được giải mã lại và so từng byte với pixel đầu
vào trước khi ghi; lệch thì bỏ biến thể và log như bug. Key không đổi (8.3), vì cách nén không phải nội dung. Việc
này xảy ra ngay lúc asset được tạo, không phải pass riêng.

### 9.10 Ghi kết quả ra channel

Ba channel tách theo loại, writer do caller đưa qua `CompileOutputs`. Lõi ghi bằng `WriteAsync`, nên khi caller
dùng channel bounded và đọc chậm thì encoder chờ, còn unbounded thì không bao giờ chờ; đó là lựa chọn của caller.
Mỗi channel FIFO và đúng trình tự thời gian (I18).

**Object: theo đúng thứ tự trong file storyboard.** Channel object ghi `Order` tăng ngặt, tức thứ tự khai báo,
cũng là z-order và thứ tự dòng trong `.osb`. Lớp ngoài chỉ cần ghi nối tiếp từ đầu tới cuối file. Một item được
ghi khi đủ hai điều kiện:

1. **Nó đã xong:** đã đóng và không còn bước nào sửa được nó. Khi chưa có pass gom thrash, đó là lúc item đóng.
   Khi có, pass đó có thể thay item trong cảnh, nên item chỉ xong ở cuối cảnh. Item còn mở ở cuối video xong bởi
   `FinalizeAll`.
2. **Tới lượt nó:** mọi item có `Order` nhỏ hơn đã được ghi.

Cơ chế: `EmitContext` giữ một hàng đợi sắp theo `Order` gồm các item đã xong nhưng chưa tới lượt, và một mốc là
`Order` nhỏ nhất trong số item chưa ghi, kể cả item còn mở. Mỗi khi một item xong, các item đầu hàng đợi có
`Order` nhỏ hơn mốc được ghi lần lượt. Chèn ở dưới không phá thứ tự: khóa mới nằm giữa item đích, còn mở nên chưa
ghi, và item liền trước nó, nên luôn lớn hơn mọi khóa đã ghi.

Hệ quả cần biết: item nằm dưới và sống lâu, như patch toàn khung ở frame đầu của đoạn, sprite nền của camera hay
một layer, chặn mọi item khai báo sau nó cho tới khi nó đóng. Với video có một vùng nền không bao giờ đổi, item
đó sống tới cuối và object gần như chỉ ra ở cuối lần compile. Thứ bị giữ lại là object OsuParsers và IR của chúng,
vài trăm byte mỗi object, không có pixel. Không có cách đóng sớm giả, vì thời điểm kết thúc của một item chỉ biết
khi nó đóng.

Mỗi item trên đường ghi đi qua:

```text
EmitContext.Finalize → IR (baker nếu có group commands)
  → passes theo object: MergeAdjacentCommands, DropNoOpCommands, LoopExtractor
  → PropertyOverlapValidator (lỗi = bug nội bộ: OsbMpegException)
  → OsuParsersConverter.ToOsuParsers (làm tròn thời gian một lần, 9.2)
  → hàng đợi theo Order → Objects.WriteAsync(CompiledObject(object, layer, order)) khi tới lượt
```

Không baker: sprite `Origin = TopLeft`, vị trí `mapping.PixelToStoryboard(rect.X, rect.Y)`, một `S` hằng bằng
`mapping.StoryboardScale`, cộng `F` từ track alpha nếu khác hằng 1 và `C` từ track tint nếu khác hằng trắng. Có
baker: `Origin = Centre`, tâm footprint đổi qua mapping, rồi `baker.Bake(...)` như 9.3. Converter ghép
`VectorScaleX`/`VectorScaleY` thành một `V`, giữ `MoveX`/`MoveY` ở dạng `MX`/`MY`, loop thành `LoopCommand`,
trigger thành `TriggerCommand`. Lõi giữ bản IR của các object đã ghi ra (vài trăm byte mỗi object, không pixel)
tới hết lần gọi để `WorkloadAnalyzer` và verifier dùng; nó không giữ tham chiếu tới object OsuParsers đã giao.

**Asset: theo thời điểm được lưu.** Mỗi asset được tạo mới hoặc dùng lại lần đầu trong lần compile được ghi một
`StoredAsset`, đúng thứ tự xảy ra. Asset được lưu lúc commit, còn object chỉ ghi khi xong, nên asset của một object
luôn được ghi trước object đó.

**Tiến độ: theo thời gian.** `Probing` một lần, `Encoding` mỗi 10 frame với `FramesDone` không giảm, `Verifying`
trong lúc verify. Caller muốn chỉ giữ bản mới nhất thì dùng channel bounded bỏ bản cũ nhất.

**Kết thúc.** `Task<CompileSummary>` hoàn tất sau verify. Nếu `CompleteWriters` là true, lõi gọi `Complete()` trên
mọi writer trước khi `Task` hoàn tất, nên reader của caller kết thúc tự nhiên. Lỗi hay cancel: mọi writer nhận
`Complete(exception)` và `Task` fault hoặc bị hủy với cùng exception; không còn file tạm. Writer asset và tiến độ
là tùy chọn; writer object là bắt buộc.

### 9.11 Kiểm chứng độc lập, metric và workload

Hai phía, render bằng **cùng** renderer IR, trên cùng canvas:

- **Canvas:** W×H (độ phân giải nguồn), với `CanvasMapping(W, H, Position.X, Position.Y)`, tức mapping
  auto-cover của chính video, nền **xám `(128,128,128)`** cho cả hai phía (I14). Nền xám làm lộ cả ba
  loại lỗi: lỗ hổng phủ, phơi sáng hai lần khi sprite chồng nhau dưới alpha < 1, và cộng dồn additive.
- **expected(t_i):** một sprite ảo. Texture là frame i. `Origin = Centre`, `X/Y = Position`. Commands
  là group commands (IR, đã flatten loop và sort). Cộng thêm một **intrinsic scale** bằng
  `CanvasMapping.StoryboardScale`: đây là property internal mới trên `SbObject`, mặc định 1, renderer
  nhân vào scale. Chỉ verifier dùng nó.
  - Khi không có group command, expected chính là frame i, pixel-exact (bilinear tại tỉ lệ 1:1 là
    copy).
  - Cách dựng này **không** đi qua `GroupTransformBaker`, nên bug của baker sẽ lộ ra.
- **actual(t_i):** các `CompiledObject` đã ghi ra, theo thứ tự file, đi qua `OsuParsersConverter.ToIr`, rồi
  render. Đây là đúng object caller nhận, đã làm tròn ms. Vòng ghi văn bản rồi đọc lại được kiểm riêng ở
  test của `OsbMpeg.Osb`.
- **So sánh:** `Metrics.Psnr` trên cả frame và `Metrics.TileSsim(...).Min`. Vi phạm khi
  `psnr < Preset.PsnrFloorDb` hoặc `tileMin < Preset.TileSsimFloor`.
- **Tính nhất quán với encoder:** encoder bảo đảm mọi block 16×16 đạt `blockPSNR ≥ P`. Vì MSE của frame
  là trung bình MSE của các block, `PSNR_frame ≥ min(blockPSNR) ≥ P`. Encoder cũng dùng đúng
  `TileSsim` với cùng lưới. Do đó output đúng *phải* pass verify, trừ các sai lệch nhỏ do làm tròn tọa
  độ float và do bilinear ở mép patch khi group có rotate hoặc scale khác 1 (13.2).
- Trigger bị bỏ qua ở cả hai phía, vì không thể fire khi compile.
- Hai decode độc lập (encode, rồi verify) là chủ ý. Muốn nhanh thì `Verify = false`.

**Metric dùng chung.** `Luma(r, g, b) = (77r + 150g + 29b) >> 8`, dùng ở mọi nơi. `TileSsim` trên luma, tile
64×64 không chồng, dải mép hẹp hơn 16 px gộp vào tile kề, `C1 = (0.01·255)²`, `C2 = (0.03·255)²`, mean, var, cov
tính trực tiếp trên tile; trả `(Min, P1, Mean, MinTileX, MinTileY)`. SSIM toàn khung chỉ in trong report, không
gate, vì nó là trung bình và mù đúng loại lỗi cục bộ đã làm v1 mờ (17.1).

**Renderer.** Asset RGBA; alpha tổng `= objAlpha × texelAlpha / 255`; normal `dst·(1−a) + src·a`, additive
`dst + src·a`; canvas RGB24 với màu nền là tham số (đen như osu!, xám cho verifier). Lấy mẫu bilinear: pixel đích
chỉ được vẽ khi tâm của nó rơi trong quad, giá trị lấy tại `(lx − 0.5, ly − 0.5)` với chỉ số texel láng giềng
clamp vào biên (khớp ClampToEdge); tint áp sau khi lấy mẫu; mipmap của osu! bị bỏ qua cho tới khi golden của
client thật cho thấy lệch. Mọi object được `CommandEvaluator.Prepare` một lần lúc dựng (flatten loop, sắp ổn định
theo thời gian). Asset nạp lười qua `IAssetStorage` với cache LRU trần 256 MiB byte đã decode. `Pass` và `Fail`
được vẽ theo cờ `passing`. Public `StoryboardRenderer` là wrapper mỏng dùng mapping widescreen chuẩn.

**Ngữ nghĩa evaluate command** (renderer, display model và baker dùng chung). Với mỗi property, trên danh sách đã
sắp ổn định theo `StartMs`: không có command thì giá trị mặc định; `t` trước command đầu thì `Start` của command
đầu; tập active là command có `StartMs ≤ t ≤ EndMs`, nhiều thì chọn command bắt đầu muộn nhất (ở ranh giới dùng
chung, command sau thắng), command tức thời trả `End`, còn lại trả `lerp(Start, End, ease(p))`; không có active
(khoảng trống hoặc sau command cuối) thì `End` của command kết thúc muộn nhất trong số đã kết thúc, hòa thì lấy
command bắt đầu muộn hơn. Đây là ngữ nghĩa giữ giá trị trong khoảng trống của lazer (Phụ lục A). Bản v1 trả `End`
của command kết thúc muộn nhất kể cả khi nó chưa bắt đầu, và lấy command đầu theo thứ tự danh sách: phải sửa khi
tái dùng (mục 18).

**Workload.** Tại mỗi mốc 100 ms: object sống nếu lifetime chứa t (bảo thủ, OQ-5); diện tích là bbox của quad
sau transform, tính trong storyboard units, chia `854 × 480`; object có `F` đang nội suy tính vào transient, còn
lại nếu alpha > 0 tính vào steady; alpha = 0 vẫn tính vào số sprite sống nhưng không vào tải. Bộ nhớ animation là
Σ `frames × w × h × 4`. Tổng asset bytes đếm mỗi asset khác nhau một lần. Số dòng `.osb` được ước lượng từ IR:
một dòng mỗi object, cộng một mỗi command và mỗi header loop; lõi không ghi văn bản để đếm. Thuật toán quét sự
kiện với active set, không duyệt mọi object ở mỗi mốc.

**Kết quả cuối.** `Task<CompileSummary>` mang `CompileStatistics`, `WorkloadReport` và `VerificationReport` (null
khi `Verify = false`). Báo cáo không đạt là kết quả hợp lệ, không phải exception (D2). Cancel qua `CancellationToken`
dừng decode và không để lại file tạm.

---

## 10. Vật thể di chuyển làm lộ nền

Lý do của thiết kế hai nấc nằm ở 4.8. Mục này là cơ chế.

### 10.1 M2.5a: layer nhân quả theo rect

Mục đích: nội dung được author (D14) là nhiều vật độc lập di chuyển trên nền; K=1 không bắt được. Theo D12,
không có bước định danh object: layer được *đề xuất* bằng motion và được *nhận* chỉ khi đo thấy rẻ hơn.
Theo D17, nấc này không dựng nền và không có alpha. Phụ thuộc M2 và EZ.

**Quan sát dẫn tới thiết kế.** Khi X rời một chỗ, vùng đó hiển thị sai so với nguồn, `ErrorMap` đánh dấu
bẩn và `PatchProvider` cắt nó từ frame hiện tại. Nghĩa là "phục hồi nền phần vừa lộ" closed loop đã làm
sẵn, đúng cả khi X đi nhanh, bị che hay đổi hình, mà không cần biết rect nào. Giá trị thật của việc theo
dõi X là biểu diễn *chính X* bằng một sprite di chuyển thay vì patch lại X ở mỗi vị trí mới.

Contract, internal và in-proc:

```text
LayerProposal          := { supportMask(t) mức block, Sim(2) track, footprint(t), zHint }
ILayerProposer         := frames (downscale, pass A) → LayerProposal[]
                          block flow → sequential RANSAC Sim(2), K ≤ 4 → lọc liên thông ≥ diện tích
                          tối thiểu → prior thời gian từ frame trước (tracking mô hình chuyển động,
                          không phải tracking object)
MovingRectProvider     : ICandidateProvider, rank 25, propose ở đầu đoạn theo gate 9.4
```

Candidate: sprite là ảnh cắt bounding rect của support tại frame chuẩn (frame trong đoạn có support lớn
nhất và ít bị che nhất theo sự kiện occlusion), pixel đặc, track `MX`, `MY`, `S`, `R` từ quỹ đạo qua
`TrajectoryFitter` và easing rời rạc (9.8), ghép qua baker (I5). Chỉ cần rect của frame trước và frame hiện tại để
dựng quỹ đạo; không lưu lịch sử rect, vì vùng lộ do `ErrorMap` tìm.

**Quy tắc z cho patch vá (điểm dễ sai nhất).** Trong `.osb` object khai báo sau nằm trên. Patch vá nền sinh
sau sprite X, nếu đặt mặc định sẽ nằm trên X; khi X quay lại, patch che X, vùng X bẩn và patch chồng patch.
Quy tắc: patch có rect *không giao* footprint của bất kỳ layer di chuyển nào trong khoảng sống của nó được
chèn ngay dưới layer thấp nhất (`Add(item, below)`, 7.3); patch giao footprint thì nằm trên như thường.

| Vấn đề | Cách xử lý ở M2.5a |
| --- | --- |
| Đếm layer không cần người | matching pursuit (4.7 tầng 1), K ≤ 4 |
| Di chuyển nhanh, rect cũ và mới không giao | không ảnh hưởng tới vùng lộ; chỉ cần flow bắt kịp (OQ-2) |
| Đổi kích thước, xoay | quỹ đạo Sim(2), phát `S` và `R` trên ảnh chuẩn; rect chỉ để chọn vùng cắt |
| Biến dạng | phần dư trong footprint lớn, gate từ chối, rơi về crossfade hoặc patch |
| Nền quanh X lọt vào rect | an toàn về chất lượng (verify bắt), lỗ về chi phí trên nền có texture; gate quyết định; đo bằng bucket `rect-halo` |
| X bị Y che lúc cắt ảnh chuẩn, flow bị Y làm nhiễu | thành lỗi đo được, closed loop vá hoặc gate từ chối |
| Patch vá dính mép X hoặc vệt mờ | nằm dưới X nên vô hại; lộ ra sau thì bị bắt và vá lại |
| z giữa các layer | đồ thị ràng buộc từ sự kiện occlusion cộng duyệt hết thứ tự (K ≤ 4); mâu thuẫn thì giảm K |
| X ra khỏi khung, X xuất hiện lại sau khi bị che hoàn toàn | đóng sprite; xuất hiện lại là layer mới, hash chuẩn hóa dùng lại asset nếu pixel trùng |
| Group transform của video | ghép qua baker theo single-writer (I5) |

Spike trước khi xây, pass/fail chốt trước, trên `authored`, `object_reveal_*`, `fish-short` và `minecraft`
`standard`:

| Spike | Câu hỏi | Pass |
| --- | --- | --- |
| A1 | Layer rect từ `ILayerProposer` classical có thắng EZ không | bytes ≤ 0,7× và verify 100% trên ít nhất 2 trong 3 nhóm |
| A2 | KLT trong support so với ECC trên pixel cho Sim(2) | residual screen-space dưới 0,5 px ở ≥ 90% frame trên `authored` |

Điều kiện nhận M2.5a: điều kiện nhận ở mục 12; `object_reveal_flat` cho đúng một asset X và không patch nào trong
footprint của X; `object_reveal_textured` hoặc được nhận với bytes thấp hơn M2, hoặc bị gate từ chối với
output trùng byte EZ; case generator "layered motion kèm occlusion" cho `occlusion_consistency` ≥ 0,9;
bucket `rect-halo` và `reveal-strip` được báo cáo cho mọi fixture.

### 10.2 M2.5b: mask, alpha và dựng nền nhìn trước (có gate)

Chỉ xây khi M2.5a cho bucket `rect-halo` hoặc `reveal-strip` ≥ 20% bytes trên `authored`. Nấc a trả một
patch hẹp cho mỗi dải nền lộ ra (ví dụ X rộng 200 px đi 5 px mỗi frame trong 60 frame: khoảng 60 patch,
mỗi cái khoảng 1,5 KB overhead PNG, và số sprite sống tăng dần); nấc b đưa phần đó về một asset nền.

Bước phân rã chạy mỗi đoạn, trước vòng encode:

```text
pass A, downscale  đề xuất layer, support mask và quỹ đạo như M2.5a
pass A', full-res  tích lũy pixel trong hệ quy chiếu riêng của từng layer
                   X: mẫu trong support của X; nền: mẫu ngoài mọi support foreground
                   ước lượng bền vững: median khi đủ bộ nhớ, không thì trung bình có loại ngoại lai
                   pixel chưa từng thấy: lấp bằng khuếch tán, confidence 0, không bao giờ được tin
                   alpha từ biên (ILayerBoundaryRefiner classical: guided filter, trapped-ball)
                   màu foreground F = (I − (1−α)·B)/α với B là nền đã dựng, kẹp khi α nhỏ
kiểm thử khô       như 9.4, trên composite các layer cộng nền
pass B, encode     đặt nền và layer vào display list ở đầu đoạn, closed loop vá phần dư
```

Dựng nền và dựng sprite X là cùng một phép toán: tích lũy trong hệ quy chiếu của một layer theo quỹ đạo và
mask của chính nó; panorama khi camera động là trường hợp riêng. Lấp lỗ là bắt buộc chứ không phải tùy chọn:
ở mép X alpha nhỏ hơn 1, nên lỗ trong suốt của nền sẽ lộ qua mép và vi phạm I14. Glow và flash dùng additive
layer T9 trong cùng bước: trong hệ đã bù chuyển động, `Base` là percentile thấp theo thời gian, `Light` là max
delta, `α(t)` đóng dạng, gate clamp ≤ 2% và SSIM. Chi phí thêm: một lần decode full-res mỗi đoạn có layer.

Spike A4 (additive trên glow thật) và A5 (chi phí GPU của sidecar, chỉ khi gate sidecar ở mục 12 mở) thuộc nấc này.
Điều kiện nhận M2.5b: điều kiện nhận ở mục 12; trên `object_reveal_textured` bucket `reveal-strip` gần 0 và verify 100%.

---

## 11. Đo lường và quy trình quyết định

### 11.1 Hiệu chuẩn preset

Các số preset ở 1.4 là giữ chỗ. Chúng được chốt bằng một simulator: một lần decode, 24 simulator chạy lockstep cho
PSNR {30, 32, 34, 36, 38, 40} nhân SSIM {0,80; 0,85; 0,88; 0,92}, mỗi cái giữ một canvas dựng lại riêng. Mỗi frame,
mỗi simulator chạy `ErrorMap` rồi `DirtyRects` (chỉ block vi phạm, không headroom), copy pixel nguồn của mỗi rect
vào canvas (hard swap), cộng dồn số rect, bytes PNG thật của từng rect và phần trăm diện tích refresh. Đây là cận
trên chi phí của mỗi mức sàn, vì pipeline thật chỉ có thể rẻ hơn hard swap. Ảnh dựng lại ở 25%, 50%, 75% cửa sổ
được dump để người dùng xem bằng mắt.

Người dùng chốt ba bộ số High, Medium, Low sau khi xem bảng và ảnh; tránh bộ nào có phần trăm refresh gần 100%
trên nội dung sạch. Nếu PSNR tối thiểu 34 dB bất khả thi trên nội dung nhiễu, preset chuyển sang lấy SSIM tile làm
gate chính với PSNR thấp hơn. Đây là điểm dừng chờ người duy nhất trước khi xây vòng kín (I8).

### 11.2 Fixture và dataset

Catalog fixture, profile (`native`, `quick`, `standard`, `full`), quy tắc chọn cửa sổ và lệnh sinh clip nằm duy
nhất ở `tests/fixtures/README.md`. Quy tắc bắt buộc: medium và hard không bao giờ chạy ở độ dài, độ phân giải và
fps gốc trừ khi người dùng yêu cầu rõ; chúng chạy trên clip dẫn xuất, và clip đó là *nguồn* cho mọi phép đo.

Fixture synthetic, sinh bằng script ffmpeg, mỗi cái kiểm một cơ chế:

| Fixture | Kiểm | Kỳ vọng |
| --- | --- | --- |
| `static_synthetic` | I-frame, không làm gì thừa | đúng một lượt patch ở frame 0; `Fps` lớn hơn nguồn cho output giống hệt |
| `dissolve_synthetic` | crossfade | số asset không quá 1/5 so với khi tắt crossfade |
| `fade_black_synthetic` | photometric | 0 asset mới trong đoạn fade |
| `tint_synthetic` | photometric `C` | sinh track `C` |
| `pan_synthetic` | track camera | asset bytes không quá 1/5 so với khi tắt tầng phân tích |
| `pan_foreground_synthetic` | gate hai tầng | gate nhận hoặc từ chối đúng theo lợi ích ròng, kiểm cả hai nhánh bằng kích thước foreground |
| `object_reveal_flat`, `object_reveal_textured` | layer rect, patch chèn dưới | xem 10.1 |

Tier `authored` (motion graphics, UI, 2D animation, D14) gồm ba nguồn có ground truth ở mức chương trình: generator
storyboard (`tools/OsbMpeg.DatasetGen`, 5.5), Crello Animation và Animation2Code (license ở README fixture). Nấc
`DS` dựng tier này cùng metric tầng hai ở 5.5.

### 11.3 Attribution

Sau mỗi nấc lớn, chia bytes của output theo bucket để biết tiền đang đi đâu (I9). Dữ liệu trung gian của encoder
(flow, mô hình, commit log) được serialize khi `Attribution` bật.

| Bucket | Cách đo |
| --- | --- |
| `local-coherent-motion` | bytes patch trong vùng mà sequential RANSAC pass 2 trên flow đã lưu tìm được mô hình K=2 giải thích ít nhất 70% block |
| `pan-reveal` | bytes patch trong dải mép ngược hướng camera, bề rộng bằng quãng dịch chuyển của dải đó |
| `perspective-residual` | bytes patch ở vùng inlier ở tâm nhưng residual tăng theo bán kính |
| `near-duplicate-assets` | bytes asset có aHash 8×8 trùng asset khác nhưng exact hash khác |
| `rect-halo` | bytes patch nằm trong footprint của một layer rect đã nhận, tức nền bị kéo theo trong rect |
| `reveal-strip` | bytes patch chèn dưới layer để vá nền vừa lộ |
| `layer-edge` | bytes patch trong dải biên dưới 8 px của layer có alpha (M2.5b) |
| `deformation` | patch còn lại có SSIM giữa hai lần refresh liên tiếp cùng vùng ít nhất 0,7, tức đổi có cấu trúc |
| `noise` | phần còn lại, bucket không cứu được |

### 11.4 Oracle bằng solver (`OR`, D18)

Trong project đo, trên cửa sổ ngắn của tier `authored` và clip `standard`: chế độ exhaustive sinh mọi candidate,
encode và kiểm pixel từng cái, rồi giải bài toán phủ có trọng số bằng CP-SAT:

```text
ô    = (block, frame) vi phạm;  x_c ∈ {0,1} cho mỗi candidate c đã đo là đạt sàn
min  Σ bytes_c · x_c
s.t. mỗi ô được phủ ít nhất một lần; candidate đã chọn không chồng nhau trong cùng ô
     AliveSprites(t) ≤ 500, SbLoad(t) ≤ 4.0  với mọi t
```

So nghiệm với lựa chọn của ladder ra oracle gap. Ràng buộc không chồng nhau bỏ sót tổ hợp chồng hợp lệ như
crossfade lồng transform, nên kết quả là cận dưới của gap và được báo cáo kèm ghi chú đó. Gap dưới 10% thì đóng
câu hỏi ladder; từ 10% thì đề xuất bộ chọn best-among-passing và mở giải ngân sách bằng solver (9.9).
`Google.OrTools` chỉ nằm trong project đo, không vào lõi cho tới khi nhánh đó được duyệt.

### 11.5 Quy trình quyết định

Bucket lớn nhất từ khoảng 25 đến 30% trở lên thì mở chu trình research, prototype, benchmark cho hướng tương ứng:
Panorama, CanonicalAssetReuse, PerspectiveStrips; `rect-halo` hoặc `reveal-strip` từ 20% mở M2.5b; `layer-edge` từ
20% mở sidecar. Bucket `noise` trội thì dừng, ship, và ghi giới hạn vào README. Không đặt tên nấc mới trước; nấc
sau M3.5 chỉ tồn tại sau khi attribution và benchmark của prototype thắng. Metric tầng hai của `DS` và oracle gap
của `OR` đổ vào cùng quy trình. Mọi số đo được ghi lại cùng phiên bản tài liệu này.

---

## 12. Nấc năng lực và điều kiện nhận

Pipeline được xây theo nấc. Mỗi nấc là một tập năng lực có điều kiện nhận đo trên chính pipeline mới: so với sàn
tuyệt đối, hoặc so với nấc liền trước. Không nấc nào so với code v1. Thứ tự và gate dưới đây là thiết kế; bước
triển khai cụ thể thuộc plan được sinh sau.

| Nấc | Năng lực | Phụ thuộc | Điều kiện nhận |
| --- | --- | --- | --- |
| M0 | Bộ đo: ngữ nghĩa evaluate đúng, renderer bilinear RGBA, `TileSsim`, validator chồng lấn, workload, verifier độc lập; thư viện, storage, API channel | không | golden giải tích khớp ±1 mỗi kênh; positive control (mỗi frame là một sprite crop chính xác) đạt 100% với PSNR ≥ 99; dịch một sprite 1 px thì có vi phạm đúng frame đó |
| M0.5 | Hiệu chuẩn preset (11.1) | M0 | người dùng chốt số |
| M1 | Vòng kín với Patch, Photometric, Crossfade; chế độ rời nhau | M0.5 | 100% frame đạt `Medium` qua verifier trên mọi fixture; kỳ vọng của bốn fixture synthetic đầu ở 11.2; test tương đương display model 1000+ thao tác xanh; test đồng thời xanh |
| DS | Dataset ground truth, tier `authored`, metric tầng hai (5.5, 11.2) | M1 | là instrument; sai lệch tier rẻ so với render thật là phát hiện |
| M2 | Display model có transform và alpha theo pixel; tầng phân tích đoạn; gate commit trước, vá sau; track camera | DS | `pan_synthetic` ≤ 1/5 bytes so với M1 trên cùng clip; `pan_foreground_synthetic` kiểm cả hai nhánh gate; đoạn bị từ chối trùng byte với M1; verify 100% |
| EZ | Easing rời rạc cho mọi track (9.8) | M2 | ≥ 30% command bytes ít hơn trên `authored`, 0 verify regression; trên case generator "từng họ easing", easing đúng ở ≥ 90% case không mập mờ; trên medium và hard trùng byte hoặc nhỏ hơn M2 |
| M2.5a | Layer nhân quả theo rect (10.1) | EZ | trên `authored`: bytes ≤ 0,5× và sprite ≤ 0,2× so với EZ, verify 100%; trên medium và hard bytes không tăng quá 2%; kỳ vọng `object_reveal_*` ở 10.1; trượt thì quay về DEFER |
| M2.5b | Mask, alpha, dựng nền nhìn trước, khử lem màu (10.2) | M2.5a và bucket `rect-halo` hoặc `reveal-strip` ≥ 20% trên `authored` | bytes ≤ 0,7× M2.5a trên fixture kích hoạt nó, verify 100%, không fixture nào tệ đi |
| M3 | Ngân sách workload, Animation có cap, nén PNG lossless (9.9) | M2.5a (ngân sách và animation có thể sớm hơn nếu số đòi) | bad_apple giữ sàn với animation trong ngân sách; 100% asset giải mã trùng từng byte; không fixture nào vượt ngân sách mặc định hoặc có violation report rõ ràng |
| OR | Oracle solver (11.4) | M3 | là instrument |
| M3.5 | Attribution và quy trình quyết định (11.3, 11.5) | M3, OR | không phải feature |
| Sidecar | `ILayerBoundaryRefiner` bằng model matting trong tiến trình Python (D13) | M2.5b, bucket `layer-edge` ≥ 20% hoặc verify fail tập trung ở biên layer trên 2D animation | contract chỉ nói bằng mask và alpha; model chọn tại thời điểm gate mở; không có sidecar thì output không đổi |

Hai tripwire áp cho mọi nấc: wall time encode tăng quá 3 lần so với nấc liền trước trên cùng clip `standard` thì
dừng và profile (OQ-1) trước khi tối ưu mò; asset bytes của một fixture thật tăng quá 3 lần so với nấc liền trước
thì báo người dùng kèm bảng, không coi là fail vì sàn cứng là lựa chọn có chủ đích (D2).

Concept chưa vào nấc nào: synthesis nhận biết loop (`LP`, khi dataset hoặc attribution thấy chu kỳ phổ biến),
lookahead của encoder, tier vàng chụp từ client thật, nhiều video dùng chung decode, nguồn frame tùy biến, memo
toàn kết quả, đóng gói NuGet.

---

## 13. Failure modes và rủi ro

### 13.1 Failure modes theo nội dung

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
| Nhiễu và grain | phá đồng thời motion coherence, dedupe và crossfade | không hướng nào cứu được | preset thấp hơn cho cách biểu diễn; asset vẫn lossless nên nhiễu trả bằng bytes |
| Occlusion | không | hệ quả của kiến trúc (4.3) | không |

Nhiễu là ranh giới vật lý của toàn bộ cách tiếp cận này. Dữ liệu cũ đã cho thấy: tuner bó tay trên
birdbrain, còn bad_apple ra hơn 10 nghìn sprite cho 5 giây. Nói rõ một lần trong tài liệu sản phẩm,
đừng để mỗi milestone lại "phát hiện" lại.

### 13.2 Rủi ro kỹ thuật và edge case

| Rủi ro | Mức | Giảm thiểu |
| --- | --- | --- |
| Nội dung nhiễu cần patch gần như full refresh dưới sàn cứng, output lớn | Cao | hiệu chuẩn preset (11.1) cho thấy đường cong trước; tripwire kích thước (12); preset thấp hơn cho cách biểu diễn, asset vẫn lossless |
| Hiệu năng: preview trên mọi frame bị ảnh hưởng, span search của crossfade, verify decode và render lần hai | Cao | preview chỉ trên region; tripwire hiệu năng (12); `Verify = false` khi cần; OQ-1 |
| Hành vi lazer lấy từ deepwiki (giữ giá trị trong khoảng trống, S nhân V, chaining) chưa kiểm trên client thật | Trung bình | golden giải tích; tier vàng client thật là concept (12) |
| Mép patch lệch nhỏ khi group có xoay hoặc scale khác 1 (bilinear ở mép từng sprite so với một texture liền) | Thấp | chấp nhận; verify có thể báo vi phạm sát sàn, ghi vào report, không nới sàn |
| Sprite bị che một phần tích tụ, overdraw tăng | Trung bình | quyền sở hữu block đóng sprite bị che kín; đỉnh có trong workload; ngân sách ở M3 |
| Group có chuyển động: baker lấy mẫu mỗi frame cho từng sprite sống, số command tăng theo số sprite sống nhân số frame | Trung bình | đo command count; fit quỹ đạo của group là concept |
| Item nằm dưới và sống lâu chặn channel object tới khi nó đóng, object ra dồn ở cuối | Trung bình | là hệ quả trực tiếp của ghi theo thứ tự file (9.10); bộ nhớ giữ lại chỉ là metadata; caller cần tín hiệu sớm thì đọc channel tiến độ và asset |
| Caller dùng channel bounded nhỏ mà không đọc, encoder chờ mãi | Thấp | do caller quyết; doc comment của `CompileOutputs` ghi rõ; cancel qua `CancellationToken` vẫn dừng được |
| Decoder OsuParsers không thread-safe | Thấp | chỉ `OsbMpeg.Osb` dùng decoder, qua `StoryboardDecoderGate`; lõi không dùng |
| Antivirus hoặc indexer khóa file mới trên Windows | Thấp | retry có backoff (8.2) |
| Asset bị xóa từ bên ngoài giữa lúc chạy | Thấp | verifier lộ lỗi ngay; ghi trong doc comment; không tự sửa |
| `FrameSource` có thể bỏ lại một reader mồ côi khi ffmpeg không mở pipe | Thấp | giữ cách sửa đã đo (17.6); bọc thành `VideoDecodeException` |
| Decode của ffmpeg lệch ±1 frame ở biên cửa sổ khi seek | Trung bình | test tính quyết định luôn dùng `From = 0`; verifier decode cùng cách với encoder |

Edge case phải có test: kích thước không chia hết cho 16 hoặc lẻ (1918×1078); video nhỏ hơn 64 px; video 4K (rect
cap 1024, không asset nào vượt 3840); video toàn đen (PSNR = 100, asset đơn sắc dedupe đúng theo kích thước); một
frame duy nhất; VFR (filter `fps` chuẩn hóa); `Fps` lớn hơn nguồn (output giống hệt `Fps = null`); `Fps` nhỏ hơn
nguồn (60 xuống 30, thời gian rơi trên lưới 33,33 ms); cancel giữa encode (`OperationCanceledException`, không còn
file tạm); đĩa đầy (`IOException` lan ra, không còn file tạm); một instance compile hai video khác nhau đồng thời.

---

## 14. Bảng quyết định

Mỗi dòng một hướng, kèm điều kiện đảo quyết định. Không hướng nào bị khóa chết bằng ý kiến.

| Hướng | Bằng chứng | Lợi ích kỳ vọng | Độ phức tạp | Hợp với storyboard | Quyết định | Điều gì đảo nó |
| --- | --- | --- | --- | --- | --- | --- |
| Closed loop và sàn tuyệt đối | vòng hở là nguyên nhân gốc của mờ (17.1) | hết mờ do cấu trúc | trung bình | hoàn hảo, ở mức IR | MUST (M1) | không |
| Crossfade (T3) | blend lerp đã verify khớp shader; dissolve rất phổ biến | giảm mạnh refresh rate vùng đổi chậm | thấp | native (`F`) | MUST (M1) | không |
| Photometric (T8) | fade và tint là command native, gần 0 bytes | fade gần như miễn phí | thấp | native (`F`, `C`) | MUST (M1) | không |
| Global motion K=1 (T4) | R1 nội suy liên tục; 79 nghìn sprite là bằng chứng thiếu nó | thắng lớn nhất trên pan và zoom | trung bình | native (`M`, `S`) | MUST (M2), spike flow trước | spike flow (OQ-2) cho thấy flow không dùng được trên corpus thì thu hẹp về nội dung synthetic và game |
| Sub-segment theo motion | scene tĩnh rồi pan rồi tĩnh là ca thường | mỗi segment một representation đúng | thấp | không đổi | MUST (M2) | không |
| Layered motion K > 1, nấc rect nhân quả (T5, M2.5a) | nội dung được author là nhiều vật độc lập (D14); closed loop đã tự vá nền vừa lộ (4.8) | lớn trên `authored`, chưa rõ trên phần còn lại | trung bình | tốt (K sprite xếp z, patch vá chèn dưới) | MUST (M2.5a, sau R2) với keep-gate: trên `authored` bytes ≤ 0,5× và sprite ≤ 0,2× so với M2, verify 100%; trên medium và hard bytes không tăng quá 2% | trượt gate thì quay về DEFER tới M3.5 |
| Mask, alpha, dựng nền nhìn trước, khử lem màu cho layer (M2.5b) | chỉ xóa được phần dải lộ và quầng nền mà nấc a để lại (4.8) | bằng tổng hai bucket `rect-halo` và `reveal-strip` | cao, thêm một pass full-res | tốt (T12) | DEFER tới sau M2.5a | `rect-halo` hoặc `reveal-strip` ≥ 20% bytes trên `authored` |
| Panorama (T6) | toán khả thi (4.5); strip giải được giới hạn texture | cận dưới bytes cho pan dài | trung bình | tốt | DEFER tới M3.5 | bucket `pan-reveal` từ 25 tới 30% |
| Tái dùng asset chuẩn hóa | hash đã có; chuẩn hóa là mở rộng key (I10) | lớn nếu bucket near-dup lớn | thấp tới trung bình | hoàn hảo | DEFER tới M3.5 | bucket `near-duplicate-assets` từ khoảng 20% |
| Perspective strips | similarity thiếu shear, sai hệ thống và đo được | vừa, tùy nội dung | trung bình | chấp nhận được | DEFER tới M3.5 | bucket `perspective-residual` từ 25% |
| Suy luận occlusion riêng | là hệ quả của z-order cộng closed loop (4.3) | 0, vì đã có sẵn | không | không | REJECT (như một feature) | tìm ra ca mà closed loop không tự xử lý được |
| Object detection và tracking ngữ nghĩa làm xương sống (SAM 2/3) | lệch granularity (4.6); lợi ích identity đã có đường rẻ hơn (4.4); SAM 2 tự ghi nhận mất track qua che khuất dài, lẫn object giống nhau, không tự liệt kê object | thấp so với giá | cao, GPU, PyTorch | kém | REJECT (D12) | không |
| Model học tinh chỉnh biên layer (`ILayerBoundaryRefiner`: SAM, MatAnyone) | biên là chỗ duy nhất model học hơn classical | nhỏ trên `authored`, vừa trên footage | trung bình, nằm ngoài assembly (sidecar Python) | tốt, chỉ đổi alpha ở dải biên | DEFER tới sau M3.5 (D13) | bucket "mép layer" ≥ 20% bytes, hoặc verify fail tập trung ở biên layer trên 2D animation |
| Depth estimation model | z-order suy được từ dữ liệu occlusion (5.2) | thấp | cao | gián tiếp | REJECT | layered motion bế tắc riêng vì không suy được thứ tự z |
| Learned optical flow | classical khớp đúng granularity rect (4.6) | chỉ khi classical fail | cao (ONNX, model, GPU) | không đổi | REJECT ở giai đoạn này | spike flow và cả DIS flow đều fail trên một bucket lớn |
| RDO Lagrangian, graph search hay solver toàn cục làm bộ chọn trong lúc encode | fidelity là ràng buộc cứng chứ không phải λ; bài học predict so với measure (17.5) | thấp | cao | không đổi | REJECT | đo được greedy ladder bỏ phí hơn 10 tới 15% trên corpus |
| DP đặt keyframe | RDP đủ theo mọi số đo hiện có | nhỏ | thấp | không đổi | DEFER | đo thấy RDP bỏ phí keyframe đáng kể |
| Fuzzy, pHash Hamming hay embedding cho near-dup, thay asset không verify pixel | O(N²) và rủi ro sàn; hai asset gần giống thay nhau giữa hai run kề sinh pop; canonical exact phủ phần lớn (4.4) | nhỏ so với canonical | cao | không đổi | REJECT | canonical reuse đã ship mà bucket near-dup vẫn lớn, và chỉ khi có verify SSIM so với sàn |
| Animation fallback (T7) | thrash có thật (bad_apple); R4 giới hạn memory | chặn nổ sprite count | thấp | native, đắt memory | MUST (M3), có cap | không |
| JPEG, WebP, AVIF lossy, palette xấp xỉ, dither, giảm độ phân giải asset | người dùng chốt asset luôn lossless (D19); đo cũ còn cho JPEG tệ hơn PNG: +6,5% trên anime, +62,6% trên bad_apple; palette không dither là một nguyên nhân mờ của v1 (17.1) | âm về chất lượng | thấp | được osu! hỗ trợ một phần | REJECT vĩnh viễn (D19) | không |
| Nén PNG lossless tốt hơn (mức nén, filter, RGB, grayscale, indexed khi ≤ 256 màu thật) | giảm bytes mà pixel giải mã trùng từng byte | nhỏ tới vừa, lớn trên nội dung màu phẳng | thấp | native | MUST (M3, 9.9) | không |
| Fit easing rời rạc trong TrajectoryFitter | lý thuyết vững, fit dạng đóng nên rẻ (5.3); nội dung ưu tiên được author bằng tween (D14) | lớn trên nội dung được author, gần 0 trên trajectory nhiễu | thấp | native (35 easing cố định, không tham số tự do) | MUST (`EZ`, ngay sau M2) | keep-gate: ≥ 30% command bytes trên `authored`, 0 verify regression; trượt thì giữ RDP linear |
| Cubic Bézier, spring, curve tham số tự do trong track | `.osb` chỉ có 35 easing cố định; Bézier chỉ xấp xỉ được bằng chuỗi segment | âm (không phát được) | không đổi | không có | REJECT | osu! thêm easing tham số |
| Dataset ground truth và metric tầng hai | generator và renderer đã có; Crello Animation và Animation2Code có ground truth sẵn; mở khóa câu hỏi "chọn sai ở đâu" | là hạ tầng đánh giá | trung bình | là instrument | MUST (`DS`, ngay sau M1) | không cần gate; sai lệch tier rẻ so với tier vàng tự nó là phát hiện |
| Solver CP-SAT/ILP làm oracle offline (`OR`) | là cách duy nhất có nghiệm tối ưu để so; chạy trên candidate đã đo nên không dự đoán (4.9) | biết chắc greedy bỏ phí bao nhiêu | trung bình, nằm ngoài assembly | không đổi | MUST (`OR`, sau M3) | không cần gate, là instrument |
| Solver cho ngân sách workload (9.9) | ràng buộc theo lát thời gian liên kết nhau, greedy có thể vừa vi phạm vừa tốn (4.9) | vừa, chỉ trên đoạn vượt | trung bình | không đổi | INVESTIGATE (M3) | greedy ở 9.9 để lại vi phạm, hoặc R3 thấy gap ≥ 10% ở đoạn vượt |
| Bộ chọn best-among-passing | bậc chi phí cách nhau bậc độ lớn nên nghi ngờ gap nhỏ; chưa đo | nhỏ tới vừa | thấp | không đổi | DEFER tới `OR` | gap từ 10% trên dataset và fixture |
| Synthesis nhận biết loop | `.osb` bytes là trục lợi duy nhất; LoopExtractor hậu kỳ đã có | nhỏ trừ khi nội dung có chu kỳ | trung bình | native (`L`) | DEFER (`LP`, mục 12) | `DS` hoặc attribution thấy chu kỳ phổ biến |
| Prune nhiều độ phân giải trong Evaluate | chỉ loại chứ không nhận nên an toàn với I4 | chỉ về hiệu năng | thấp | không đổi | DEFER | profile cho thấy evaluate là nút cổ chai |
| Score truy hồi tổng hợp | chưa có dữ liệu về tương quan giữa metric và "truy hồi đúng" | chưa rõ | chưa rõ | không đổi | REJECT cho tới khi có dữ liệu | benchmark `DS` cho thấy các metric rời tương quan mạnh và cần một số duy nhất |
| Mesh hoặc subdivision tổng quát | nổ sprite count; thiếu shear làm lưới thô sai hệ thống | chưa rõ | cao | kém (fill-rate) | REJECT | chưa thấy đường đảo |
| `[Variables]` để nén `.osb` | R9, 3 giây thành 45 giây | âm | không đổi | không đổi | REJECT vĩnh viễn | không |
| Value-chaining shorthand | lazer chỉ đọc cặp giá trị đầu (R9) | âm trên lazer | không đổi | sai trên chuẩn verify | REJECT vĩnh viễn (D1) | không |
| Nền động hoặc vùng lớn bằng `Animation` | R4: N frame full-res resident cùng lúc, 1080p × 300 frame xấp xỉ 2,4 GB | âm | không đổi | nổ memory | REJECT; `Animation` chỉ cho vùng nhỏ với cap resident bytes (M3) | không |
| Cost function dự đoán với trọng số tay (asset, command, overdraw) | predict thua measure 2,6 đến 5,4 lần (17.5); ba trục workload là vách đá chứ không phải dốc (1.3) | âm | không đổi | không đổi | REJECT; chi phí đo thật, ràng buộc cứng | không |
| Freeze gate so frame t+1 với frame t | chính là vòng hở gây mờ của v1 (17.1) | âm | không đổi | không đổi | REJECT; chỉ so với asset đang hiển thị trên composite | không |

---

## 15. Câu hỏi mở và giả định cần xác minh

### 15.1 Câu hỏi mở

| ID | Câu hỏi | Vì sao chưa quyết | Spike quyết định | Phạm vi an toàn |
| --- | --- | --- | --- | --- |
| OQ-1 | Chiến lược re-composite nào của `ReconstructionState` nhanh nhất | vấn đề hiệu năng, không phải tính đúng | benchmark 2 chiến lược trên fish 5 giây | chỉ đổi bên trong class, giữ test đối chiếu render đầy đủ |
| OQ-2 | Block matching có đủ tốt trên footage thật không, và trên motion graphics ít texture thì KLT trong support hay ECC trên pixel | chất lượng flow phụ thuộc nội dung; Zhang 2023 ghi flow neural lẫn SIFT và KLT kém trên motion graphics | spike flow trước M2 (mục 12) | code spike ném đi được, chỉ số liệu là deliverable |
| OQ-3 | Con số preset cuối cùng | phụ thuộc nội dung | M0.5 | chờ người dùng xem ảnh, không tự chốt |
| OQ-5 | Ngữ nghĩa alive chính xác cho `WorkloadAnalyzer` (alpha lớn hơn 0 lần đầu so với span command) | ảnh hưởng độ chính xác report, không ảnh hưởng output | hỏi cụ thể qua deepwiki khi cần | dùng bản bảo thủ (span command) tới lúc đó |
| OQ-6 | osu! stable và lazer có đọc đúng PNG grayscale và PNG indexed không | nén lossless ở 9.9 dùng các biến thể này; chưa verify trên client | spike thủ công trước M3 | khoảng 30 phút; fail thì chỉ giữ biến thể RGB và RGBA, không tìm cách lách |

### 15.2 Giả định cần xác minh trước khi tin số đo

Các giả định dưới đây là nền của renderer, verifier và hiệu năng. Mỗi cái có một phép thử nhỏ, chạy trước hoặc
trong M0, và kết quả được ghi lại cùng tài liệu này.

- **Ngữ nghĩa hiển thị của lazer:** giữ giá trị trong khoảng trống giữa hai command cùng loại, `S` nhân `V`, blend
  straight alpha, tint nhân. Sai thì sửa renderer và evaluator theo lazer.
- **Không lộ đường nối:** patch ghép sát nhau không lộ đường nối khi lazer scale theo độ phân giải màn hình, thường
  là tỷ lệ không nguyên. Lộ thì cần overlap hoặc bleed ở mép patch, một thay đổi lớn cho gom vùng bẩn và asset.
- **Không nháy:** chuỗi patch ở 60 fps không nháy đen ở ranh giới.
- **Chạy mượt:** output kiểu vòng kín chạy mượt trong lazer ở số sprite sống và tải mà ngân sách 18 cho phép.
- **OsuParsers 18.2:** API khớp Phụ lục A, xác nhận bằng code compile được.
- **ffmpeg:** ngữ nghĩa resample fps đúng 9.2 với giá trị `round` đã chọn.
- **Hệ thống file:** ghi nguyên tử nhiều tiến trình trên Windows đúng 8.2.

---

## 16. Prior art đã đối chiếu

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
| Layer theo motion, không theo ngữ nghĩa | Wang và Adelson, *Representing Moving Images with Layers*, IEEE TIP 1994; object-centric learning (SAVi++, NeurIPS 2022) còn yếu trên video thật khi không có điều kiện | là khung của mục 4.1 |
| Keyframe và easing | Roberts, PhD 2018 (optimal keyframe selection giống animator); Nowicki, Cowan, Mann, GI 2019; Sawicki 2020 (Bézier tổng quát hóa easing); PELT (Killick 2012); PVLDB 2018 trajectory simplification study. Không có paper riêng cho nhận dạng easing từ quỹ đạo | `EZ` là contribution nhỏ nhưng thật, nếu có ground truth |
| De-rendering animation sang code | Ji et al., *Animation2Code*, NeurIPS 2026: 1.069 web animation, VLM đạt appearance ~0,8 nhưng temporal similarity ~0,3 kể cả sau fine-tune; Liang et al., *VAnim*, ICML 2026 | bằng chứng bài toán chưa bị mô hình lớn giải; dataset dùng cho tier `authored` (kiểm license trước) |

---

## 17. Bài học từ v1

Phần này giữ những gì đã trả giá bằng đo đạc, để không ai phải suy diễn lại hay đề xuất lại thứ đã bị bác bỏ. Các con số là bằng chứng cho quyết định, không phải mốc so sánh.

### 17.1 Vì sao v1 mờ và nặng

**Mờ.**

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

**Nặng.**
Chỉ có đúng một primitive. `TileEncodeLoop.Emit` phát ra mỗi run thành một sprite kèm một lệnh `S`
tĩnh để đặt scale. Không có lệnh động nào (`M`, `F`, `C`, `R`) được dùng để mô tả nội dung. Camera
pan làm mọi tile đổi mỗi frame, và mỗi tile đổi lại phải re-emit ở độ phân giải nguồn. Con số thực
đo được trên Minecraft full clip 60fps: 79.612 sprites, 79.223 assets, 735,01 MB assets cộng 9,31 MB
`.osb`, tổng 744,32 MB, thời gian compile 542 phút.

Animation không dedupe được (format cần chuỗi file đánh số riêng cho mỗi object) và có thể lên tới
300 frame PNG full-res cho một object. Mọi frame đó nằm trong RAM cùng lúc khi osu! load.

Yêu cầu 60fps qua frame duplication nhân đôi mật độ biên run.

### 17.2 Motion và region compensation trên lưới cố định (đã bác bỏ)

`GlobalMotionEstimator`, `RegionSegmenter`, `RegionTracker`, `TransformEstimator`, `TrajectoryFitter`,
`OcclusionAnalyzer` từng bị bác bỏ vì lý do cấu trúc chứ không phải chưa tinh chỉnh. `.osb` không có
predictive hay residual coding, mỗi Sprite là một texture đầy đủ và tự chứa. Một motion vector chỉ
dùng để *dự đoán* tile lưới cố định tại (x,y) từ frame trước không mua được gì: bytes của asset vẫn
đến từ việc crop vùng (x,y) cố định của frame *hiện tại*, mà vùng đó đổi mỗi frame dưới một cú pan bất
kể vector nói gì. Muốn asset trùng byte qua các frame khi có chuyển động thật thì phải dịch lưới *lấy
mẫu* theo nội dung, nhưng khi đó lưới *xuất* cũng phải dịch đúng như vậy để dựng lại cho đúng, và điều
đó mở ra khoảng trống ở một mép cùng phần thừa ở mép kia.

Điểm khác biệt của hướng mới ở mục 4: asset là sprite bền di chuyển bằng lệnh `M` và `S`, không
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

### 17.3 AssetTrimmer (đã bác bỏ)

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

Bài học đó được tôn trọng trong crossfade ở 9.6: sprite cũ được giữ sống cho tới khi
sprite mới opaque hoàn toàn.

### 17.4 Tầng RDO (đã bác bỏ)

Trước khi xây bất kỳ framework nào, một phép đo được đặt làm điều kiện: quét
`--min-animation-uniqueness` (ngưỡng heuristic chọn Sprite so với Animation sẵn có) xem một lần tinh
chỉnh mặc định đơn giản có bắt được các ca mà framework theo chi phí lẽ ra cần cho không. Đã quét hai
lần, trước và sau một sửa lỗi không liên quan ở `QuadtreeMerger`, và output trùng byte cả hai lần trên
fixture test, trước cũng như sau. Cả hai lần thắng thật về nén trong phiên đó đều truy về việc sửa bug
cơ học ở *phía trên* tầng chọn candidate (một lỗi làm tròn và một lỗi tiêu chí merge), không phải do
bản thân heuristic chọn sai. Không xây; tiền đề (một ca mà ngưỡng uniqueness về cấu trúc không thể
biểu đạt lựa chọn đúng) vẫn chưa được chứng minh trên mọi fixture đã thử.

### 17.5 Phân hoạch tile theo heatmap (spike, không nhận)

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
in-memory để có chi phí PNG thật, so với output có tuning theo scene của chính v1 khi đó:

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

### 17.6 Bài học khác đã trả giá

- **Kích thước phải nằm trong key asset.** Hash chỉ trên byte pixel làm hai crop đơn sắc khác kích thước (10×20 và
  20×10 cùng màu đen) trùng key và dùng nhầm file. Key ở 8.3 gồm kích thước và số frame.
- **Cache toàn cục, không định danh theo lần chạy.** v1 từng đặt thư mục asset theo thứ tự xuất hiện của video
  trong một document, nên cùng một video ở hai project rơi vào hai thư mục. Một store phẳng, content-addressed,
  dùng chung cho mọi lần gọi là đúng; tên chỉ phụ thuộc pixel (D9).
- **Mọi tham số ảnh hưởng pixel phải nằm trong key.** Một tham số lượng tử hóa cố định lúc dựng store từng làm hai
  cảnh dùng lại nhầm asset của nhau. Pipeline mới không còn tham số như vậy vì asset luôn lossless (D19), nhưng quy
  tắc giữ nguyên: nếu một ngày có, nó vào key.
- **Đổi lưới làm mất toàn bộ cache.** Hash phủ cả buffer, nên đổi kích thước tile cho 0 cache hit kể cả trên cùng
  footage. Pipeline mới không có lưới cố định cho asset, nhưng mọi cơ chế cắt phải ổn định giữa các lần compile lại
  để cache có ý nghĩa.
- **`FrameSource` treo khi ffmpeg không mở pipe.** Reader của named pipe chờ vô hạn nếu ffmpeg thoát trước khi mở
  phía output, ví dụ vì file không tồn tại. Ba cách sửa hụt đã được đo là sai (hủy theo task của ffmpeg, giới hạn
  lần đọc đầu, giết tiến trình đã tự thoát). Cách đúng: chỉ giới hạn phần chờ channel của chính mình
  (`ChannelReader.WaitToReadAsync(token)`) cho item đầu tiên, `StartupTimeout` 60 giây; input hỏng thật vẫn fail
  trong khoảng 1,3 giây.
- **Fan-out theo channel dễ deadlock.** Một bản decode dùng chung ghi vô điều kiện tới mọi consumer; consumer không
  được đọc làm đầy buffer và chặn producer, deadlock ở concurrency từ 4, và ở concurrency 1 thì giao thiếu frame
  nhưng báo thành công. Decode một lần vào bộ nhớ rồi replay là cách an toàn; mọi thiết kế dùng chung decode sau
  này phải có test cả hai concurrency.
- **Ngưỡng cut 0,7.** Ngưỡng đoán 0,8 bỏ sót một cut thật đo ở 0,721, trong khi một frame chuyển động mạnh bình
  thường đo 0,662; 0,7 nằm giữa hai số đo thật. Gom các frame vượt ngưỡng gần nhau (dung sai 300 ms) thành một ứng
  viên, chỉ từ chối ứng viên tạo scene dài gần 0 (dưới 1000 ms).
- **Merge một frame không phải nội dung chung.** Gộp các tile chỉ vì chúng đóng cùng lúc, không kiểm nội dung,
  từng gộp những tile 100% khác nhau ở vùng chuyển động nhanh. Bài học cho mọi cơ chế gom: tín hiệu thời gian
  không thay được kiểm pixel (I4).
- **Dung sai thời gian phải tính theo frame phân số.** Ở 30 fps, biên run lượng tử về ms xen kẽ 33 và 34 ms, dung
  sai 0,5 ms phân loại sai khoảng 29% run một frame.
- **Sprite count và bytes là hai trục khác nhau.** Một thay đổi giảm bytes 28,9% nhưng giảm sprite 94% trên cùng
  nội dung. Đây là lý do workload là ràng buộc riêng, không gộp vào bytes (1.3).

---

## 18. Thành phần v1 tái sử dụng

Code v1 là kho thành phần. Bảng dưới nói cái gì được lấy, sửa gì khi lấy, và cái gì bị bỏ. Không có gì ở đây là
mốc so sánh.

| Thành phần v1 | Dùng lại thế nào | Sửa khi lấy |
| --- | --- | --- |
| IR (`SbDocument`, `SbObject`, `SbCommand`), `LoopFlattener` | IR nội bộ của lõi | chuyển thành `internal`; thêm `IntrinsicScale` cho verifier |
| `Ir/Passes` (`MergeAdjacentCommands`, `DropNoOpCommands`, `LoopExtractor`) | passes theo object ở 9.10 | chạy theo từng object thay vì cả document |
| `CommandEvaluator`, `EasingTable` (đủ easing 0 tới 34) | evaluate cho renderer, display model, baker | sửa ngữ nghĩa khoảng trống và thứ tự (9.11); thêm `Prepare` |
| `Compositor` (blend đã verify khớp osu-framework) | renderer và display model | thêm RGBA và bilinear |
| `SoftwareStoryboardRenderer` | renderer của verifier | đọc asset qua `IAssetStorage` với LRU có trần; bỏ cache vô hạn |
| `CanvasMapping` (khớp phép toán của `DrawableStoryboard`, auto-cover) | mapping pixel sang storyboard | không |
| `GroupTransformBaker` | ghép group transform theo một người ghi (9.3) | thêm nhánh ghép track alpha, tint và transform nội dung |
| `FrameSource`, `MediaProbe` | decode và probe | cấu hình ffmpeg theo từng lần gọi; chọn `round` của filter `fps` theo ngữ nghĩa 9.2; giữ cách sửa treo (17.6) |
| `ContentHasher` (XXH3-128) | không dùng trực tiếp; key asset tính theo 8.3 | key mới gồm tag, loại, kích thước, số frame |
| `OsbWriter`, `OsbReader` | chuyển sang `OsbMpeg.Osb` | ghi qua `TextWriter`; culture bất biến |
| `Metrics.Psnr` | metric dùng chung | thêm `Luma` số nguyên, `TileSsim`, `BlockMse` |
| `AnimationDetector` | chỉ tham khảo logic khi viết pass thrash (9.9) | viết lại gọn |
| Ngưỡng cut và thuật toán gom cut của `SceneBounds` | quy tắc cut trong vòng frame (9.5) | không còn pass riêng |

Bỏ hẳn: `ParameterTuner` và toàn bộ tuning, `TileEncodeLoop`, `TileRunTracker`, `TileGrid`, `QuadtreeMerger`,
`EncodePipeline`, `NaiveBaseline`, `OsbValidator`, `AssetStore` (thay bằng `AssetIndex` và `IAssetStorage`),
`ScenePrePass`, `VideoCompiler` tĩnh, `VideoSourcePlanner`, `VideoSourceKey`, `FrameWriter`, `CanvasVideoFrame`,
parser `.osbv`, project CLI và các subcommand ẩn.

---

## Phụ lục A: sự thật bên ngoài và trạng thái xác minh

**osu!lazer** [deepwiki, `ppy/osu`, cần golden client thật để khẳng định]:

- `LegacyStoryboardDecoder` chỉ đọc cặp giá trị đầu của mỗi command, bỏ qua các giá trị chaining.
- `DrawableStoryboardSprite.DrawScale = base.DrawScale (có flip) × VectorScale`, tức S nhân V.
- Trong khoảng trống giữa hai command cùng loại, property giữ `EndValue` của command trước.

**OsuParsers** [deepwiki, `mrflashstudio/OsuParsers`, **cần xác nhận bằng code compile được trên 1.7.2**]:

- `StoryboardSprite(Origins, string filePath, float x, float y)` và
  `StoryboardAnimation(Origins, string, float, float, int frameCount, double frameDelay, LoopType)` đều
  có `Commands: CommandGroup`.
- `CommandGroup { List<Command> Commands; List<TriggerCommand> Triggers; List<LoopCommand> Loops }`.
- `Command`: `StartTime` và `EndTime` là `int`; có `StartFloat`/`EndFloat`,
  `StartVector`/`EndVector` (`System.Numerics.Vector2`), `StartColour`/`EndColour`
  (`System.Drawing.Color`). Constructor:
  `(CommandType, Easing, int, int, float, float)`, `(CommandType, Easing, int, int, Vector2, Vector2)`,
  `(Easing, int, int, Color, Color)`, `(CommandType, Easing, int, int)`.
- `LoopCommand(int startTime, int loopCount)`; `TriggerCommand(string, int, int, int groupNumber)`.
- `Storyboard` có 5 layer hình, `SamplesLayer`, `Variables`, `GetLayer(StoryboardLayer)`, và
  `Save(string path)`. Encoder công khai hay không thì chưa rõ.
- `StoryboardDecoder.Decode(string path | IEnumerable<string> | Stream)`. Không thread-safe (state
  static).

**FFMpegCore 5.x** [context7]: `FFOptions` được truyền theo từng lần gọi qua
`ProcessAsynchronously(throwOnError, ffOptions)` và qua mọi method của `FFProbe`. Không cần
`GlobalFFOptions`.
