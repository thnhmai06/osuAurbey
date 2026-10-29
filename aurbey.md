### Kiến trúc Hoàn chỉnh: Pipeline Biên dịch Video sang osu! Storyboard Dựa trên Thực thể (Object-Centric Representation Pipeline - Aurbey v2)

Pipeline được chuẩn hóa thành 4 giai đoạn xử lý khép kín: **Định danh & Trích xuất quỹ đạo (kèm Alpha Matting & Khử lem viền)** $\longrightarrow$ **Phân loại & Biểu diễn Delta (Similarity 2D, Trắc quang & Vệt sáng trôi, Phân tầng Frame do giới hạn góc nhìn)** $\longrightarrow$ **Chiến lược Tách/Nhóm (Motion Coherence & Cost Function) & Khôi phục nền (Amodal Static / Dynamic Fallback)** $\longrightarrow$ **Lưu trữ Bất biến & Khử trùng lặp CAS**.

```text
[Video Input Stream]
         │
         ▼
[GIAI ĐOẠN 1: SAM 2 WORKER & MATTING/DECONTAMINATION ENGINE]
  • Gán nhãn ID xuyên suốt video qua Memory Attention & FIFO
  • Bóc tách Sub-Entities (vật thể chính, vệt sáng di chuyển độc lập)
  • Tự động tạo Trimap từ mặt nạ: Dilation/Erosion ranh giới biên (r = 2-5px)
  • Continuous Alpha Estimation (A_k(t) in [0, 1]) & Foreground Decontamination
  • Triệt tiêu viền lem màu nền cũ (Color Bleeding/Halo Artifacts) & Chuẩn hóa Straight/Premultiplied Alpha
  • Xuất chuỗi trạng thái T_k = {M_k(t), A_k(t), F_k(t), BBox_k(t)}
         │
         ▼
[GIAI ĐOẠN 2: PHÂN TÍCH DELTA & BIỂU DIỄN THAM SỐ]
         │
         ├──► [Nhánh 1: Similarity 2D (S2D)] ──(Đạt sàn hình học Sim(2))──► Lệnh M, S, R (0 byte asset)
         │            │ (Khóa chết tọa độ & góc xoay về Canonical Patch qua Umeyama SVD)
         │            ▼
         ├──► [Nhánh 2: Xử lý Trắc quang & Bù sáng (Photometry)]
         │            ├─► Phép nhân màu toàn cục: Lập trình tuyến tính có chặn (Bounded LSQ) ──► Lệnh C (0 byte asset)
         │            ├─► Vệt sáng tách rời / Cộng năng lượng: Phân tách Priors Khóa mốc ──────► Lệnh P, A + M, F (Layer Additive độc lập)
         │            │   (Khóa Base min_t, Khóa Light max_t delta; hồi quy giải tích alpha(t))
         │            ▼
         └──► [Nhánh 3: Fall-through Phân tầng Frame (Giới hạn góc nhìn 3D & Phần cứng Render)]
                      │ (Khi vi phạm S2D do quay ngoài mặt phẳng SE(3), biến dạng mềm, hoặc trắc quang vỡ:
                      │  PSNR < 34dB / SSIM < 0.92 / Bão hòa > 2% / Bóng đổ âm 3D)
                      │
                      ├─► [Pacing tuyến tính & Loop / Single-cycle] ──► Animation Sprite (anim_*)
                      └─► [Phi tuyến / Scene Boundary / Idle] ────────► Sprite Replacement (Tách rời)
                                │
                                ▼
[GIAI ĐOẠN 3: TÁCH/NHÓM TRẠNG THÁI & KHÔI PHỤC NỀN]
  • Spatiotemporal Motion-Coherence Grouping (Gom cụm dựa trên tương quan vector chuyển động, không theo ngữ nghĩa)
  • Dynamic Cost Function Optimizer (Cân bằng dung lượng đĩa vs Overdraw vs Command count/Draw calls)
  • Bộ lọc hình thái học Morphological Open/Close (Triệt tiêu nhiễu hạt, cấm spam pixel 1x1)
  • Lossless Freeze Gate (Bit-exact / Screen-Quantization MAE ≤ 1 -> Kéo dài lifetime tEnd)
  • Amodal Background / Dynamic Fallback:
      ├─► Nền tĩnh (Lookahead Buffer Inpainting không ảo giác) ──► 1 File PNG phẳng + Native Z-Order
      └─► Nền động (Optical flow khác không: sóng nước, lá bay)  ──► Chuỗi Animation Sprite (anim_*)
         │
         ▼
[GIAI ĐOẠN 4: LƯU TRỮ ĐỊNH DANH BẤT BIẾN & KHỬ TRÙNG LẶP (CAS)]
  • Single Sprite Fingerprint: obj_{AspectBin}_{pHash128}_{ColorHash64}.png
  • Sequence Fingerprint: anim_{AspectBin}_{Count}_{MerkleBlake3}.png
  • Khử liên kết Không-Thời gian: Two-Tier Memory + RDP Algorithm triệt tiêu keyframe thừa
  • Max-Resolution Promotion & Bù trừ Scale logic tự động (s = W_target_osb / W_png)
  • Tra cứu O(1) / O(log N) (MIH / HNSW)

```

---

### Giai đoạn 1: Định danh Thực thể, Trích xuất Quỹ đạo, Alpha Matting & Khử Lem Viền (Color Decontamination)

Hệ thống kết hợp SAM 2 và bộ vi xử lý trích xuất Alpha biên để bóc tách triệt để ranh giới quang học và màu sắc thực của vật thể:

1. **Khởi tạo & Bảo toàn ID:**
* SAM 2 gán nhãn từng thực thể (Foreground objects, Occluders, Background).
* Khối **Memory Attention & FIFO Memory Bank** của SAM 2 tự động duy trì tracking ID xuyên suốt timeline, bảo toàn danh tính vật thể kể cả khi bị che khuất hoàn toàn (Full Occlusion) rồi xuất hiện trở lại.
* **Bóc tách Thực thể phụ (Sub-Entity Decoupling):** Nếu có một vệt sáng, đốm phản xạ hoặc chi tiết chuyển động trượt độc lập trên bề mặt đối tượng (do góc nhìn 3D hoặc nguồn sáng di chuyển), SAM 2 chủ động theo dõi nó như một sub-entity độc lập, chuyển toàn bộ trách nhiệm chuyển động cho S2D để cô lập vùng pixel cho bộ xử lý trắc quang / Additive Blending.


2. **Cơ chế Khử Viền Lem & Răng Cưa (Continuous Alpha & Foreground Decontamination):**
* **Sinh Trimap Tự động từ Mask nhị phân:** Từ mặt nạ $\mathbf{M}_k(t) \in \{0, 1\}$ thô của SAM 2, hệ thống thực hiện phép toán hình thái học Dilation (giãn) và Erosion (co) với bán kính vi phân $r \in [2, 5]\text{ px}$ để phân chia không gian thành 3 vùng:



$$\Omega = \Omega_{\text{FG}} \cup \Omega_{\text{BG}} \cup \Omega_{\text{Unknown}}$$

Trong đó $\Omega_{\text{Unknown}}$ bao trọn dải ranh giới chứa sợi tóc, chuyển động mờ (motion blur), hoặc mép bán minh bạch.

* **Ước lượng Alpha Liên tục ($\mathbf{A}_k(t)$):** Vùng $\Omega_{\text{Unknown}}$ được giải qua mô hình continuous alpha estimation nhằm thu về ma trận độ trong suốt thực tế $\alpha(x, y) \in [0.0, 1.0]$ thay vì cắt cụt nhị phân gây bậc thang (aliasing).
* **Khử nhiễm màu nền cũ (Foreground Color Decontamination):**
Pixel quan sát tại biên là tổ hợp: $I(x, y) = \alpha(x, y) F(x, y) + (1 - \alpha(x, y)) B_{\text{old}}(x, y)$.
Để triệt tiêu hiện tượng viền sáng/đen lem nhem (Halo Artifacts / Color Bleeding) khi dán đè lên nền mới đã được khôi phục, thuật toán giải phương trình ngược để bóc tách màu gốc $F$:

$$F(x, y) = \frac{I(x, y) - (1 - \alpha(x, y)) \cdot B_{\text{old}}(x, y)}{\alpha(x, y)} \quad \forall (x, y) \in \Omega_{\text{Unknown}}$$

```
 *(Với các điểm \(\alpha(x, y) \to 0\), giá trị \(F\) được nội suy khuếch tán từ vùng \(\Omega_{\text{FG}}\) lân cận để tránh lỗi chia ma trận cho 0).*

```

* **Chuẩn hóa Alpha Output:** Texture được format về hệ màu tương thích hoàn toàn với cơ chế pha trộn của osu! (khử sạch nền tối giả lập).

3. **Trích xuất Chuỗi Trạng thái Chuẩn hóa (State Trajectory):**
Mỗi thực thể $k$ được mô tả bằng chuỗi trạng thái đầy đủ thông tin:

$$\mathcal{T}_k = \{ \mathbf{S}_k(t) \}_{t=t_{\text{start}}}^{t_{\text{end}}} \quad \text{với} \quad \mathbf{S}_k(t) = \left( \mathbf{M}_k(t), \ \mathbf{A}_k(t), \ \mathbf{F}_k(t), \ \text{BBox}_k(t) \right)$$

* $\mathbf{M}_k(t) \in \{0, 1\}$: Mặt nạ phân loại vùng.
* $\mathbf{A}_k(t) \in [0, 1]$: Kênh Alpha liên tục mượt mà.
* $\mathbf{F}_k(t) \in \mathbb{R}^{H \times W \times 3}$: Phân bố màu RGB nguyên bản sau khi đã khử sạch màu nền cũ.
* $\text{BBox}_k(t)$: Hộp bao khít tối thiểu (Tight Bounding Box) triệt tiêu tối đa diện tích alpha trong suốt, bảo vệ GPU Fill-rate của osu!.

---

### Giai đoạn 2: Phân tích Delta giữa các Trạng thái & Biểu diễn Tham số

Tại mỗi bước chuyển $t \to t+1$, compiler đo lường độ sai lệch giữa hai trạng thái để phân nhánh vào các cơ chế tối ưu:

#### 1. Nhánh Biểu diễn Tối ưu Hình học (Similarity 2D - S2D)

* **Điều kiện kích hoạt:** Thực thể chỉ chuyển động phẳng trong nhóm $Sim(2)$ (dịch chuyển, xoay trong mặt phẳng, co giãn đồng dạng), không bị biến dạng phi tuyến mềm hay quay lệch góc 3D ngoài mặt phẳng.
* **Toán tử chuyển đổi:**

$$\begin{bmatrix} x' \\ y' \end{bmatrix} = s \begin{bmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{bmatrix} \begin{bmatrix} x \\ y \end{bmatrix} + \begin{bmatrix} t_x \\ t_y \end{bmatrix}$$

* **Phương pháp giải nghiệm đóng (Closed-form):**
* *Umeyama SVD:* Trích xuất các điểm đặc trưng hoàn toàn bên trong $\mathbf{M}_k$, khử $100\%$ nhiễu background và giải nghiệm đóng SVD trong vài micro-giây trên CPU.


* **Ánh xạ Storyboard:** Phát lệnh `M` (dịch chuyển), `S` (co giãn), `R` (góc quay).
* **Asset tiêu thụ:** **$0$ byte** (tái sử dụng Sprite gốc ở frame tham chiếu - Canonical Patch).

#### 2. Nhánh Xử lý Trắc quang & Bù sáng (Photometry & Blending)

Khi cấu trúc hình học phẳng đã được S2D ổn định (stabilized), phân bố pixel thay đổi do ánh sáng được xử lý qua 2 cơ chế:

* **Đổi tông màu & Cường độ đồng nhất (`Colour Tinting` - Lệnh `C`):**
* *Bản chất toán học:* Nghiệm đóng xác định (Closed-form parameter estimation). Vì lệnh `C` trong osu! là phép nhân làm tối màu ($c_k/255 \le 1.0$), frame chuẩn tắc được chọn là frame có độ sáng cực đại: $I_{\text{master}} = \max_t I_t$.
* *Phương pháp:* Giải bài toán bình phương tối thiểu có chặn bằng `scipy.optimize.lsq_linear`:



$$\min_{\mathbf{c}} \Vert{}I_{\text{master}} \cdot \frac{\mathbf{c}}{255} - I_t\Vert{}_2^2 \quad \text{sao cho } 0 \le c_k \le 255$$

* *Storyboard:* Xuất lệnh `C,0,tStart,tEnd,r1,g1,b1,r2,g2,b2` (**0 byte asset mới**).
* **Phát sáng & Cộng năng lượng (`Additive Blending` - Lệnh `P, ..., A`):**
* *Bản chất toán học:* Mô hình hóa vệt sáng trượt hoặc phát sáng tuyến tính: $\text{Observed} \approx \text{Base} + \alpha(t) \cdot \text{Light}$.
* *Kỹ thuật đóng nghiệm bằng Tiên nghiệm Khóa mốc (Anchor Priors):*
1. *Khóa Base (The Darkest Prior):* Lấy frame tối nhất làm nền tĩnh: $\text{Base}(x, y) = \min_t I(x, y, t)$.
2. *Khóa Light (Max Delta):* Lấy vệt sáng cực đại làm texture additive: $\Delta I_{\max}(x, y) = \max_t (I_t - \text{Base})$.
3. *Hồi quy 1 biến giải tích:* Biến số $\alpha(t)$ tại mỗi frame trở thành nghiệm đóng:





$$\alpha(t) = \frac{\sum_{(x,y)} (I_t - \text{Base}) \cdot \Delta I_{\max}}{\sum_{(x,y)} (\Delta I_{\max})^2}$$

* *Storyboard:* Xuất 1 file PNG nền tĩnh ($\text{Base}$) và 1 file PNG vệt sáng ($\Delta I_{\max}$), bật lệnh `P,0,tStart,tEnd,A`, gán keyframe lệnh `F` theo mảng $\alpha(t)$ và lệnh `M` bám theo chuyển động trôi của vệt sáng.
* **Cổng Kiểm soát Kỹ thuật (Fall-through Gate do Giới hạn Biểu diễn):**
Nếu gặp các trường hợp vượt khỏi khả năng biểu diễn của toán tử phẳng:
* Biến dạng không cứng (Non-rigid), góc quay 3D ngoài mặt phẳng ($SE(3) \setminus Sim(2)$) khiến S2D không hội tụ.
* Phản xạ quang học phi Lambertian phức tạp hoặc bóng đổ 3D (Self-shadowing: $\sum \max(0, \text{Base} - I_t)$ vượt ngưỡng).
* Vi phạm ngưỡng bảo toàn cấu trúc: $\text{PSNR} < 34\text{ dB}$ hoặc $\text{SSIM} < 0.92$.
* Tràn sắc (Clamping): $\frac{\text{Count}(\text{Base} + \Delta I > 255)}{\text{Total Pixels}} > 2\%$.


Compiler lập tức **dừng toàn bộ tối ưu S2D / trắc quang**, kích hoạt cơ chế Fall-through đẩy vùng biến thái xuống Phân tầng Frame.

#### 3. Nhánh Phân tầng Frame (Asset Generation do Giới hạn Hệ thống Đầu ra)

Khi vật thể quay 3D, đổi góc chiếu hoặc biến dạng mềm, việc liên tục tạo asset mới là điều hiển nhiên của hệ thống đồ họa 2D:

* **Sử dụng `Animation` Sprite (`anim_*`):**
* *Điều kiện kích hoạt:*
1. Nhịp biến đổi đồng tốc (Constant Pacing): $\sigma(\Delta t) \le 1.5\text{ ms}$ (không có gia tốc phi tuyến).
2. Tuần hoàn khép kín (`LoopForever`): Khoảng cách sai số $D(I_{N-1}, I_0) \le \text{Threshold}$ (run cycle, quạt quay, idle loop).
3. Chuỗi tuyến tính đơn kỳ (`LoopOnce`): Khói bốc, tia chớp diễn ra với tốc độ cố định từ $t_{\text{start}} \to t_{\text{end}}$.


* *Ràng buộc:* Tuyệt đối không vắt qua ranh giới cảnh (Shot Cut do `scenedetect` xác định).


* **Sử dụng Thay thế Sprite hoàn toàn (`Sprite Replacement`):**
* Kích hoạt khi: Biến động qua mốc chuyển cảnh, chuyển động góc 3D phi tuyến, có quãng nghỉ đứng yên (idle ngắt quãng), hoặc từng frame cần gán keyframe lệnh `R`, `S`, `C` riêng biệt. Tách thành các Sprite độc lập và điều khiển kích hoạt bằng lệnh `F` (Fade) hoặc switch tọa độ.



---

### Giai đoạn 3: Chiến lược Tách/Nhóm Trạng thái & Khôi phục Nền

Nhằm triệt tiêu việc chia nhỏ vụn vặt gây quá tải CPU/RAM cho engine osu!:

#### 1. Gom cụm Đồng nhất Chuyển động (Spatiotemporal Motion-Coherence Grouping)

* **Bản chất tối ưu:** Gom nhóm các điểm biến đổi không dựa trên ý nghĩa ngữ nghĩa (Semantic) mà dựa trên **sự tương đồng về vector tham số chuyển động và biến thiên màu sắc**:
* Các pixel hoặc vùng biến đổi lân cận có cùng vector biến đổi ($\Delta \vec{v} \le \epsilon$) sẽ được tự động **hợp nhất thành một Bounding Box chung duy nhất**.


* **Bộ tối ưu Hàm chi phí (Cost Function Optimizer):**
Trước khi quyết định cắt một vùng ra thành sprite con độc lập, compiler tính toán hàm phạt:

$$\text{Cost}_{\text{split}} = w_1 \cdot \text{AssetSize} + w_2 \cdot \text{CommandCount} + w_3 \cdot \text{DrawCallsOverdraw}$$

$$\text{Cost}_{\text{merge}} = w_1 \cdot \text{PaddedAssetSize}_{\text{parent}}$$

Nếu $\text{Cost}_{\text{split}} \ge \text{Cost}_{\text{merge}}$, hệ thống **bắt buộc phải giữ nguyên cụm lớn**, chấp nhận diện tích ảnh lớn hơn một chút nhưng triệt tiêu hàng nghìn dòng lệnh script và tránh làm nghẽn luồng xử lý đơn nhân của osu!.

* **Bộ lọc hình thái học (Morphological Open/Close):** Quét sạch các biến đổi diện tích cực nhỏ ($S < S_{\min}$ như nhiễu grain, bụi sáng 1x1). Các điểm này bị ép sáp nhập vào sprite nền hoặc sprite mẹ, **tuyệt đối cấm spam sprite 1x1 vô nghĩa**.

#### 2. Kiểm soát Vùng ít biến đổi theo Chuẩn Lossless Hoàn toàn

Kiểm tra ma trận sai số vi sai $\Delta(x, y) = \vert{}I_{t+1}(x, y) - I_t(x, y)\vert{}$:

* **Mức 1: Bit-Exact Lossless:** $\Delta(x, y) = 0 \ \forall (x, y) \in \mathbf{M}$. Kéo dài `tEnd` của sprite cũ (**0 byte asset**).
* **Mức 2: Screen-Quantization Lossless:** Dành cho nhiễu lượng tử hóa 8-bit hoặc sai lệch nén YUV $\to$ RGB:

$$\max_{(x,y)} \Delta(x, y) \le 1, \quad \frac{\text{Count}(\Delta > 0)}{\text{Total Pixels}} < 0.1\%, \quad \text{PSNR} \ge 48\text{ dB}$$



Thỏa mãn điều kiện: Thực hiện **Freeze Frame**, kéo dài lifetime trên timeline `.osb`.
* **Cắt Frame mới:** Nếu vượt ngưỡng sai số lượng tử hóa $\longrightarrow$ Cắt BBox mới và xuất PNG độc lập.

#### 3. Khôi phục Nền: Amodal Static vs. Dynamic Fallback

* **Nhánh Nền Tĩnh (Amodal Background Stitching):**
* Quét Lookahead Buffer ở các frame trước/sau khi foreground object đã di chuyển để lấy các pixel sạch đắp vào vùng bị che $\mathcal{O}(t) = \hat{\mathbf{M}}(t) \setminus V(t)$.
* Nền sau khi vá phẳng trở thành **1 file ảnh duy nhất không lỗ thủng**.
* **Native Z-Order:** Nền đặt ở Layer dưới, Foreground Object (đã khử sạch viền lem nhờ Giai đoạn 1) đặt ở Layer trên. GPU của osu! tự động che phủ tự nhiên, mép bán minh bạch hòa trộn hoàn hảo, không sinh vệt halo.


* **Nhánh Nền Động (Dynamic Background Fallback):**
* Khi nền có trường dòng quang học biến thiên liên tục ($\nabla \mathbf{I}_{\text{bg}} \neq 0$ do sóng nước, lá cây lay động, người đi xa hoặc chuyển động thị sai Parallax khi lia máy):
* Compiler tự động giáng cấp lớp nền thành **chuỗi Animation Sprite (`anim_*`)** hoặc cắt thành các mảng biến đổi động cục bộ, bảo đảm tính toàn vẹn thời gian của khung cảnh.



---

### Giai đoạn 4: Lưu trữ Định danh Bất biến & Khử trùng lặp Tài nguyên (CAS)

Quản lý dữ liệu texture độc lập với timestamp và resolution qua Content-Addressed Storage:

#### 1. Khử liên kết Không-Thời gian (Spatiotemporal Decoupling)

* **Two-Tier Memory:** Tier 1 (Working Memory SAM 2) bám đuổi frame ngắn hạn; Tier 2 (Long-Term Canonical Registry) lưu trữ fingerprint bền vững trên `index.db` để tra cứu hồi quy xuyên suốt video.
* **Bất biến FPS:** Tham số hóa chuyển động liên tục $\mathbf{P}(t)$. Thuật toán Ramer–Douglas–Peucker (RDP) triệt tiêu các keyframe trung gian nếu sai số nội suy tuyến tính $\text{Error} \le \text{Threshold}$, giảm tải tối đa số dòng lệnh script.

#### 2. Quy chuẩn Định danh Đa tầng (Content Hash Naming)

* **Đối với Sprite Đơn (Single Sprite):**

$$\mathbf{Asset\_Name} = \texttt{obj\_}\{\text{AspectBin}\}\texttt{\_}\{\text{pHash128}\}\texttt{\_}\{\text{ColorHash64}\}\texttt{.png}$$

* `AspectBin`: $\text{round}\left(\frac{W}{H} \times 100\right)$ $\to$ Lọc thô $O(1)$.
* `pHash128`: DCT Perceptual Hash kênh Alpha + Luminance trên lưới chuẩn $64 \times 64$ (dung sai Hamming $D_H \le 2$).
* `ColorHash64`: Histogram không gian màu LAB/HSV 16-bin để phân biệt vật liệu.
* **Đối với Chuỗi Diễn hoạt (Animation Sequence):**
Để bảo toàn thứ tự diễn hoạt và tuân thủ cú pháp `path_0.png`, `path_1.png` của osu!, chuỗi $N$ frame chuẩn hóa được định danh qua mã băm Merkle/BLAKE3:

$$H_{\text{seq}} = \text{BLAKE3}\Big( N \parallel \Delta t \parallel h_0 \parallel h_1 \parallel \dots \parallel h_{N-1} \Big)$$

$$\mathbf{Anim\_Prefix} = \texttt{anim\_}\{\text{AspectBin}\}\texttt{\_}\{\text{FrameCount}\}\texttt{\_}\{H_{\text{seq}}[0..15]\}$$

* Khi xuất đĩa: `anim_{Aspect}_{Count}_{Hash}_0.png`, `anim_{Aspect}_{Count}_{Hash}_1.png`,...
* Cú pháp lệnh gọi trong `.osb`:
```text
Animation,Foreground,Centre,"sb\anim_100_8_a1f9e0c4b82d.png",x,y,8,40,LoopForever

```



#### 3. Khắc phục Biến thiên Độ phân giải (Resolution Independence)

* **Tầng Định danh:** `pHash128` tính trên lưới chuẩn $64 \times 64$ nên video ở các độ phân giải khác nhau đều sinh ra fingerprint đồng nhất.
* **Max-Resolution Promotion:** Cache luôn ưu tiên lưu trữ phiên bản asset có độ phân giải vật lý cao nhất từng ghi nhận.
* **Bù trừ Scale động:** Tọa độ map về không gian logic osu! ($854 \times 480$). Giá trị tham số co giãn `S` trong `.osb` tự động tính theo tỷ lệ:

$$s = \frac{W_{\text{target\_osb}}}{W_{\text{png}}}$$

Đảm bảo hiển thị chuẩn xác $100\%$ không phụ thuộc kích thước file PNG trên đĩa.

### Giai đoạn 5: Tối ưu Biểu diễn Biến thiên bằng Easing & Curve Fitting

Sau khi đã xác định được trạng thái và tài nguyên cần biểu diễn, compiler tiếp tục tối ưu **cách biến thiên của từng tham số theo thời gian**. Thuật toán không cần biết tham số đó đại diện cho vị trí, scale, rotation, alpha hay màu sắc; nó chỉ quan tâm đến hàm biến thiên:

\(y = f(t), \qquad t \in [0,1]\)

**Mục tiêu:** tìm một representation có độ phức tạp thấp nhất nhưng vẫn tái tạo chính xác chuỗi giá trị đầu vào trong phạm vi yêu cầu Lossless.

#### 1. Chuẩn hóa & Phân tích Quỹ đạo

* Chuỗi giá trị được chuẩn hóa theo khoảng thời gian và biên độ về miền $[0,1]$.
* Compiler phân tích đặc tính biến thiên: đơn điệu, gia tốc/giảm tốc, điểm uốn, overshoot và các đoạn đứng yên.
* Không phụ thuộc semantic của dữ liệu: cùng một thuật toán có thể xử lý `X`, `Y`, `S`, `R`, `F`, `C` hoặc bất kỳ scalar parameter nào.

#### 2. Easing / Curve Fitting

Compiler thử các family easing có sẵn và các curve tham số hóa:

```text
Linear
Sine / Quad / Cubic / Quart / Quint
Expo / Circ
Back / Elastic / Bounce
Cubic Bézier
Custom Parametric Curve
```

Với mỗi candidate $f_i(t)$, hệ thống tính sai số giữa curve và chuỗi quan sát:

\(E_i = \sum_j w_j \left|y_j - f_i(t_j)\right|^2\)

Trong đó trọng số $w_j$ có thể tăng tại các điểm thay đổi mạnh hoặc các timestamp quan trọng của storyboard.

#### 3. Piecewise Curve Decomposition

Khi một easing đơn không thể biểu diễn toàn bộ quỹ đạo với sai số cho phép, compiler tự động tìm các breakpoint và chia thành nhiều segment:

```text
0 ────────────────┬────────────────── 1
                  │
            breakpoint

    easeOutQuad    │    easeInCubic
───────────────────┴────────────────────
```

Mỗi segment được fit độc lập. Thuật toán ưu tiên số segment nhỏ nhất trước khi giảm tiếp sai số.

#### 4. Model Selection

Không chọn candidate chỉ dựa trên sai số. Compiler tối ưu đồng thời **độ chính xác và độ phức tạp của representation**:

$$$\text{Cost}(M) =
\text{Error}(M)
+
\lambda \cdot \text{Complexity}(M)$$

Trong đó `Complexity` có thể bao gồm số segment, số parameter và số keyframe cần lưu.

* **Lossy mode:** chấp nhận sai số dưới một threshold xác định.
* **Lossless mode:** yêu cầu tái tạo chính xác tại toàn bộ sample/timestamp cần bảo toàn:

$$\text{Error}(M)=0$$

* Nếu không tồn tại một easing đơn phù hợp, compiler tăng số segment hoặc fallback về keyframe representation.

Như vậy, kết quả có thể chuyển từ hàng trăm keyframe rời rạc thành một representation nhỏ hơn:

```text
Keyframes:
  t0 → t1 → t2 → t3 → ... → tN

Optimized:
  [0.00 → 0.37] easeOutQuad
  [0.37 → 0.71] cubic-bezier(...)
  [0.71 → 1.00] easeOutExpo
```

#### 5. Hướng Triển khai

Kiến trúc triển khai được tách thành các module độc lập:

```text
Curve Model
  ├─ Linear
  ├─ Easing Functions
  ├─ Cubic Bézier
  └─ Custom Parametric Curves

Curve Fitter
  └─ Parameter Optimization / Least Squares

Segmenter
  └─ Breakpoint Search / Piecewise Decomposition

Model Selector
  └─ Error + Complexity Optimization

Storyboard Encoder
  └─ Chuyển representation tối ưu thành M/S/R/F/C...
```

Giai đoạn đầu có thể tận dụng các implementation easing và curve-fitting hiện có thay vì tự xây toàn bộ từ đầu. Hiện đã có các thư viện/implementation cung cấp easing functions và cubic-Bézier fitting, điển hình như `robertpenner/easing` với `fitCubicBezier`, cùng các implementation dựa trên thuật toán curve fitting của Philip J. Schneider.

Tuy nhiên, các implementation này chủ yếu giải quyết **một phần bài toán** (evaluation hoặc fitting). Chưa có một thư viện .NET rõ ràng bao trọn pipeline **automatic easing selection → piecewise segmentation → complexity-aware model selection → lossless storyboard representation**. Vì vậy, phần optimizer và representation selection nên được xây dựng riêng trong Aurbey, còn các easing/curve-fitting primitive có thể được tái sử dụng.
$$$
