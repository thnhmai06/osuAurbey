# Review concept Aurbey v2 (`aurbey.md`): khả thi, dự đoán chất lượng/hiệu năng, khoảng trống

Tài liệu này review `aurbey.md` (pipeline object-centric SAM 2 → delta → nhóm/nền → CAS → easing)
đối chiếu với (a) những gì osu! storyboard thật sự biểu diễn được và các số đo đã có trong
`docs/direction.md`, (b) literature hiện có. Mục 7 trả lời riêng ba câu hỏi trong email gửi giảng
viên về tính mới của đề tài.

Ký hiệu: **OK** = đúng và có cơ sở; **SAI** = mâu thuẫn với osu! hoặc với số đo đã có trong repo;
**HỞ** = chưa giải quyết, concept nêu tên nhưng chưa có cơ chế; **PA** = đã có prior art giải quyết.

---

## 0. Kết luận ngắn

1. **Concept không phải một hướng mới về decomposition.** Đặt đúng tên thì nó là MPEG-4 Part 2
   *sprite coding / object-based coding* (1998–2010) cộng dòng *sprite decomposition* 2021–2024
   (Layered Neural Atlases, Deformable Sprites, Sprite-from-Sprite, Fast Sprite Decomposition) cộng
   *motion vectorization* (Zhang et al. 2023), retarget sang DSL `.osb`. Mỗi khối trong 5 giai đoạn
   đều có paper làm rồi, ở mức tốt hơn mô tả trong concept.
2. **Phần thật sự còn trống** là tầng *compile*: chọn representation trong một DSL cố định, không
   khả vi, bị ràng buộc cứng (không shear, `C` chỉ làm tối, Animation preload toàn bộ, track cùng
   property không được chồng, chi phí fill-rate), dưới một sàn fidelity *được verify bằng render*, với
   hàm chi phí *đo* chứ không *dự đoán*. Đây là bài toán rate–distortion trên không gian chương
   trình, và là chỗ concept hiện viết sơ sài nhất (Stage 3 cost function với ba trọng số tay).
3. **Khả thi về kỹ thuật**, nhưng không ở dạng đang viết. Năm lỗi phải sửa trước khi làm gì tiếp
   (mục 3): freeze gate đang là vòng hở (đúng nguyên nhân gốc gây mờ của bản hiện tại); nền động
   đẩy vào `Animation` là bom memory; cubic Bézier và curve tùy ý không phát được ra `.osb`; swap
   asset theo pHash Hamming ≤ 2 không có verify pixel sẽ gây flicker; cost function dự đoán đã từng
   thua đo thật 2,6–5,4 lần trong chính repo này.
4. **Dự đoán chất lượng**: thắng lớn trên motion graphics, UI, 2D animation sạch (giảm sprite 1–2
   bậc độ lớn); thắng vừa trên game render có camera (phần lớn lợi ích đến từ global motion K=1 và
   panorama, chứ không từ object tracking); gần như không thắng trên footage thật có nhiễu, vì nhiễu
   là ranh giới vật lý của mọi cách tiếp cận sprite (direction §7). Trên bad_apple không khác hôm nay.
5. **Dự đoán hiệu năng**: bắt buộc GPU. SAM 2 chạy từng object độc lập; ở 1080p, 10 object, 3 phút
   60fps thì riêng SAM 2 đã cỡ một giờ trên A100, chưa tính matting, flow, inpainting. Không chạy
   được trong tiến trình .NET hiện tại; cần sidecar Python hoặc ONNX Runtime kèm model vài trăm MB.
6. **Cho khóa luận**: còn chỗ, nhưng phải đổi câu hỏi nghiên cứu từ "phân rã video thành thực thể"
   (đã có) sang "biên dịch video thành chương trình animation có kiểm chứng, với bộ chọn
   representation tối ưu chi phí dưới sàn fidelity" và kèm benchmark có ground truth ở mức chương
   trình. Animation2Code (NeurIPS 2026) vừa cho thấy VLM de-render animation với temporal similarity
   chỉ ~0,3, nghĩa là bài toán chưa được giải bằng mô hình lớn.

---

## 1. Concept thực chất là gì: bảng ánh xạ sang prior art

| Giai đoạn trong `aurbey.md` | Prior art gần nhất | Khác biệt còn lại |
| --- | --- | --- |
| Toàn bộ pipeline: nền tĩnh + object foreground + transform/frame | **MPEG-4 Part 2 sprite coding** (static sprite + GME 8 tham số + VOP foreground). Sprite mode hiệu quả gấp vài lần normal mode khi foreground < 10–15% diện tích; chưa bao giờ có bản thương mại vì bước segmentation quá khó (Glantz 2010, Krutz 2008, Waseda R03) | Target là DSL render được bởi osu!, không phải bitstream; segmentation nay có SAM 2/3 |
| Object = canonical image + affine/frame + z-index | **Zhang et al. 2023, SVG Motion**: color clustering LAB + trapped-ball, differentiable compositing fit affine (scale, translate, rotate, skew), greedy chọn mapping không xung đột, canonical image cập nhật khi object lộ ra lớn hơn; object bị che hoàn toàn thì *mất ID* khi xuất hiện lại | Concept dùng Sim(2) thay affine (đúng với osu!, vì không có shear); dùng SAM 2 thay color clustering nên chạy được trên nội dung có texture |
| Sprite = static texture + tham số animation, khởi tạo bằng VOS | **Suzuki et al. 2024, Fast Sprite Decomposition** (ECCV): static-texture assumption, texture prior chống artifact, init bằng VOS + bbox một frame, dataset Crello Animation | Giống gần như hoàn toàn Stage 1–2; concept chưa có texture prior |
| Decomposition cartoon, tween và sprite tái dùng | **Sprite-from-Sprite** (Zhang, Wong, Liu, SIGGRAPH Asia 2022): Pixel MLP, sprite dễ giải trước dẫn đường sprite khó | Người dùng phải chọn số sprite |
| Một texture + biến dạng theo frame | **Deformable Sprites** (Ye 2022), **Layered Neural Atlases** (Kasten 2021) | osu! không có primitive biến dạng, nên phần này chỉ dùng được làm *detector* để fall-through, không dùng làm representation |
| Hiệu ứng gắn với object (bóng, phản xạ, glow) | **Omnimatte** (Lu 2021, nền homography tĩnh), OmnimatteRF (3D background), **Generative Omnimatte** (CVPR 2025, video diffusion) | Concept chỉ xử lý glow additive, chưa xử lý bóng đổ (shadow là *nhân tối*, có thể map sang `C` trên một sprite phụ, chưa ai thử cho osu!) |
| Matting từ mask SAM | **MatAnyone** (CVPR 2025): chỉ cần mask frame đầu, memory propagation, ổn định vùng lõi, giữ chi tiết biên | Thay thế hoàn toàn "trimap tự sinh bằng dilation/erosion" |
| Khử lem màu nền (decontamination) | **Germer et al. 2020, Fast Multi-Level Foreground Estimation**; **FBA Matting** (Forte & Pitié 2020) ước lượng F, B, α đồng thời | Công thức F = (I − (1−α)B)/α trong concept là closed-form ngây thơ, nổ ở α → 0 và cần B đã inpaint |
| Nền amodal từ lookahead | **Granados 2012** background inpainting camera động; **ProPainter** (ICCV 2023) propagation + transformer; **TABE** (amodal qua SAM 2 + diffusion outpainting) | ProPainter thất bại khi vùng che chưa từng lộ; TABE là generative nên vi phạm "không ảo giác" |
| Base + α(t)·Light | **Factored Time-Lapse Video** (Sunkavalli 2007), **Intrinsic Video** (Ye 2014); về toán là NMF hạng 1 với anchor prior | Hạng 1 chỉ đúng khi vệt sáng *không di chuyển*; vệt sáng trôi phải track trước rồi mới factor |
| Grouping theo motion coherence, không theo ngữ nghĩa | **Wang & Adelson 1994** layered representation; chính `direction.md` §5.1 (block flow → sequential RANSAC, K ≤ 4) | Concept và direction.md đồng ý ở điểm này; vậy SAM 2 đóng vai gì? (xem mục 6) |
| Easing / curve fitting, piecewise, chọn mô hình theo error + λ·complexity | Keyframe reduction (Roberts 2018, optimal keyframe selection), changepoint **PELT** (Killick 2012), MDL; Sawicki 2020 tổng quát hóa easing bằng Bézier; tool thực dụng (MotionCode, fit cubic-bezier và spring) | Không có paper riêng cho "nhận dạng easing từ quỹ đạo"; đây là chỗ nhỏ nhưng còn trống thật |
| CAS, pHash, multi-index hashing | pHash DCT, MIH (Norouzi 2012) | Không mới, và `direction.md` 5.4 đã chỉ ra near-dup phải verify bằng SSIM so với sàn |

Nhận xét chung: concept liệt kê đúng các khối, nhưng mỗi khối đều mô tả *phiên bản đơn giản hơn*
phiên bản đã publish. Giá trị của concept không nằm ở các khối.

---

## 2. Khả thi theo từng giai đoạn

### 2.1 Giai đoạn 1: SAM 2, matting, decontamination

* **OK**: SAM 2 cho mask tốt hơn hẳn block matching trên nội dung có texture; memory bank giữ ID
  qua che khuất *ngắn*.
* **SAI**: "SAM 2 gán nhãn từng thực thể" và "bảo toàn danh tính kể cả Full Occlusion". SAM 2 là
  mô hình *promptable*: không tự liệt kê object, cần prompt (click, box, mask) hoặc grid prompt rồi
  lọc. Limitations của chính paper: fail qua shot change, lẫn object trong cảnh đông, mất track sau
  che khuất dài hoặc video dài, kém với vật mỏng chuyển động nhanh, lẫn object giống nhau; mỗi object
  xử lý *độc lập*, không trao đổi thông tin. Zhang 2023 cũng ghi: object bị che hoàn toàn khi xuất
  hiện lại nhận ID mới. Khắc phục có sẵn: **SAM 3** (11/2025) có detector + tracker, prompt bằng
  noun phrase hoặc exemplar, re-prompt định kỳ bằng detection độ tin cậy cao; **SAM 3.1 Object
  Multiplex** (03/2026) track nhiều object chung memory. Re-identification xuyên che khuất dài vẫn
  phải làm bằng canonical-hash của asset (direction 5.4) chứ không trông vào tracker.
* **SAI**: "SAM 2 theo dõi vệt sáng, đốm phản xạ như sub-entity". SAM 2 huấn luyện trên object
  (SA-V); vệt sáng là *effect*, là địa hạt của Omnimatte. Không có bằng chứng SAM 2 track được
  specular highlight ổn định.
* **HỞ**: scene cut. SAM 2 fail qua shot change, nên scene detection phải chạy *trước* và reset
  tracker, không chỉ dùng để chặn Animation vắt qua cut như concept viết.
* **HỞ**: alpha liên tục `A_k(t)` *theo từng frame* mâu thuẫn với "Canonical Patch 0 byte". Một
  sprite chỉ có một texture, tức một alpha. Mép mềm thay đổi theo frame (motion blur, biến dạng) thì
  hoặc phát asset mới mỗi frame, hoặc chấp nhận sai số ở mép và phải gate bằng verify.
* **PA**: trimap từ dilation/erosion rồi "continuous alpha estimation" không nêu tên thuật toán.
  Dùng thẳng MatAnyone (mask frame đầu → alpha toàn video). Decontamination dùng Germer 2020 hoặc
  FBA; công thức chia cho α trong concept sẽ khuếch đại nhiễu ở biên, nơi α nhỏ, đúng vùng mắt nhìn
  thấy halo.
* **HỞ**: decontamination cần `B_old(x,y,t)`, tức nền *đã inpaint* tại thời điểm đó, nhưng inpaint
  nằm ở Giai đoạn 3. Phụ thuộc vòng, giải tuần tự được, nhưng lỗi inpaint sẽ đi thẳng vào màu viền
  foreground.
* **Fill-rate (R3)**: PNG có alpha vẫn tốn fill theo cả quad. Mask SAM loang rộng (tóc, khói) sinh
  rect lớn gần như trong suốt. Tight bbox như concept nói là đúng hướng nhưng chưa đủ; cần cap tổng
  diện tích alive theo `SteadySbLoad ≤ 4.0 viewport` (direction 2.2).

### 2.2 Giai đoạn 2: S2D, trắc quang, fall-through

* **OK**: Sim(2) là nhóm đúng cho osu! (transform tuyến tính là `R(θ)·diag(sx,sy)`, không shear).
  Chọn Sim(2) thay affine như Zhang 2023 là quyết định đúng, và khớp với T4 trong direction §4.1.
* **HỞ**: Umeyama cần *tương ứng điểm*. Concept nói "trích xuất feature hoàn toàn bên trong mask",
  nhưng Zhang 2023 ghi rõ SIFT/KLT và cả flow neural kém trên nội dung ít texture, phải dùng
  differentiable compositing có khởi tạo bằng shape context + flow + RANSAC. Với footage có texture
  thì KLT trong mask đủ; với cartoon và motion graphics thì không. Cần hai đường: dense tracker
  (CoTracker 3) hoặc fit trực tiếp trên pixel (ECC / DC).
* **OK nhưng nhỏ**: `C` tint. Nhận xét "C chỉ nhân, chọn frame sáng nhất làm master" đúng. Bài toán
  bounded LSQ có nghiệm đóng (tỷ số tích vô hướng, clip về [0,1]), không cần `lsq_linear`. Lợi ích
  thật chỉ ở fade và đổi sáng *đều* trên cả sprite; ánh sáng thật hiếm khi đều, gate SSIM sẽ loại
  phần lớn. Lưu ý thêm từ implementation-plan: fade phải dùng `C` về đen, không dùng `F`, vì video
  nằm trên storyboard khác và `F` làm lộ phần dưới.
* **HỞ**: additive. `Base = min_t I` bị nhiễu kéo xuống một cách hệ thống (min của mẫu nhiễu luôn
  dưới trung bình), rồi lớp Light phải bù cả nhiễu, không còn thưa. Dùng percentile thấp thay min.
  Quan trọng hơn: factor hạng 1 chỉ đúng khi vệt sáng *đứng yên trong hệ tọa độ sprite*. Concept
  nói "lệnh M bám theo chuyển động trôi của vệt sáng", nghĩa là phải track vệt sáng trước rồi mới
  factor trong hệ đã bù chuyển động, mà track vệt sáng chính là điểm yếu ở 2.1.
* **OK**: các ngưỡng gate (PSNR 34, SSIM 0,92, clamp 2%) hợp lý về bậc, trùng preset `medium`. Nhưng
  gate phải đo trên *composite render* chứ không trên từng object: lỗi mép giữa hai layer chỉ hiện ra
  khi composite.
* **SAI**: nhánh Animation cho "pacing tuyến tính". `Animation` của osu! preload toàn bộ N frame vào
  memory (R4). Dùng cho object nhỏ thì được; dùng cho vùng lớn thì không, xem 2.3.

### 2.3 Giai đoạn 3: nhóm, cost, freeze gate, nền

* **SAI, quan trọng nhất**: Lossless Freeze Gate so `I_{t+1}` với `I_t`. Đây là vòng hở: thay đổi
  chậm (dissolve, gradient trôi, ánh sáng đổi dần) luôn dưới ngưỡng từng bước nên tích lũy vô hạn.
  Đây *chính xác* là nguyên nhân gốc gây mờ của bản hiện tại (direction 1.1, `TileRunTracker`). Phải
  so frame hiện tại với *asset đang hiển thị* (canonical), và tốt nhất là so composite render với
  source. Toàn bộ direction §8 (closed loop, `ErrorMap` sau `AdvanceTo`) được xây để sửa đúng lỗi này.
* **SAI**: Cost function `w1·AssetSize + w2·CommandCount + w3·Overdraw` với trọng số tay là mô hình
  *dự đoán*. Spike heatmap partitioning (direction 13.4) có ước lượng đẹp nhưng đo thật thua 2,6 đến
  5,4 lần; tầng RDO đã bị bác (13.3). Chi phí asset là bytes PNG *sau encode*, chỉ biết khi encode.
  Ba trục workload là ràng buộc cứng (vách đá), không phải trọng số (direction 2.2).
* **OK**: morphological open/close chống sprite 1×1; motion-coherence grouping thay semantic.
  Nhưng nếu grouping theo motion coherence là tiêu chí cuối, thì SAM 2 chỉ còn là một *nguồn
  ứng viên* cho layer, không phải xương sống. Nên viết đúng như vậy (mục 6).
* **SAI**: "Nền động → chuỗi Animation Sprite". Nền full-frame 1080p, 300 frame, preload toàn bộ là
  khoảng 2,4 GB texture resident. Nền động phải là chuỗi sprite time-disjoint (T2) hoặc, khi là
  parallax/pan, là layer T4/T6. Concept không phân biệt "nền động vì camera" (giải được bằng K=1 và
  panorama, rẻ) với "nền động vì nội dung" (sóng, lá; chỉ có T2/T7 có cap).
* **HỞ**: nền tĩnh "không ảo giác" từ lookahead đúng về nguyên tắc (median theo pixel trong hệ
  panorama, direction 5.5), nhưng vùng *chưa từng lộ* thì không có dữ liệu. Để trong suốt và để
  closed loop vá khi lộ là cách đúng; concept chưa nói.
* **HỞ**: z-order. SAM 2 không cho thứ tự sâu. Suy được từ sự kiện occlusion (mask nào thắng pixel
  nhìn thấy), dựng đồ thị ràng buộc và topological sort; có chu trình là phân rã sai (direction 6.2).
  Concept chỉ viết "Native Z-Order".

### 2.4 Giai đoạn 4: CAS, fingerprint, độ phân giải

* **SAI về cú pháp**: `Animation` của osu! đặt tên frame bằng cách *nối thẳng* chỉ số vào trước
  phần mở rộng: path `anim_x.png` → `anim_x0.png`, `anim_x1.png`. Ví dụ trong concept
  (`..._a1f9e0c4b82d_0.png`) sẽ không load trừ khi prefix kết thúc bằng `_`. Repo hiện dùng
  `a/{hash}/f{n}.png` là đúng.
* **SAI về rủi ro**: dedupe theo pHash Hamming ≤ 2 rồi dùng asset khác byte. Hai asset "gần giống"
  thay nhau giữa hai run kề nhau sinh pop nhìn thấy được. Direction 5.4 và bảng 11: canonical
  *exact* hash trước (chuẩn hóa pose rồi hash); near-dup chỉ khi verify SSIM so với sàn, và tới nay
  là REJECT cho tới khi canonical reuse ship mà bucket near-dup vẫn lớn.
* **OK**: Max-Resolution Promotion và `s = W_target / W_png` khớp R2 (author asset ở độ phân giải
  nguồn rồi scale xuống không gian 640). Nhưng promotion làm tên asset phụ thuộc *lịch sử* cache,
  mâu thuẫn với D9 ("tên chỉ phụ thuộc pixel"). Cần tách: key theo pixel, promotion là bước chọn
  instance khi emit.
* **OK**: RDP cho keyframe; đã có trong `TrajectoryFitter` cũ và là lớp (3) của direction 5.7.

### 2.5 Giai đoạn 5: easing và curve fitting

* **SAI**: `Cubic Bézier` và `Custom Parametric Curve` không phát được ra `.osb`. osu! chỉ có 35
  easing cố định (0–34: Linear, Out, In, Quad/Cubic/Quart/Quint/Sine/Expo/Circ In/Out/InOut,
  Elastic ×5, Back ×3, Bounce ×3), không có tham số tự do ngoài `(v0, v1)` của segment. Bézier chỉ
  xấp xỉ được bằng chuỗi segment easing hoặc linear. Vậy "Curve Model" phải là tập hữu hạn đó, và
  bài fit là *chọn* chứ không *tối ưu tham số* (chính điều này làm nó rẻ: direction 6.3).
* **OK**: piecewise + model selection error + λ·complexity là đúng hướng; đây là changepoint với
  cost là residual của easing tốt nhất trên đoạn. PELT hoặc DP cho nghiệm tối ưu với O(n log n) đến
  O(n²), hoàn toàn đủ cho vài nghìn sample mỗi track. "Lossless mode (Error = 0)" chỉ có nghĩa trên
  nội dung được author bằng tween; trên quỹ đạo nhiễu thì không tồn tại và phải fallback keyframe.
* **PA**: không có paper riêng cho nhận dạng easing từ quỹ đạo; gần nhất là Roberts 2018 (optimal
  keyframe selection giống animator), Nowicki 2019 (speed trong animation vẽ tay xấp xỉ bậc ba),
  Sawicki 2020 (Bézier tổng quát hóa easing), và các tool thực dụng fit cubic-bezier/spring từ
  screen recording. Đây là một contribution nhỏ nhưng có thật nếu làm cẩn thận với dataset ground
  truth (direction 6.5, R1/R2).

---

## 3. Năm lỗi phải sửa trước khi xây gì

| # | Lỗi | Vì sao chết người | Sửa |
| --- | --- | --- | --- |
| 1 | Freeze gate so `I_{t+1}` với `I_t` | vòng hở, drift tích lũy; đúng bug gây mờ hiện tại | so với asset đang hiển thị, verify trên composite render |
| 2 | Nền động → `Animation` | R4: N frame full-res resident cùng lúc | T2 (sprite time-disjoint) hoặc T4/T6 khi là camera motion; `Animation` chỉ cho object nhỏ, có cap bytes resident |
| 3 | Cubic Bézier / custom curve | không có trong `.osb` | tập 35 easing, chọn rời rạc + piecewise |
| 4 | Swap asset theo pHash Hamming ≤ 2 | pop/flicker giữa run kề | canonical exact hash; near-dup chỉ sau verify SSIM |
| 5 | Cost function trọng số tay | predict thua measure 2,6–5,4× trong repo | ladder rẻ → đắt, chi phí đo thật (bytes sau encode, workload từ timeline), ràng buộc cứng thay trọng số |

Ba lỗi nhỏ hơn: tên frame Animation (`_0.png`), SAM 2 không tự liệt kê object, "F" cho fade.

---

## 4. Dự đoán chất lượng và hiệu năng

### 4.1 Theo loại nội dung (so với bản hiện tại và với lộ trình M1–M3 của direction.md)

| Nội dung | Fixture tương ứng | Dự đoán với concept (sau khi sửa mục 3) | Phần lợi ích đến từ đâu | So với direction M2–M3 |
| --- | --- | --- | --- | --- |
| Motion graphics, UI game, logo, text | (chưa có fixture; nên thêm Crello Animation / Animation2Code) | giảm sprite và asset 1–2 bậc độ lớn; easing khớp chính xác; nhiều object 0-byte | object layer + Sim(2) + easing | thắng rõ so với K=1, vì nội dung vốn là nhiều object độc lập |
| 2D animation sạch | short_animation | giảm mạnh ở nhân vật rigid (xe, vật thể), ít ở nhân vật biến dạng (mặt, tay) → fall-through | layer + canonical reuse | thắng vừa; bucket biến dạng vẫn là patch/crossfade |
| Game render 3D có camera | minecraft (79.612 sprite, 735 MB, 542 phút ở 60fps hôm nay) | phần lớn lợi ích từ K=1 + panorama nền; object foreground (tay, mob) parallax ngoài mặt phẳng → fall-through | global motion, panorama | gần bằng M2 + panorama; SAM thêm ít |
| Footage thật, nhiễu | birdbrain, fish_spinning | gate lossless không bao giờ đạt; decontamination khuếch đại nhiễu ở biên; S2D hội tụ nhưng residual lớn | gần 0 | không hơn M2; nhiễu là ranh giới vật lý (direction §7) |
| Silhouette nhấp nháy | bad_apple | SAM mask tốt nhưng morph phi rigid mỗi frame → Animation/T2 như hôm nay | 0 | bằng M3 |

Hai hệ quả: (1) corpus fixture hiện tại thiên về game và footage, là nơi concept thắng *ít nhất*;
muốn chứng minh concept phải thêm fixture motion graphics và có ground truth chương trình (direction
6.5); (2) trên corpus hiện tại, thứ tự lợi ích vẫn là closed loop → crossfade/photometric → K=1 →
panorama → layer; SAM 2 chỉ đổi *cách tìm layer*, không đổi thứ tự này.

### 4.2 Chất lượng hình

* Hết mờ do cấu trúc chỉ khi gate chạy trên composite render với sàn tuyệt đối (sửa lỗi 1). Bản
  thân object-centric không sửa được mờ; vòng hở theo object vẫn mờ y như vòng hở theo tile.
* Artifact mới mà bản hiện tại không có: viền/halo ở biên layer (matting sai), nền inpaint sai lộ ra
  khi object rời đi, pop khi canonical image đổi, z-order sai gây "xuyên" object. Tất cả đều là lỗi
  *cục bộ nhưng rất dễ thấy*, khác với lỗi tile (đều, khó thấy). Verify theo worst-tile SSIM bắt được.

### 4.3 Hiệu năng compile

Giả định: 1080p, 60fps, 3 phút = 10.800 frame, 10 object được track.

| Khối | Số đo công khai | Ước lượng |
| --- | --- | --- |
| SAM 2 (Hiera-L, A100, 1024px) | 24,2 fps/object, 30,2 fps compiled; B+ 34,8; S 43,3; T 47,2; mỗi object chạy riêng, chỉ dùng chung image encoder | 10 object × 10.800 frame ≈ 108k lượt → khoảng 1 giờ trên A100 với L; GPU tiêu dùng chậm 3–10× |
| MatAnyone | per-object, per-frame, cùng bậc với SAM 2 | +0,5 đến 1 giờ |
| Flow / tracker cho Umeyama | RAFT hoặc CoTracker, 1 lần/frame | +10 đến 30 phút |
| Inpainting nền (ProPainter) | nặng theo diện tích che và độ dài | +30 phút trở lên; có thể bỏ nếu dùng median panorama |
| Verify render + PNG encode | như hiện tại | cùng bậc bản hiện tại |

So với 542 phút CPU của bản hiện tại trên Minecraft, một pipeline GPU không nhất thiết chậm hơn,
nhưng đổi kiến trúc triển khai: dependency Python/PyTorch hoặc ONNX Runtime + model vài trăm MB +
GPU bắt buộc, trái với D6/D8 (thư viện .NET thuần). Cần quyết định sản phẩm trước khi quyết định
kỹ thuật: Aurbey là thư viện .NET hay là pipeline nghiên cứu Python phát `.osb`.

### 4.4 Hiệu năng runtime trong osu!

Tốt hơn bản hiện tại về số sprite và command (R3, R10), nhưng hai rủi ro mới: fill-rate từ rect alpha
lớn, và `Animation` resident bytes. Cả hai đã có ràng buộc số trong direction 2.2; concept chưa có.

---

## 5. Khó khăn chưa giải quyết, xếp theo mức chặn

1. **Verify và chi phí đo thật** chưa có trong concept. Không có cái này thì mọi tầng trên không
   kiểm soát được lỗi, và mọi so sánh với bản hiện tại là vô nghĩa.
2. **Khởi tạo và đếm object tự động.** Ai quyết định có bao nhiêu layer, prompt ở đâu, khi nào tách
   hay gộp? Zhang 2023 và Suzuki 2024 đều cần input người dùng hoặc giả định motion graphics. SAM 3
   cần noun phrase. Tự động hoàn toàn trên video bất kỳ là bài toán mở; cách an toàn là motion
   coherence (direction 5.1) đề xuất layer, SAM tinh chỉnh biên.
3. **Re-identification xuyên che khuất dài và xuyên scene.** Tracker không làm được; phải làm bằng
   canonical-hash của asset. Chưa thiết kế.
4. **Mép layer.** Matte đổi theo frame nhưng sprite chỉ có một texture; motion blur; fill-rate của
   rect alpha. Chưa có cơ chế quyết định khi nào mép mềm đáng giá hơn patch.
5. **Nền chưa từng lộ** và nền có parallax (homography không mô tả được, đúng failure case của
   Omnimatte). Chưa có nhánh.
6. **Biến dạng phi rigid chậm** (người đi xa, vải, mặt) là bucket lớn trong animation; concept đẩy
   hết vào fall-through, trong khi T3 crossfade lồng T4 (direction 4.3) rẻ hơn nhiều. Concept không có
   crossfade.
7. **Group transform của `AnimationVideo`.** Mọi tile sinh ra phải compose qua `GroupTransformBaker`
   vì track cùng property không được chồng (R5). Concept phát `M`, `S`, `R`, `C`, `F` trực tiếp trên
   object mà không nói gì về việc compose với transform của group.
8. **Easing trên nội dung không được author**: "lossless" không tồn tại; cần định nghĩa sai số cho
   phép trong screen space và đo xem easing tiết kiệm được bao nhiêu so với RDP linear (R2 keep-gate:
   ≥ 30% command bytes trên nội dung được author, 0 regression).

---

## 6. Đề xuất: hòa giải concept với `direction.md`

Concept và direction.md không đối nghịch nếu đặt đúng vai:

```text
closed loop + verify + sàn tuyệt đối            (direction §8, giữ nguyên, là máy đo)
   └─ ladder representation rẻ → đắt           (direction 5.7, giữ nguyên)
        ├─ Photometric, Crossfade, Patch         (M1)
        ├─ TransformTrack K=1, Panorama          (M2)
        └─ MotionLayerSet K>1                    (DEFER ở direction; Aurbey vào đây)
             ├─ nguồn ứng viên layer A: block flow + sequential RANSAC   (classical)
             └─ nguồn ứng viên layer B: SAM 2/3 + MatAnyone (+ Omnimatte cho effect)
                  → cùng một contract: {mask(t), alpha(t), Sim(2) track, canonical asset}
                  → closed loop đo chi phí thật, sai thì trả bằng patch, không bằng artifact
```

Nghĩa là: Aurbey là **một candidate provider** cho tầng layer, chạy sau M2, và phải thắng gate đã
ghi ở direction bảng 11 ("bucket local-motion từ 25 đến 30% bytes" sau attribution M3.5). Những thứ
trong concept cần giữ nguyên văn: Sim(2) thay affine; additive layer cho glow; easing fitting rời
rạc; canonical asset + instance. Những thứ bỏ: cost function trọng số, pHash swap, nền động bằng
Animation, freeze gate theo frame trước.

Thứ tự spike, mỗi cái có số pass/fail trước khi chạy (cùng tinh thần `feasibility-plan.md`):

| Spike | Câu hỏi | Fixture | Pass |
| --- | --- | --- | --- |
| A1 | Layer từ SAM 2 + MatAnyone có thắng layer từ block flow + RANSAC trên cùng contract không | short_animation, 1 clip motion graphics, minecraft 10s | bytes ≤ 0,7× và verify pass 100% trên ít nhất 2/3 |
| A2 | Sim(2) từ KLT-trong-mask so với CoTracker so với ECC trên pixel | như trên | residual screen-space < 0,5 px ở ≥ 90% frame trên motion graphics |
| A3 | Easing rời rạc + PELT so với RDP linear | dataset sinh từ generator (direction 6.5) + fixture | ≥ 30% command bytes trên nội dung author, 0 regression |
| A4 | Additive factor (percentile base) trên glow thật | clip có flash/glow | SSIM worst-tile ≥ sàn với 2 asset thay vì N patch |
| A5 | Chi phí GPU end-to-end so với 542 phút CPU | minecraft full | nêu số, không gate |

---

## 7. Về tính mới cho khóa luận: trả lời ba câu hỏi trong email

**Câu 1. Biểu diễn video dưới dạng thực thể, trắc quang, motion, quan hệ đã được giải tới đâu?**
Với motion graphics: gần như xong ở mức affine + canonical image + z-index (Zhang 2023, Suzuki 2024,
có dataset và metric). Với cartoon: Sprite-from-Sprite. Với video tự nhiên: omnimatte line giải được
object + effect, nhưng cần mask đầu vào, nền tĩnh hoặc 3D, và tốn phút tới giờ mỗi clip. Trắc quang
tách riêng khỏi hình học đã có từ intrinsic video. Quan hệ che khuất đã có qua z-index per-frame và
layer compositing. Nói ngắn: "phân rã thành thực thể và phân loại biến đổi hình học/trắc quang/nội
dung" không còn là khoảng trống.

**Câu 2. Phân tích biến đổi và chọn representation theo từng trường hợp có là bài toán khác?**
Chỉ khác nếu *target* khác và việc chọn được định nghĩa là tối ưu có kiểm chứng. Điểm khác biệt
có thể bảo vệ được:

* Target là một DSL *thực thi bởi renderer bên thứ ba, không khả vi, có ràng buộc cứng khác SVG*:
  không shear, không clip, tint chỉ nhân, additive là cách duy nhất làm sáng, Animation preload, track
  cùng property không chồng, chi phí fill-rate và alive count. SVG Motion không phải đối mặt những
  ràng buộc này, và phần lớn cái "chọn" trong concept là hệ quả trực tiếp của chúng.
* Mục tiêu là *nén có kiểm chứng* (min bytes dưới sàn fidelity mọi frame, mọi tile), không phải
  "edit được". Đây là rate–distortion trên không gian chương trình, họ hàng với MPEG-4 sprite coding
  hơn là với video editing, và là góc nhìn mà các paper decomposition không đo (họ đo PSNR của
  reconstruction và user study, không đo bytes, không đo workload renderer).
* Không có identifiability toàn cục (vô số chương trình render ra cùng video), nên "truy hồi đúng
  chương trình gốc" là mục tiêu sai; mục tiêu đúng là chương trình rẻ nhất trong sai số ε (direction
  6.1). Điều này dẫn tới câu hỏi đo lường có giá trị riêng: khi nào bộ chọn ladder greedy xa tối ưu,
  và metric tầng hai nào tương quan với "chọn đúng" (direction 6.5).
* Animation2Code (NeurIPS 2026) vừa chứng minh VLM de-render animation sang code với temporal
  similarity ~0,3 dù appearance ~0,8, kể cả sau fine-tune. Một pipeline cổ điển/tối ưu đạt fidelity
  *được verify* trên chính bài toán này là kết quả có thể so sánh trực tiếp.

Nếu viết đề tài là "object-centric video decomposition" thì nó là biến thể; nếu viết là **"biên dịch
video thành chương trình animation có kiểm chứng: lựa chọn representation tối ưu chi phí trong một
DSL ràng buộc"** thì nó là bài toán riêng, với baseline rõ (Zhang 2023 retarget sang `.osb`, bản
tile-grid hiện tại, MPEG-4 sprite coding) và metric rõ (bytes, sprite/command, workload, PSNR/SSIM
worst-tile, complexity_ratio và representation_confusion so với ground truth chương trình).

**Câu 3. Đủ phạm vi và chiều sâu cho khóa luận CV không?**
Đủ, với điều kiện thu hẹp. Phạm vi an toàn cho một khóa luận:

1. Hệ thống: closed-loop compiler với ladder provider, trong đó một provider là layer từ SAM 2/3 +
   matting với Sim(2) track và easing rời rạc. Không làm Omnimatte, không làm inpainting generative.
2. Thực nghiệm: corpus ba loại (motion graphics có ground truth, 2D animation, game render), so với
   ba baseline trên, báo cáo cả bytes lẫn fidelity lẫn workload.
3. Phân tích: attribution bytes theo bucket (nội dung nào đi vào primitive nào), oracle gap của
   ladder, ablation easing so với RDP, ablation SAM-layer so với flow-layer.
4. Dataset nhỏ có ground truth chương trình (generator + renderer đã có trong repo) là contribution
   phụ độc lập, giống cách Suzuki 2024 và Animation2Code xây benchmark.

Phần nên cắt khỏi khóa luận: CAS/pHash/HNSW (kỹ thuật hệ thống, không có câu hỏi nghiên cứu), nền
động generative, sub-entity vệt sáng qua SAM.

Lưu ý cho email: nên dẫn thêm MPEG-4 sprite coding và Omnimatte vào phần related work, vì giảng
viên CV sẽ hỏi ngay; và nên nêu thẳng rằng novelty nằm ở tầng chọn/kiểm chứng, không ở decomposition,
để tránh bị đánh giá là biến thể của SVG Motion.

---

## 8. Tài liệu đã đối chiếu

Decomposition và vectorization

* Zhang, Ma, Wu, Ritchie, Agrawala. *Editing Motion Graphics Video via Motion Vectorization and
  Transformation*. SIGGRAPH Asia 2023 / TOG. https://arxiv.org/abs/2309.14642 ; project
  https://sxzhang25.github.io/publications/motion-vectorization ; supplement (failure cases)
  https://sxzhang25.github.io/publications/pdfs/motion-vectorization-supp.pdf
* Suzuki, Kikuchi, Yamaguchi. *Fast Sprite Decomposition from Animated Graphics*. ECCV 2024.
  https://arxiv.org/abs/2408.03923
* Zhang, Wong, Liu. *Sprite-from-Sprite*. SIGGRAPH Asia 2022. https://lllyasviel.github.io/GitPageToonDecompose
* Ye et al. *Deformable Sprites for Unsupervised Video Decomposition*. CVPR 2022. https://arxiv.org/abs/2204.07151
* Kasten et al. *Layered Neural Atlases for Consistent Video Editing*. SIGGRAPH Asia 2021. https://arxiv.org/abs/2109.11418
* Lu et al. *Omnimatte*. CVPR 2021. https://arxiv.org/abs/2105.06993 ; Lin et al. *OmnimatteRF*. ICCV 2023;
  Lee et al. *Generative Omnimatte*. CVPR 2025. https://arxiv.org/abs/2411.16683
* Smirnov et al. *MarioNette: Self-Supervised Sprite Learning*. NeurIPS 2021.
* Elsayed et al. *SAVi++*. NeurIPS 2022 (object-centric learning trên video thật còn yếu khi không có điều kiện).

Segmentation, tracking, matting

* Ravi et al. *SAM 2*. ICLR 2025. https://arxiv.org/abs/2408.00714 (Appendix C Limitations; tốc độ A100 trong repo facebookresearch/sam2)
* *SAM 3: Segment Anything with Concepts*. 11/2025. https://arxiv.org/abs/2511.16719 ; SAM 3.1 Object Multiplex 03/2026 (README repo facebookresearch/sam3)
* Yang et al. *MatAnyone*. CVPR 2025. https://pq-yang.github.io/projects/MatAnyone
* Germer et al. *Fast Multi-Level Foreground Estimation*. ICPR 2020. https://arxiv.org/abs/2006.14970
* Forte, Pitié. *F, B, Alpha Matting*. 2020.
* Hudson, Smith. *Track Anything Behind Everything*. https://arxiv.org/abs/2411.19210
* Zhou et al. *ProPainter*. ICCV 2023. https://arxiv.org/abs/2309.03897

Codec và biểu diễn layer

* Glantz, Krutz, Sikora. *Automatic MPEG-4 sprite coding*. 2010. http://elvera.nue.tu-berlin.de/files/1227Glantz2010.pdf
* Krutz et al. *Automatic Object Segmentation Algorithms for Sprite Coding using MPEG-4*. 2008.
* Watanabe, Jinzenji. *Sprite Coding in Object-based Video Coding Standard: MPEG-4*. https://www.ams.giti.waseda.ac.jp/data/pdf-files/R03.pdf
* Wang, Adelson. *Representing Moving Images with Layers*. IEEE TIP 1994.
* Sunkavalli et al. *Factored Time-Lapse Video*. SIGGRAPH 2007. https://pixl.cs.princeton.edu/gfx/pubs/Sunkavalli_2007_FTV/ftlv.pdf
* Ye et al. *Intrinsic Video and Applications*. SIGGRAPH 2014.

Keyframe, easing, changepoint

* Roberts. *Converting Motion Capture Into Editable Keyframe Animation*. PhD thesis 2018. https://doi.org/10.26686/wgtn.17067632
* Nowicki, Cowan, Mann. *Analysis of Speed in Traditional Animation*. GI 2019.
* Sawicki et al. *Bézier Curve as a Generalization of the Easing Function*. 2020.
* Killick, Fearnhead, Eckley. *PELT*. JASA 2012.
* Zhang et al. *Trajectory Simplification: An Experimental Study*. PVLDB 2018.

Benchmark de-rendering

* Ji et al. *Animation2Code*. NeurIPS 2026. https://anya-ji.github.io/animation2code-website/
* Liang et al. *VAnim*. ICML 2026. https://arxiv.org/abs/2605.01517
* Suzuki et al. Crello Animation dataset (xem trên).
