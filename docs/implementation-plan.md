# OsbMpeg: kế hoạch triển khai library-first (Phase A → M0 → M0.5 → M1)

Trạng thái: đã được người dùng duyệt định hướng ngày 2026-09-28 (mục 2). Đây là spec thi công dành
cho agent triển khai. Mục tiêu là agent đọc xong có thể làm end-to-end mà không cần hỏi lại về định
hướng.

---

## 0. Cách dùng tài liệu này

**Thứ tự ưu tiên nguồn.** Tài liệu này là nguồn cho *xây gì*: mọi quyết định đã chốt (mục 2, D1 đến D18),
kiến trúc và invariants (mục 4), từng bước từ Phase A tới M1 (mục 8) và lộ trình sau M1 (mục 12).
`docs/direction.md` là nguồn cho *lý do*: chẩn đoán, sự thật về osu!, không gian representation, bảng
bác bỏ (mục 8 của nó), prior art (mục 10) và lịch sử đo (mục 11). Hai tài liệu không override nhau:
phát hiện mâu thuẫn thì sửa tài liệu sai, không thêm bảng ngoại lệ. `docs/feasibility-plan.md` mô tả
spike kiểm chứng chạy trước Phase A.

**Mỗi bước** trong mục 8 ghi rõ: mục tiêu, file, API, thuật toán, test, acceptance, và những gì
không được làm. Mỗi bước kết thúc bằng: build Release sạch, toàn bộ test xanh, đúng một commit theo
Conventional Commits. Không gộp nhiều bước vào một commit, không bắt đầu bước sau khi bước trước chưa
xanh.

**Điểm dừng chờ người** chỉ có hai chỗ: kết thúc M0.5 (mục 8.3, người dùng chốt số preset sau khi xem
ảnh) và tripwire hiệu năng hoặc kích thước ở M1 (mục 8.4, acceptance 7 và 8). Ngoài hai chỗ đó, agent
tự quyết trong phạm vi spec. Gặp điều spec không nói và không suy ra được từ mục 2 và mục 4.5: dừng lại
và hỏi người dùng, không tự phát minh kiến trúc.

**Lệnh build và test** (sau Phase A, bước A2):

```text
dotnet build -c Release
cd tests/OsbMpeg.Tests/bin/Release/net10.0 && ./OsbMpeg.Tests.exe
./OsbMpeg.Tests.exe -method "*TenTest*"          # lọc một test
OSBMPEG_FIXTURES=1 ./OsbMpeg.Tests.exe -trait "Category=Fixture"   # harness nặng, xem mục 9
```

Không dùng `dotnet test` (runner Microsoft.Testing.Platform trong `dotnet.config` không chạy được trên
SDK này, xem `CLAUDE.md`).

**Quy ước code.** Khớp văn phong code xung quanh: doc comment dài giải thích *vì sao*, ghi chú
`ponytail:` cho mọi chỗ cố ý đơn giản hóa có trần đã biết. Identifier tiếng Anh. Nullable bật. Không
thêm dependency ngoài danh sách ở bước A2 và A5.

**Nguồn sự thật bên ngoài.** Hành vi của osu!lazer và API của OsuParsers ghi trong tài liệu này được
tra qua deepwiki (index nhánh main trên GitHub, không phải bản NuGet 1.7.2). Mục nào đánh dấu
`[deepwiki]` là chưa được xác nhận bằng client thật hoặc bằng compile. Bước A0 xác nhận phần
OsuParsers bằng compile. Muốn tra thêm thư viện thì dùng context7 hoặc deepwiki, không dịch ngược DLL.

---

## 1. Mục tiêu cuối và phạm vi

OsbMpeg trở thành **một thư viện .NET 10, một assembly `OsbMpeg`**. Nó biến **một video** thành tập
object storyboard osu! biểu diễn bằng object model của OsuParsers, cộng các asset PNG. Kết quả phải đạt
bốn điều:

1. **Chất lượng:** mọi frame, tại fps yêu cầu, đạt sàn tuyệt đối của preset. Điều này được kiểm chứng
   độc lập với encoder (mục 7).
2. **Cache asset theo nội dung:** tên asset chỉ phụ thuộc pixel, nằm trong một thư mục dành riêng do
   caller chỉ định, dùng lại được xuyên lần gọi, xuyên video và xuyên tiến trình.
3. **An toàn đồng thời:** nhiều compile song song trong một tiến trình, hoặc nhiều tiến trình OsbMpeg
   dùng chung một thư mục asset, không bao giờ ghi đè nhau và không đọc phải file dở.
4. **Không phụ thuộc nơi lưu:** `.osb` ghi qua `TextWriter`, asset đi qua `IAssetStorage`. Trong
   memory chỉ giữ metadata; pixel được nạp lười và có giới hạn.

**Không còn** CLI, format `.osbv`, auto-tuner, scene pre-pass.

**Hết tài liệu này** (xong M1), encoder là closed loop với ba provider: Patch, Photometric (lệnh `C`)
và Crossfade. Verifier chạy mặc định sau mỗi compile.

**Ngoài phạm vi.** Các mục sau chỉ có concept ở mục 12: motion (M2), animation và thrash, thực thi ngân
sách workload, encoding asset lossy, attribution, dataset, nhiều video dùng chung decode, nguồn frame
tùy biến, lookahead.

---

## 2. Quyết định đã chốt (người dùng, 2026-09-28 và 2026-10-29) và hệ quả bắt buộc

| ID | Quyết định | Hệ quả thi công |
| --- | --- | --- |
| D1 | osu!lazer là chuẩn verify, stable ở mức best-effort | Không phát bất cứ thứ gì đã biết là lazer đọc sai. Value-chaining bị REJECT. `S` và `V` là hai property riêng (lazer nhân `Scale × VectorScale` [deepwiki]). Thứ chỉ rủi ro trên stable thì cảnh báo, không fail. |
| D2 | Sàn chất lượng cứng là mặc định; library trả report, không throw khi vi phạm | `VerificationReport.Passed = false` là kết quả hợp lệ, không phải exception. |
| D3 | Public gồm: facade compile, API encode một video trả object để caller chèn vào storyboard của họ, renderer, verifier. Encoder core để internal. | Xem mục 4.2. `ICandidateProvider` để internal, chưa mở plugin. |
| D4 | Bỏ `.osbv`. Method chính chỉ xử lý **một** `VideoAnimation`. Command lồng trong video dùng object model của OsuParsers. Cache nằm ngay trong thư mục asset, chỉ theo nội dung. Caller chỉ định thư mục asset riêng để không lẫn với asset storyboard có sẵn. | Ranh giới public (vào và ra) là OsuParsers. IR nội bộ (`SbDocument`, …) chuyển thành `internal`. Đây là cách hòa giải D3 với D4: "IR public" ở D3 được hiểu là object model OsuParsers. OsuParsers object không mang layer, nên result phải mang `Layer` và có `AddTo(Storyboard)`. |
| D5 | Trừu tượng hóa nơi lưu tối đa. `.osb` ghi qua `StreamWriter`/`TextWriter`. Asset quản lý kiểu database: index trong memory, nội dung trên storage, pixel nạp lười hoặc async, không nạp dồn. | `IAssetStorage` là public (mục 5). `AssetIndex` chỉ giữ metadata. Renderer và verifier nạp pixel lười với cache LRU có trần theo byte. |
| D6 | Chỉ .NET 10 | `net10.0`, không multi-target. |
| D7 | `Fps` = số frame chương trình xử lý và biểu diễn mỗi giây. Ép video về fps đó rồi xử lý tiếp; frame bị loại được frame trước đại diện. | Resample bằng filter `fps=` của ffmpeg (đã có trong `FrameSource`). `null` nghĩa là fps nguồn. Fps lớn hơn nguồn chỉ tốn thời gian, không đổi output, nên không cần xử lý riêng. |
| D8 | Không CLI; chỉ core engine | Xóa `OsbMpeg.Cli` và test của nó. Harness đo nằm trong test project (mục 9). |
| D9 | Được đổi format, nhưng quy tắc đặt tên asset phải nhất quán và **chỉ phụ thuộc nội dung pixel**. Tối ưu truy vấn, không tạo file hoặc folder vô nghĩa. | Key = XXH3-128 trên (tag phiên bản, format pixel, kích thước, pixel cuối cùng được lưu). Bỏ seed `colors`. Animation lưu phẳng, không mỗi animation một folder. Không file index, không file phụ. |
| D10 | Plan chi tiết cho phần trước mắt; phần chưa biết chỉ nêu concept và khả thi | Mục 8 chi tiết (Phase A, M0, M0.5, M1). Mục 12 là spec mức bước cho phần sau M1, chi tiết file và test viết khi tới lượt. |
| D11 | Nhiều tiến trình OsbMpeg chạy song song trên cùng thư mục asset phải an toàn | Hợp đồng "tạo nếu chưa có" nguyên tử (mục 5.1). Ghi file tạm rồi rename, không bao giờ ghi đè. |
| D12 | **Không có khái niệm object hay thực thể ngữ nghĩa.** Đơn vị làm việc là *thay đổi liên kết* (coherent change): vùng không-thời gian lớn nhất mà một primitive trong ladder giải thích được trong sàn. "Sprite" là hệ quả của việc một track Sim(2) giải thích được một vùng, không phải đầu vào. | Không có kênh prompt, không có bước "định danh object". Layer được *đề xuất* bằng tín hiệu motion (block flow, sequential RANSAC Sim(2), liên thông không gian, prior thời gian) và được *nhận* chỉ khi closed loop đo thấy rẻ hơn. Re-identification xuyên che khuất và xuyên scene là canonical hash của asset (I10), không phải ID của tracker. |
| D13 | **Lõi là .NET thuần; model học chỉ ở sidecar tùy chọn.** SAM 2/3, MatAnyone hoặc tương đương chạy trong tiến trình Python riêng, chỉ để *tinh chỉnh biên* (alpha matte) của layer đã được đề xuất bằng motion. Không có sidecar thì pipeline vẫn chạy đầy đủ. | Contract `ILayerBoundaryRefiner` (mục 12.6, sidecar ở 12.9). Sidecar chỉ được xây sau khi attribution (M3.5) chỉ ra bucket "mép layer" đủ lớn. Không thêm ONNX hay PyTorch vào assembly `OsbMpeg`. |
| D14 | **Ưu tiên nội dung được author**: motion graphics, UI, 2D animation. Game và footage vẫn phải không tệ đi (verify pass, bytes không tăng), nhưng không là mục tiêu tối ưu trong 6 tháng tới. | Thêm tier fixture `authored` (mục 9.2 và `tests/fixtures/README.md`). R1 (dataset có ground truth chương trình) đi ngay sau M1, R2 (easing rời rạc) ngay sau M2, motion layer K>1 là M2.5 với keep-gate đo trên tier `authored` (mục 12.1). |
| D15 | **Thứ tự: closed loop trước, provider sau.** Phase A → M0 → M0.5 → M1 giữ nguyên. Không spike SAM hay Sim(2) trên code v1. | Mọi số đo về layer, easing, additive đều phải là số đo *sau* verify. Spike về layer, Sim(2), easing và additive (mục 12.5, 12.6) chạy trên nền M1 và M2, không sớm hơn. |
| D16 | **Các hướng đã bác bỏ có bằng chứng, không đề xuất lại** (bảng quyết định, `direction.md` mục 8): tracking ngữ nghĩa làm xương sống; freeze gate so với frame trước (vòng hở, chính là lỗi mờ của v1); nền động hay vùng lớn bằng `Animation` (R4); cubic Bézier và curve tham số tự do (không có trong `.osb`); dedupe theo pHash hay fuzzy không verify pixel (pop); cost function dự đoán với trọng số tay (đo thật thua 2,6 đến 5,4 lần); value-chaining và `[Variables]`. | Không viết code cho các mục này; muốn đảo thì phải có số đo nêu trong cột "điều gì đảo nó" của bảng đó. Những kỹ thuật được giữ cho M2.5: Sim(2) thay affine; `C` đóng dạng với master là frame sáng nhất; additive layer `Base + α(t)·Light` với base theo percentile, chỉ trong hệ sprite đã bù chuyển động; easing rời rạc + piecewise + chọn theo error + λ·complexity; canonical asset + instance; nền dựng từ pixel thật đã thấy (tích lũy theo mask, không ảo giác), pixel chưa từng thấy được lấp bằng khuếch tán và không bao giờ được tin (mục 12.6). |

| D17 | **Vật thể di chuyển làm lộ nền được xử lý theo hai nấc.** Nấc a (M2.5a): theo dõi vùng của X bằng quỹ đạo Sim(2), biểu diễn X bằng *một* sprite cắt theo bounding rect kèm track, chạy nhân quả; nền vừa lộ ra *không* được dựng riêng mà để closed loop vá bằng patch như mọi vùng bẩn khác, và patch đó được chèn **dưới** X. Nấc b (M2.5b): mask và alpha cho X, dựng nền nhìn trước cả đoạn, khử lem màu ở mép; chỉ xây khi số đo của nấc a cho thấy bucket `rect-halo` hoặc `reveal-strip` đủ lớn. | Không có pass tích lũy, không dựng nền, không alpha ở M2.5a. Cần ba thay đổi nền tảng ở M2 (S2.R, S2.G) và chèn-dưới trong `ReconstructionState`/`EmitContext` (S1.2). Gate theo đoạn bằng kiểm thử khô; bị từ chối thì output trùng byte với bước trước. |
| D18 | **Solver (CP-SAT, ILP) là thước đo trước khi là công cụ.** Dùng làm oracle offline cho R3 để đo ladder greedy bỏ phí bao nhiêu; chỉ được vào lõi ở bài toán ngân sách workload M3 (multiple-choice knapsack theo lát thời gian) khi oracle hoặc số vi phạm ngân sách đòi. Không dùng làm bộ chọn representation toàn cục trong lúc encode. | `Google.OrTools` chỉ được thêm vào project đo hoặc test, không vào assembly `OsbMpeg` cho tới khi S3.1b được duyệt. Các bài toán con có cấu trúc tốt hơn giữ thuật toán chuyên dụng: chia đoạn easing dùng DP, z-order K ≤ 4 duyệt hết, gán block vào layer dùng graph cut nếu cần, span crossfade dùng greedy. |
---

## 3. Quan hệ với `docs/direction.md`

Không có override. `direction.md` đã được viết lại để khớp mọi quyết định ở mục 2: không CLI, không
`.osbv`, không scene pre-pass, không shared decode; `S` và `V` là hai property riêng; value-chaining và
`[Variables]` bị bác; JPEG là INVESTIGATE; photometric chỉ dùng `C`; encoder composite trên display model
riêng còn verifier render IR; mọi A/B so với tag `v1-baseline`. Những gì tài liệu này **lấy nguyên** từ
đó: cơ chế ladder, headroom và hysteresis, preset hai trục (PSNR mỗi frame, worst-tile SSIM), M0.5 là
điểm dừng chờ người, bảng bác bỏ (mục 8 của nó), và lịch sử đo (mục 11 của nó, nơi mọi acceptance ghi
số).

---

## 4. Kiến trúc đích

### 4.1 Bố cục solution sau Phase A

```text
OsbMpeg.slnx
src/OsbMpeg/OsbMpeg.csproj              net10.0, một assembly duy nhất
  *.cs (root, namespace OsbMpeg)        PUBLIC API (mục 4.2)
  Storage/                              public: IAssetStorage, FileSystemAssetStorage
                                        internal: AssetIndex, AssetKey, AssetEntry
  Ir/                                   internal: SbDocument, SbObject, SbCommand…, CanvasMapping,
                                        LoopFlattener, Utilities (IsEqual)
  Ir/Passes/                            internal: MergeAdjacentCommands, DropNoOpCommands, LoopExtractor
  Osb/                                  public: StoryboardWriter
                                        internal: OsbWriter, OsuParsersConverter, StoryboardDecoderGate
  Rendering/                            public: StoryboardRenderer (M0)
                                        internal: CommandEvaluator, EasingTable, Compositor, Canvas,
                                        SoftwareStoryboardRenderer, AssetPixelCache (M0)
  Media/                                internal: FrameSource, MediaProbe, MediaInfo, VideoFrame
  Compilation/                          internal: GroupTransformBaker, CompilePipeline
                                        (Phase A: LegacyTileEncoder và các file Analysis cũ)
  Encoder/                              internal (M1): ClosedLoopEncoder, ReconstructionState,
                                        DisplayItem, BlockOwnership, FrameWindow, EmitContext,
                                        Candidates/*
  Verification/                         internal: Metrics, ErrorMap, DirtyRects, QualityVerifier,
                                        WorkloadAnalyzer, PropertyOverlapValidator, CalibrationSimulator
tests/OsbMpeg.Tests/OsbMpeg.Tests.csproj   gộp Parsers.Tests và Compiler.Tests
tests/fixtures/                         mp4 có sẵn, cộng synthetic ở S1.14
```

Không đặt namespace tên `OsbMpeg.Encoding`, vì nó che `System.Text.Encoding` trong mọi file thuộc
namespace `OsbMpeg.*`. Dùng `OsbMpeg.Encoder`.

### 4.2 Public API (đích sau M1; cột "Bước" ghi bước đưa vào)

```csharp
namespace OsbMpeg;

// Bước A5. Một video cần compile. Tương đương "Compile(video, at, from, to, ...)".
public sealed record VideoAnimation
{
    public required string VideoPath { get; init; }        // đường dẫn tương đối resolve theo CWD, giống File API của .NET
    public required int StartTime { get; init; }           // "at": mốc ms trên storyboard mà frame tại From xuất hiện
    public TimeSpan From { get; init; } = TimeSpan.Zero;   // điểm bắt đầu trong video
    public TimeSpan? To { get; init; }                     // null nghĩa là tới hết video
    public double? Fps { get; init; }                      // null nghĩa là avg fps của nguồn (D7)
    public StoryboardLayer Layer { get; init; } = StoryboardLayer.Background;
    public Vector2 Position { get; init; } = new(320, 240); // tâm video trong storyboard space, auto-cover; cũng là pivot của group transform
    public CommandGroup? Commands { get; init; }           // group transform (OsuParsers), thời gian tuyệt đối trên storyboard; null = không có
}

// Bước A5
public sealed record CompileOptions
{
    public required IAssetStorage Storage { get; init; }    // gốc = thư mục beatmap (nơi osu! resolve đường dẫn)
    public required string AssetDirectory { get; init; }    // ví dụ "sb/osbmpeg"; dành riêng cho OsbMpeg; ghi nguyên văn vào .osb
    public QualityPreset Quality { get; init; } = QualityPreset.High;
    public bool Verify { get; init; } = true;               // bước S0.6
    public FfmpegSettings Ffmpeg { get; init; } = new();
}

public sealed record FfmpegSettings(string? BinaryFolder = null, string? HardwareAcceleration = null);

// Bước A5 (số giữ chỗ), số thật chốt ở M0.5
public sealed record QualityPreset(double PsnrFloorDb, double TileSsimFloor, double HeadroomDb = 2.0,
    double TileSsimHeadroom = 0.02, int HysteresisFrames = 2)
{
    public static QualityPreset High { get; }
    public static QualityPreset Medium { get; }
    public static QualityPreset Low { get; }
}

// Bước A5. Thread-safe: nhiều CompileAsync chạy đồng thời trên cùng một instance là hợp lệ.
public sealed class OsbMpegCompiler
{
    public OsbMpegCompiler(CompileOptions options, ILogger? logger = null);   // validate options ngay (mục 5.7)
    public Task<VideoCompileResult> CompileAsync(VideoAnimation video,
        IProgress<CompileProgress>? progress = null, CancellationToken cancellationToken = default);
    public Task<VerificationReport> VerifyAsync(VideoAnimation video, IReadOnlyList<IStoryboardObject> objects,
        IProgress<CompileProgress>? progress = null, CancellationToken cancellationToken = default);  // bước S0.6
}

// Bước A5 (Workload thêm ở S0.5, Verification thêm ở S0.6)
public sealed record VideoCompileResult(
    StoryboardLayer Layer,
    IReadOnlyList<IStoryboardObject> Objects,   // thứ tự = thứ tự khai báo = z-order trong layer
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

// Bước S0.6
public sealed record VerificationReport(bool Passed, int FrameCount,
    double MinPsnr, double P1Psnr, double MeanPsnr,
    double MinTileSsim, double P1TileSsim, double MeanTileSsim,
    int ViolationCount, IReadOnlyList<QualityViolation> Violations);   // Violations giữ tối đa 1000 bản ghi đầu
public sealed record QualityViolation(int FrameIndex, double StoryboardTimeMs, double Psnr, double TileSsim,
    int WorstTileX, int WorstTileY);

// Bước S0.5
public sealed record WorkloadReport(double PeakSteadySbLoad, double PeakTransientSbLoad, int PeakAliveSprites,
    long AnimationResidentBytes, long TotalAssetBytes, int OsbLineCount,
    IReadOnlyList<WorkloadSample> Timeline);   // mẫu mỗi 100 ms
public readonly record struct WorkloadSample(double TimeMs, int AliveSprites, double SteadySbLoad, double TransientSbLoad);

public class OsbMpegException : Exception { /* ctor chuẩn */ }
public sealed class VideoDecodeException : OsbMpegException { /* ctor chuẩn */ }

namespace OsbMpeg.Osb;
public static class StoryboardWriter    // bước A4
{
    // Ghi 5 layer hình và SamplesLayer. Throw NotSupportedException nếu storyboard.Variables khác rỗng:
    // không âm thầm làm mất dữ liệu của caller, và R9 cấm tự phát [Variables].
    public static void Write(Storyboard storyboard, TextWriter writer);
}

namespace OsbMpeg.Rendering;
public sealed class StoryboardRenderer  // bước S0.4
{
    public StoryboardRenderer(Storyboard storyboard, IAssetStorage storage, int width, int height, bool passing = true);
    public void Render(double timeMs, Span<byte> rgb24);    // rgb24.Length == width*height*3
}

namespace OsbMpeg.Storage;
public interface IAssetStorage { /* mục 5.1 */ }
public sealed class FileSystemAssetStorage : IAssetStorage { public FileSystemAssetStorage(string rootDirectory); }
```

`ILogger` lấy từ `Microsoft.Extensions.Logging.Abstractions` (bản 10.0.x ổn định mới nhất), `null` thì
dùng `NullLogger`. Ngoài danh sách trên, mọi type khác là `internal`. Bước A7 khóa surface bằng
snapshot test.

### 4.3 Luồng của một lần `CompileAsync` (sau M1)

```text
validate VideoAnimation (mục 10.1)
→ MediaProbe (FFOptions theo lần gọi) → fpsEff, From, To, W, H                    [Probing]
→ group commands: OsuParsers CommandGroup → IR → LoopFlattener → sort
    → PropertyOverlapValidator (có lỗi thì ArgumentException)
    → quyết disjoint mode (S1.8)
→ FrameSource.ReadFramesAsync(W×H, fpsEff, From, To-From)                          [Encoding]
    → ClosedLoopEncoder: frame → FrameWindow → AdvanceTo → ErrorMap → DirtyRects → ladder
      → Commit (EmitContext, AssetIndex) → Apply
→ EmitContext.FinalizeAll → IR (baker nếu có group commands) → passes (Merge, DropNoOp, LoopExtractor)
→ PropertyOverlapValidator (có lỗi thì OsbMpegException, vì đó là bug nội bộ)
→ OsuParsersConverter.ToOsuParsers (làm tròn thời gian, mục 6) → Objects
→ WorkloadAnalyzer
→ nếu Verify: QualityVerifier (mục 7), decode lần hai                               [Verifying]
→ VideoCompileResult
```

### 4.4 Ai sở hữu state nào

| State | Chủ sở hữu | Vòng đời | Ghi chú |
| --- | --- | --- | --- |
| `AssetIndex` (metadata của asset) | `OsbMpegCompiler` instance | đời instance | dùng chung giữa các compile đồng thời; không giữ pixel |
| File asset | `IAssetStorage` | vĩnh viễn | library không bao giờ xóa hay ghi đè |
| `ReconstructionState`, `FrameWindow`, `EmitContext`, `BlockOwnership` | một lần `CompileAsync` | một video | không chia sẻ |
| IR nội bộ | `CompilePipeline` | một lần gọi | không lộ ra ngoài |
| `IReadOnlyList<IStoryboardObject>` | caller | sau khi trả về | library không giữ tham chiếu |
| Cache pixel của renderer | một renderer instance | một lần render hoặc verify | LRU có trần byte |
| Lock `StoryboardDecoderGate` | static toàn process | process | state toàn cục duy nhất được phép (OsuParsers không reentrant) |

### 4.5 Invariants

Mọi code, hiện tại và tương lai, phải giữ. Vi phạm là bug kiến trúc, không phải lựa chọn phong cách.

- **I1.** IR nội bộ (`SbDocument`, `SbObject`, `SbCommand`) là ngôn ngữ chung duy nhất *bên trong*.
  Mọi representation, hiện có hay thêm sau, phát ra `SbSprite` hoặc `SbAnimation` cùng command chuẩn
  vào `SbDocument`; không có kênh phụ, không có "gợi ý cho renderer" nằm ngoài IR. Ranh giới public là
  OsuParsers, và chỉ có đúng một converter hai chiều (`OsuParsersConverter`).
- **I2.** Verification độc lập là thẩm quyền cuối. Output đi qua vòng text (`StoryboardWriter`, rồi
  `StoryboardDecoder` của OsuParsers, rồi IR) trước khi render bằng renderer IR; verifier không dùng
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
  frame tới và vá chỗ sai (S2.G). Không frame nào được ship mà chưa qua pixel check.
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
- **I10.** Dedupe duy nhất là key nội dung của `AssetIndex` (mục 5.3). Mở rộng (chuẩn hóa pose, biến thể
  encoding) phải là mở rộng của key, không phải một cơ chế song song.
- **I11.** Library không bao giờ xóa hay ghi đè một asset đã tồn tại. Ngoại lệ duy nhất là file tạm của
  chính nó đã quá hạn (mục 5.2).
- **I12.** Không có state toàn cục thay đổi được, trừ lock của OsuParsers. Cấu hình ffmpeg đi theo từng
  lần gọi, không dùng `GlobalFFOptions.Configure`.
- **I13.** Public surface bị khóa bằng snapshot test. Muốn thêm type hay member public thì phải sửa file
  approved một cách có chủ đích.
- **I14 (phủ kín).** Từ `StartTime` tới hết video, mọi pixel trong rect của video phải được phủ bởi ít
  nhất một item opaque (alpha nội dung bằng 1). Trong library, video nằm trên storyboard của caller:
  chỗ nào không được phủ thì layer bên dưới hoặc background beatmap sẽ lộ ra. Những thứ tồn tại để giữ
  I14:
  - I-frame ở frame 0 và quy tắc cut (S1.8). Agent **không được** "tối ưu" I-frame bằng cách bỏ qua các
    block trông như đã sạch.
  - `BlockOwnership` (S1.3) chỉ đóng item khi block của nó đã có chủ mới.
  - Nền của `ReconstructionState` và của verifier là xám `(128,128,128)`, không phải đen. Nhờ vậy lỗ
    hổng trên vùng nguồn màu đen (letterbox, cảnh tối) trở thành lỗi đo được.

  Có test cấu trúc: sau frame 0, tại mọi frame, mọi block đều có chủ, kể cả với input có letterbox.
- **I15.** Lõi `OsbMpeg` không phụ thuộc model học, PyTorch hay ONNX (D13). Thứ cần model chạy trong
  tiến trình riêng và giao tiếp bằng file mask và alpha qua một contract internal; thiếu nó, pipeline
  vẫn chạy đầy đủ với provider classical.

### 4.6 Kiểm tra ngược: kiến trúc có mở không

Câu hỏi kiểm tra: tuần sau phát hiện một representation mới tốt hơn mọi cái hiện có, có thêm được như
một candidate mà không phải viết lại `ClosedLoopEncoder`, `ReconstructionState` hay verifier không? Có,
vì bốn lý do. Lõi encoder chỉ biết `ICandidateProvider` và `IRepresentationCandidate`, nên thêm nghĩa là
thêm class rồi đăng ký vào danh sách provider kèm rank. `ReconstructionState` composite từ *object IR*
trong display list chứ không từ "loại representation", nên nó render được candidate mới mà không cần
biết đó là gì; điều kiện duy nhất là primitive động mới nằm trong tập command mà `CommandEvaluator`
đánh giá được, và tập đó chính là toàn bộ command space của `.osb`. Verifier render từ output đã qua
vòng text nên theo định nghĩa không biết representation nào sinh ra nó. Giả thuyết phân rã mới (layer
set, panorama, sau này là bất cứ gì) chỉ là một analyzer mới ghi thêm vào `SceneMotionInfo` cộng một
provider mới tiêu thụ nó.

---

## 5. Asset storage và cache

### 5.1 Hợp đồng `IAssetStorage`

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

### 5.2 `FileSystemAssetStorage`: thuật toán

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

### 5.3 Key và đường dẫn asset (D9)

- **Key** = XXH3-128 (`System.IO.Hashing.XxHash128`, seed 0) trên chuỗi byte sau:
  `"OSBMPEG-ASSET-V1"` (ASCII) ‖ kind (1 byte: 0 = sprite, 1 = animation) ‖ pixel format (1 byte:
  0 = Rgb24) ‖ width (int32 LE) ‖ height (int32 LE) ‖ frameCount (int32 LE, sprite = 1) ‖ pixel của từng
  frame.
- **Pixel được hash là pixel cuối cùng mà file lưu.** M1 chỉ có PNG lossless không quantize, nên đó
  chính là pixel crop từ frame. Khi sau này có encoding lossy (mục 12), key là hash của pixel sau
  lossy, nên vẫn chỉ phụ thuộc nội dung.
- Kích thước nằm trong key để sửa bug hiện có: `AssetStore.GetOrAdd` hash mỗi byte pixel, nên hai crop
  đơn sắc khác kích thước (ví dụ 10×20 và 20×10 cùng màu đen) trùng key và dùng nhầm file.
- Tag phiên bản `V1` là hằng số cho mọi video. Đổi cách encode trong tương lai thì tăng `V2`; cache cũ
  không bị dùng nhầm.
- **Sprite:** `{AssetDirectory}/s/{key:x32}.png`.
- **Animation** (chỉ concept, M3): `{AssetDirectory}/a/{key:x32}.png` là base path ghi vào `.osb`, còn
  frame i nằm ở `{AssetDirectory}/a/{key:x32}{i}.png`. osu! suy frame bằng `Path.Replace(".", $"{i}.")`
  trên *toàn* đường dẫn; key hex có độ dài cố định 32 ký tự, nên dạng phẳng không va chạm. Không tạo
  folder riêng cho mỗi animation. Điều kiện là `AssetDirectory` không chứa dấu `.` (mục 5.7). Một
  animation được coi là tồn tại khi *mọi* frame tồn tại; frame nào thiếu (tiến trình chết giữa chừng) thì
  ghi bù frame đó.
- Không tạo file index hay manifest: storage *chính là* index, vì tên file suy trực tiếp từ key. Truy
  vấn là một `GetLengthAsync`, O(1) trên NTFS.
- Asset theo layout cũ (`s/{hash}.png` seed bằng `colors`) không được dùng lại và không bị xóa (I11).
  Người dùng tự xóa thư mục cũ nếu muốn.

### 5.4 `AssetIndex` (trong tiến trình)

```csharp
internal sealed class AssetIndex(IAssetStorage storage, string assetDirectory)
{
    // Trả entry cùng cờ Created: true nghĩa là CHÍNH lần gọi này đã publish file (tính vào AssetsCreated).
    Task<(AssetEntry Entry, bool Created)> GetOrCreateSpriteAsync(ReadOnlyMemory<byte> rgb24, int width, int height, CancellationToken ct);
    Task<(AssetEntry Entry, bool Created)> GetOrCreateAnimationAsync(IReadOnlyList<ReadOnlyMemory<byte>> frames, int width, int height, CancellationToken ct); // M3, và Phase A cho legacy
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

### 5.5 Ma trận đồng thời

| Tình huống | Hành vi đúng | Cơ chế |
| --- | --- | --- |
| Hai `CompileAsync` đồng thời trên một `OsbMpegCompiler`, trùng nội dung | Encode và ghi đúng một lần | `Lazy<Task>` trong `AssetIndex` |
| Hai tiến trình (hoặc hai instance) cùng ghi một key | Một bên publish; bên kia nhận `false`, file không bị ghi đè | Temp rồi `File.Move(overwrite:false)` |
| Tiến trình B đọc đúng lúc A đang ghi | B thấy "chưa có" hoặc thấy file đủ, không bao giờ thấy file dở | Chỉ tên cuối cùng mới được đọc; rename là nguyên tử trên cùng volume |
| Tiến trình chết giữa lúc ghi | Chỉ còn lại file tạm mồ côi | Dọn sau 24 giờ (mục 5.2) |
| Cancel giữa compile | Không để lại file tạm, không để lại file dở | `finally` xóa tmp |
| File asset bị xóa từ bên ngoài giữa chừng | Index vẫn tưởng file còn (stale) | Verifier đọc qua storage nên lộ lỗi ngay; ghi rõ trong doc comment. Không tự sửa. |
| Antivirus giữ file mới tạo | Retry có backoff | Mục 5.2, bước 5 |

### 5.6 Quy tắc bộ nhớ

- `AssetIndex` chỉ giữ metadata.
- `ReconstructionState` giữ pixel của các `DisplayItem` *còn sống*. Pixel được copy từ frame lúc
  commit; item đóng quá một cửa sổ thì giải phóng.
- `FrameWindow` có trần theo byte, mặc định `min(2 s, 512 MiB)`, tối thiểu 3 frame.
- Renderer và verifier nạp asset lười qua `IAssetStorage.OpenReadAsync` rồi decode ImageSharp, với cache
  LRU trần theo byte decode, mặc định 256 MiB. Asset được nạp khi object lần đầu cần vẽ và bị loại theo
  LRU. Cache vô hạn `_assetCache` hiện tại phải bị thay.
- Không API nào nạp toàn bộ asset của một video vào memory cùng lúc.

### 5.7 Validate `AssetDirectory`

Không rỗng. Chỉ gồm các đoạn `[A-Za-z0-9_-]+` nối bằng `/`. Không có dấu `.` (vì cách osu! đặt tên frame
animation), không có `..`, không có `/` đầu hoặc cuối, không có `\`. Sai thì `ArgumentException` ngay
trong constructor của `OsbMpegCompiler`. Doc comment ghi rõ: thư mục này dành riêng cho OsbMpeg; library
chỉ ghi vào `{AssetDirectory}/s/` (sau này thêm `/a/`), không đụng tới phần nào khác của storage.

---

## 6. Thời gian, fps và làm tròn

- `fpsEff = video.Fps ?? MediaInfo.SourceFps`. Validate `0 < fpsEff ≤ 1000`.
- Frame i được lấy bằng `FrameSource` với filter `fps`, seek `From`, duration `To - From`.
- **Ngữ nghĩa bắt buộc (D7):** frame output i là frame nguồn *mới nhất* có `pts ≤ From + i/fpsEff`. Frame
  bị loại được frame trước đó đại diện, không bao giờ bởi frame sau.
  - Mặc định `round=near` của ffmpeg (v1-baseline dùng mặc định này) có thể lấy frame *sau* khi tỉ lệ
    không nguyên, ví dụ 60→24 hay 23,976→30.
  - Ở A5, agent chọn giá trị `round` (`zero`, `inf`, `down`, `up`, `near`, xem tài liệu filter `fps`) bằng
    test thực nghiệm. Tạo video mà frame k có mức xám k (ffmpeg `lavfi` + `geq` theo `N`), resample
    60→24, 60→30, 24→60 và 23,976→30, rồi khẳng định chỉ số frame nguồn đúng theo ngữ nghĩa trên. Ghi giá
    trị đã chọn và lý do vào doc comment của `FrameSource`.
- Thời gian local của frame i là `localMs(i) = i · 1000 / fpsEff` (double). Thời gian storyboard là
  `StartTime + localMs(i)`. Frame cuối hiển thị tới `StartTime + FrameCount · 1000 / fpsEff`, với
  `FrameCount` là số frame decode thực nhận được.
- IR giữ double. Làm tròn chỉ xảy ra **ở một chỗ**: `OsuParsersConverter.ToOsuParsers`, theo quy tắc
  `(int)Math.Round(ms, MidpointRounding.AwayFromZero)`, giống `OsbWriter.FormatTime` hiện có. Làm tròn
  đơn điệu nên track rời nhau vẫn rời nhau. Verifier chạy trên object *đã* làm tròn.
- Verifier lấy mẫu tại **giữa frame**: `t_i = StartTime + (i + 0.5) · 1000 / fpsEff`. Ranh giới làm tròn
  lệch tối đa 0,5 ms, còn điểm giữa frame cách ranh giới ít nhất 0,5 ms (tại fps ≤ 1000), nên không
  bao giờ lấy mẫu trúng ranh giới.
- Encoder đánh giá tại `localMs(i)`, dùng đúng frame i, không lệch.

---

## 7. Mô hình kiểm chứng

Hai phía, render bằng **cùng** renderer IR (sau S0.4), trên cùng canvas:

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
- **actual(t_i):** các object output đi qua `StoryboardWriter`, rồi
  `StoryboardDecoderGate.Decode(lines)`, rồi IR, rồi render.
- **So sánh:** `Metrics.Psnr` trên cả frame và `Metrics.TileSsim(...).Min`. Vi phạm khi
  `psnr < Preset.PsnrFloorDb` hoặc `tileMin < Preset.TileSsimFloor`.
- **Tính nhất quán với encoder:** encoder bảo đảm mọi block 16×16 đạt `blockPSNR ≥ P`. Vì MSE của frame
  là trung bình MSE của các block, `PSNR_frame ≥ min(blockPSNR) ≥ P`. Encoder cũng dùng đúng
  `TileSsim` với cùng lưới. Do đó output đúng *phải* pass verify, trừ các sai lệch nhỏ do làm tròn tọa
  độ float và do bilinear ở mép patch khi group có rotate hoặc scale khác 1 (mục 10.3).
- Trigger bị bỏ qua ở cả hai phía, vì không thể fire khi compile.
- Hai decode độc lập (encode, rồi verify) là chủ ý. Muốn nhanh thì `Verify = false`.

---

## 8. Lộ trình từng bước

### 8.1 Phase A: tái cấu trúc thành library (giữ encoder cũ, luôn chạy được)

Mục đích: có ngay một library chạy được end-to-end với API mới, trên encoder cũ tham số cố định, để M0
có output thật mà kiểm chứng. Không đổi chất lượng ở phase này.

**A0. Probe OsuParsers 1.7.2** (test compile được, trong `tests/OsbMpeg.Parsers.Tests` hiện tại).

Viết test `OsuParsersApiProbeTests` khẳng định các điểm sau:

- `Command.StartTime` là `int`;
- constructor của `Command`, `LoopCommand`, `TriggerCommand` như trong phụ lục;
- `new Storyboard()` và `new CommandGroup()` là public;
- `Storyboard.GetLayer(StoryboardLayer)` tồn tại;
- `StoryboardDecoder.Decode(IEnumerable<string>)` và `Decode(Stream)` tồn tại;
- shape của class sample trong `SamplesLayer`;
- `StoryboardEncoder` có public hay không.

Ghi kết quả vào phụ lục A của tài liệu này. Fallback nếu probe sai:

- Không có `Decode(IEnumerable<string>)`: dùng `Decode(Stream)` trên `MemoryStream`.
- Không có cả hai: ghi file tạm trong `Path.GetTempPath()`, xóa trong `finally`.
- `CommandGroup` không có constructor public: `VideoAnimation.Commands` vẫn là `CommandGroup?`, không
  đổi gì.
- Thời gian là double: vẫn làm tròn về int (tương thích stable).

Commit: `test: probe OsuParsers 1.7.2 storyboard API`.

**A1. Baseline v1.**

Nếu `docs/feasibility.md` đã có số V0 (baseline v1 trên cùng nguồn ở mục 9.2), **dùng lại** số đó,
chép vào mục 11.7 của `direction.md` và bỏ qua bước 1 đến 3. Nếu chưa có thì làm như sau:

1. `git tag v1-baseline` trên commit cuối cùng còn có CLI v1 (tag local, không push). Nếu tag đã tồn
   tại thì giữ nguyên.
2. `git worktree add ../OsbMpeg-v1 v1-baseline`.
3. Trong worktree, với mỗi fixture, trên đúng nguồn ở mục 9.2 (easy là file gốc; medium và hard là clip
   `standard`), tạo `.osbv` một dòng
   `AnimationVideo,Background,Centre,"<đường dẫn tuyệt đối tới nguồn>",320,240,0`, rồi chạy exe CLI
   `src/OsbMpeg.Cli/bin/Release/net10.0/osbmpeg.exe in.osbv out/x.osb out/x_assets`. Thêm thư mục ffmpeg
   vào `PATH` của tiến trình nếu cần.
4. Ghi các số sau vào mục 11.7 mới của `direction.md` ("Baseline v1 trên nguồn chuẩn"): wall time,
   `.osb` bytes, asset bytes, số asset, sprite, animation, command.
5. Không commit output. Output để ở `%TEMP%/osbmpeg-baseline/`.

Commit: `docs: record v1 baseline measurements`.

**A2. Gộp project** (không đổi hành vi).

- `git mv` toàn bộ `src/OsbMpeg.Compiler` sang `src/OsbMpeg`. Chuyển file của `src/OsbMpeg.Parsers`
  vào các thư mục ở mục 4.1.
- Đổi namespace theo bảng:

  | Cũ | Mới |
  | --- | --- |
  | `OsbMpeg.Parsers.Ir`, `OsbMpeg.Parsers` (Utilities) | `OsbMpeg.Ir` |
  | `OsbMpeg.Parsers.Ir.Passes` | `OsbMpeg.Ir.Passes` |
  | `OsbMpeg.Parsers.Osb` | `OsbMpeg.Osb` |
  | `OsbMpeg.Parsers.Render`, `OsbMpeg.Compiler.Shared.Render` | `OsbMpeg.Rendering` |
  | `OsbMpeg.Parsers.Osbv` | `OsbMpeg.Osbv` (tạm, xóa ở A6) |
  | `OsbMpeg.Compiler.Compilation` | `OsbMpeg.Compilation` |
  | `OsbMpeg.Compiler.Encode`, `OsbMpeg.Compiler.Shared.Analysis`, `OsbMpeg.Compiler.Detection`, `OsbMpeg.Compiler.Tuning` | `OsbMpeg.Compilation.Legacy` |
  | `OsbMpeg.Compiler.Shared.Media` | `OsbMpeg.Media` |
  | `OsbMpeg.Compiler.Shared.Evaluation` | `OsbMpeg.Verification` |

- `OsbMpeg.csproj`: `net10.0`, `Nullable`, `ImplicitUsings`, `GenerateDocumentationFile=true` (tắt
  CS1591 bằng `NoWarn`, để không ép doc cho type internal), PackageReference gồm FFMpegCore 5.4.0,
  SixLabors.ImageSharp 2.1.13, System.IO.Hashing 10.0.8, OsuParsers 1.7.2. Cli tạm tham chiếu
  `src/OsbMpeg` cho tới A6.
- Test: `git mv` các file test vào `tests/OsbMpeg.Tests` (một csproj). Bỏ BenchmarkDotNet nếu grep
  không thấy chỗ nào dùng. `InternalsVisibleTo("OsbMpeg.Tests")`.
- Cập nhật `OsbMpeg.slnx`.
- Acceptance: build sạch; số test pass bằng 43 + 41 (Cli.Tests vẫn là 1 test placeholder riêng).

Commit: `refactor: merge Parsers and Compiler into single OsbMpeg assembly`.

**A3. Storage** (mục 5; code mới, chưa ai dùng).

- File: `Storage/IAssetStorage.cs`, `Storage/FileSystemAssetStorage.cs`, `Storage/AssetIndex.cs`,
  `Storage/AssetKey.cs`.
- Test `AssetStorageTests`:
  - tạo mới trả `true`, tạo lại trả `false` và nội dung giữ nguyên;
  - path không hợp lệ thì `ArgumentException`;
  - không còn `.tmp` sau khi cancel (mô phỏng bằng token đã cancel trong lúc ghi);
  - dọn temp mồ côi cũ hơn 24 giờ nhưng giữ temp mới;
  - **đồng thời:** 8 task, mỗi task một instance `FileSystemAssetStorage` riêng trên cùng thư mục, cùng
    ghi 50 key giống nhau. Tổng số `true` đúng bằng 50, mọi file đọc lại đúng nội dung, không còn
    `.tmp`.
- Test `AssetIndexTests`:
  - cùng pixel, khác kích thước thì khác key (hồi quy bug solid colour);
  - 16 luồng cùng gọi một key thì encode đúng một lần (đếm bằng storage giả) và đúng một `Created = true`;
  - key không đổi giữa hai instance (tính quyết định);
  - lỗi storage không bị cache (lần gọi sau thử lại);
  - animation phẳng: đường dẫn frame đúng dạng `…/a/{key}{i}.png`; thiếu một frame thì chỉ ghi bù frame đó.

Commit: `feat(storage): content-addressed asset index with atomic cross-process storage`.

**A4. Converter và writer.**

- `Osb/OsuParsersConverter.cs`, internal:
  - `ToIr(IStoryboardObject, StoryboardLayer) → SbObject` và `ToIr(CommandGroup) → List<SbCommand>`:
    chuyển từ `OsbReader` hiện có, chỉ đổi chỗ đặt.
  - `ToOsuParsers(SbObject) → IStoryboardObject`: làm tròn thời gian theo mục 6. Ghép
    `VectorScaleX`/`VectorScaleY` thành một `V` bằng cùng logic `VectorPair` trong
    `OsbWriter.NormalizeCommands` (tách ra thành helper dùng chung, không viết lại). `MoveX`/`MoveY` giữ
    dạng `MX`/`MY`. Loop thành `LoopCommand`. Trigger thành `TriggerCommand`.
- `Osb/StoryboardWriter.cs`, public: `Write(Storyboard, TextWriter)`. Mỗi object của từng layer đi qua
  `ToIr` rồi `OsbWriter` (đổi `OsbWriter.Write(doc, path)` thành `Write(doc, TextWriter)`, giữ overload
  path cho test cũ nếu cần). Sample ghi `Sample,{time},{layer},"{path}",{volume}` nếu A0 xác nhận
  shape; nếu không thì throw `NotSupportedException` khi `SamplesLayer` khác rỗng. `Variables` khác
  rỗng thì throw `NotSupportedException`.
- `OsbReader.Read(path)` giữ lại cho test, nhưng đi qua converter.
- Test:
  - round-trip IR → OsuParsers → IR bằng nhau (sau làm tròn) cho mọi loại command, easing 0, 2, 3, 34,
    loop lồng, trigger;
  - `StoryboardWriter` rồi `StoryboardDecoder` rồi so object;
  - ghi dưới culture `vi-VN` không sinh dấu phẩy thập phân;
  - `Variables` khác rỗng thì throw.

Commit: `feat(osb): OsuParsers converter and TextWriter-based StoryboardWriter`.

**A5. Facade public trên encoder cũ.**

- File root: `VideoAnimation.cs`, `CompileOptions.cs` (gồm `FfmpegSettings`), `QualityPreset.cs`,
  `OsbMpegCompiler.cs`, `VideoCompileResult.cs` (gồm `CompileStatistics`, `CompilePhase`,
  `CompileProgress`), `OsbMpegException.cs`.
- Số preset giữ chỗ: High 38/0,92, Medium 34/0,88, Low 30/0,80. Ghi chú `// placeholder, calibrated in
  M0.5`.
- `Compilation/CompilePipeline.cs`, internal, điều phối mục 4.3 (bản Phase A):
  - validate (mục 10.1);
  - probe;
  - `ToIr(group commands)` (chưa có validator; validator thêm ở S0.2);
  - `LegacyTileEncoder`: đổi tên `TileEncodeLoop`, chuyển sang `AssetIndex` async (`Emit` thành async).
    Tham số cố định lấy từ baseline của tuner: TileSize 64, HashQuantLevels 32, TileTolerance 8,
    Colors 0, Gop 300, MinAnimationUniqueness 0,8, NoQuadtree false, MaxAssetPixels 17 000 000,
    RawSnapshot false. Toàn bộ cửa sổ là một đoạn, không scene detection;
  - `EmitTarget` với `CanvasMapping(W, H, Position.X, Position.Y)`, `Layer`, offset `StartTime`, và baker
    nếu có commands;
  - passes, rồi `ToOsuParsers`, rồi result;
  - kiểm tra đếm object qua vòng `StoryboardWriter` → `Decode` (thay `OsbValidator`).
- `FrameSource` và `MediaProbe`: nhận `FfmpegSettings`, dựng `FFOptions { BinaryFolder = … }` theo từng
  lần gọi, truyền vào `ProcessAsynchronously(true, ffOptions)` và `FFProbe.AnalyseAsync(path, ffOptions, ct)`.
  Hwaccel đi qua `ExtraInputArgs = $"-hwaccel {mode}"` như cũ. Lỗi ffmpeg bọc thành `VideoDecodeException`
  (giữ exception gốc làm inner).
- Progress: `Probing` một lần; `Encoding` mỗi 10 frame; hoàn tất ở 100%.
- Logging: `ILogger` cho các sự kiện mức scene hoặc video, không log từng frame.
- Test:
  - validate (mục 10.1) mỗi nhánh một test;
  - **fixture nhẹ** (luôn chạy; skip khi không tìm thấy ffmpeg, dùng `Assert.Skip`): `easy/sequence`
    0 đến 2 giây, độ phân giải và fps gốc, vào thư mục tạm. Khẳng định: có object; mọi file nằm dưới `{AssetDirectory}/s/`; ngoài
    cây đó không có file hay folder nào khác được tạo trong storage root; lần compile thứ hai cho
    `AssetsCreated == 0` và `Objects` giống hệt; hai compiler instance chạy `Task.WhenAll` trên cùng thư
    mục đều thành công, object giống nhau, không còn `.tmp`.
  - **So sánh object:** OsuParsers object không có value equality, nên "giống hệt" nghĩa là text của
    `StoryboardWriter` bằng nhau.
  - **Test tính quyết định** (cache, so sánh hai lần chạy) luôn dùng cửa sổ `From = 0`. Mục 13.5 của
    direction đã đo được decode của ffmpeg lệch ±1 frame ở biên cửa sổ khi seek.
  - Test ngữ nghĩa fps (mục 6) thuộc bước này.

Commit: `feat: public OsbMpegCompiler facade over legacy tile encoder`.

**A6. Xóa bề mặt cũ.**

- Xóa project `src/OsbMpeg.Cli` và `tests/OsbMpeg.Cli.Tests`, thư mục rác `tests/OsbMpeg.CompareTest`,
  và các dòng tương ứng trong slnx.
- Xóa `Osbv/*` và `OsbvParserTests`.
- Xóa `ParameterTuner` và `ParameterTunerTests`; `ScenePrePass`, `SceneBounds` và test; `VideoCompiler`
  (static cũ), `VideoSourcePlanner`, `VideoSourceKey` và test.
- Xóa `EncodePipeline`, `EncodeOptions`, `EncodeStatistics`, `EncodeProgress`, `NaiveBaseline`,
  `OsbValidator`.
- Xóa `AssetStore` và `AssetStoreMemoryPixelsTests`.
- Xóa `FrameWriter`, `CanvasVideoFrame`, `FrameSource.ReadBuffersAsync`.
- `SoftwareStoryboardRenderer`: thay tham số `AssetStore` và `assetRootDir` bằng `IAssetStorage`, đủ để
  build. Nâng cấp thật ở S0.4.
- **Giữ tới S1.13:** `LegacyTileEncoder`, `QuadtreeMerger`, `AnimationDetector`, `TileGrid`,
  `TileTimeline` (`TileRunTracker`), `ContentHasher`.
- Acceptance: grep không còn `Osbv`, `ParameterTuner`, `ScenePrePass`, `Spectre`, `GlobalFFOptions`.

Commit: `refactor!: remove CLI, .osbv, tuner, scene prepass and legacy stores`.

**A7. Internalize và khóa surface.**

- Mọi type không có trong mục 4.2 chuyển thành `internal` (type lồng nhau cũng vậy).
- Test `PublicApiSnapshotTests`: reflection lấy mọi type public trong assembly, cùng member public và
  protected với chữ ký dạng text, sắp xếp. So với `tests/OsbMpeg.Tests/PublicApi.approved.txt`. Khác
  nhau thì fail và in diff. File approved được tạo ở bước này và commit.

Commit: `refactor: internalize implementation, lock public API surface`.

**A8. Tài liệu.**

- `CLAUDE.md`: viết lại phần "What this is", Commands (test project mới, không CLI), Architecture
  (một assembly, public API, storage), Conventions (mục 5 và 6 tóm tắt). Giữ quy tắc về baker và
  ppy/osu#7257.
- `README.md` ngắn: ví dụ dùng library, gồm `new FileSystemAssetStorage(beatmapDir)`, `CompileAsync`,
  `AddTo`, `StoryboardWriter.Write`, cùng lưu ý R8 (`WidescreenStoryboard: 1` nằm trong `.osu`) và lưu ý
  ffmpeg.

Commit: `docs: library-first CLAUDE.md and README`.

**Acceptance Phase A:** build sạch; mọi test xanh; fixture nhẹ ở A5 xanh; snapshot API được commit; không
còn project Cli; số baseline v1 đã được ghi.

### 8.2 M0: bộ máy kiểm chứng

Thứ tự: S0.1, rồi S0.2 và S0.3 (song song được), rồi S0.4, rồi S0.5, rồi S0.6.

**S0.1. Sửa ngữ nghĩa `CommandEvaluator` và thêm golden test** (làm đầu tiên, vì mọi phép đo sau dựa
vào nó).

Bug hiện tại: `CommandEvaluator.cs:38` và `:75` trả `End` của command *kết thúc muộn nhất* khi `t` rơi
vào khoảng trống giữa hai command. Đúng phải là command *liền trước* [deepwiki: lazer giữ EndValue
của command trước], như chính doc comment của class đã viết. `first` cũng đang là command đầu theo
*thứ tự danh sách*, không phải theo `StartMs`.

Ngữ nghĩa mới, cho mỗi `kind`, với danh sách đã sort ổn định theo `StartMs`:

1. Không có command nào: trả `default`.
2. `t < first.StartMs`: trả `first.Start`.
3. Tập "active" gồm các command có `StartMs ≤ t ≤ EndMs`. Nếu nhiều, chọn command có `StartMs` lớn nhất
   (ở ranh giới dùng chung, command bắt đầu sau thắng). Command tức thời (`Start == End` theo thời
   gian) trả `End`; còn lại trả `lerp(Start, End, ease(p))`.
4. Không có active (khoảng trống, hoặc sau command cuối): trả `End` của command có `EndMs` lớn nhất
   trong số các command có `EndMs ≤ t`; hòa thì lấy `StartMs` lớn hơn.

Áp dụng giống hệt cho `EvaluateColour`. `EvaluateFlag`, `Lifetime` và `Flicker` giữ nguyên.

Hiệu năng: thêm `CommandEvaluator.Prepare(List<SbCommand>) → List<SbCommand>` (flatten loop, rồi sort
ổn định theo `StartMs`). Renderer và baker gọi một lần lúc dựng. Hàm evaluate yêu cầu input đã prepare,
có `Debug.Assert` kiểm tra đã sort.

Test `CommandEvaluatorTests`:

- ví dụ khoảng trống: `F[0,100] 0→1` rồi `F[500,600] 1→0`, tại t=300 phải ra 1;
- trước command đầu; sau command cuối;
- input không sort sau `Prepare`;
- ranh giới dùng chung (command sau thắng);
- command tức thời;
- Linear và QuadIn tại p=0,5 (0,5 và 0,25);
- khoảng trống của colour;
- flag vĩnh viễn so với flag có khoảng;
- `Flicker(1.5) == 0.5`.

`GroupTransformBakerTests` phải xanh. Test nào đang mã hóa đúng bug cũ thì sửa expectation, ghi lý do
trong commit message.

Commit: `fix(rendering): evaluator holds preceding command value in gaps`.

**S0.2. `PropertyOverlapValidator`** (`Verification/`).

- Input là danh sách `SbObject` IR. Mỗi object được `Prepare` (flatten loop) trên bản copy.
- Với mỗi object và mỗi `SbCommandKind` giá trị (`Fade`, `MoveX`, `MoveY`, `Scale`, `VectorScaleX`,
  `VectorScaleY`, `Rotate`, `Colour`):
  - sort theo `StartMs`;
  - lỗi nếu `next.StartMs < prev.EndMs − 0.5`;
  - command tức thời nằm *bên trong* khoảng của command khác cũng là lỗi;
  - hai command tức thời cùng thời điểm là lỗi.
- Bỏ qua flag (`P` không có giá trị để xung đột) và trigger.
- **Cảnh báo** (không lỗi) khi `Scale` và `VectorScale*` có khoảng giao nhau trên cùng một object, vì
  đó là rủi ro trên stable (D1).
- Output: `OverlapReport(IReadOnlyList<OverlapIssue> Errors, IReadOnlyList<OverlapIssue> Warnings)`.
  `OverlapIssue` gồm chỉ số object, kind và hai khoảng.
- Điểm gọi:
  - (a) group commands đầu `CompileAsync`: có lỗi thì `ArgumentException` nêu property và thời gian, vì
    input mơ hồ giữa stable và lazer;
  - (b) IR output trước khi convert: có lỗi thì `OsbMpegException("internal: overlapping …")`;
  - cảnh báo thì log.
- Test: `MX` chồng `MX`; `M` (qua converter) chồng `MX`; `S` chồng `S`; `S` cùng `V` chỉ ra cảnh báo;
  chồng nhau sau khi flatten loop; nối đuôi (`end == start`) pass; command tức thời bên trong khoảng là
  lỗi.

Commit: `feat(verification): property overlap validator (lazer property model)`.

**S0.3. Metrics.**

- `Metrics.Luma(r, g, b) = (77r + 150g + 29b) >> 8`. Thay `ToLuma` float bằng hàm này ở mọi nơi.
- `Metrics.TileSsim(lumaA, lumaB, w, h) → (Min, P1, Mean, MinTileX, MinTileY)`:
  - tile 64×64 không chồng;
  - dải mép hẹp hơn 16 px gộp vào tile kề;
  - `C1 = (0.01·255)²`, `C2 = (0.03·255)²`;
  - mean, var, cov tính trực tiếp trên tile.
- `Metrics.BlockMse(…)`: tiện ích cho ErrorMap.
- Giữ `Metrics.Ssim` toàn cục (chỉ để report).
- Test:
  - `TileSsim(x, x).Min == 1`;
  - invert một tile: `Min < 0.2` nhưng `Mean > 0.95` trên ảnh 512×512;
  - dịch 1 px ảnh có cạnh thì `Min` giảm rõ;
  - kích thước lẻ (1000×562) phủ đủ mọi pixel.

Commit: `feat(verification): tile SSIM and integer luma`.

**S0.4. Nâng cấp renderer.**

- **Asset RGBA:** `SpriteFrame` giữ Rgba32. Alpha tổng = `objAlpha × texelAlpha / 255`. Blend straight
  alpha như hiện tại: normal `dst·(1−a) + src·a`, additive `dst + src·a`. Canvas vẫn RGB24. Màu nền là tham số internal: mặc định đen (giống osu!); verifier truyền xám `(128,128,128)` (I14).
- **Bilinear mặc định**, có enum `Sampling { Bilinear, Nearest }`:
  - Pixel đích chỉ được vẽ khi tâm của nó rơi trong quad (`0 ≤ lx < w`, `0 ≤ ly < h`).
  - Giá trị lấy bằng bilinear tại `(lx − 0.5, ly − 0.5)`, clamp chỉ số texel láng giềng vào `[0, w−1]`
    (khớp wrap mode ClampToEdge của sprite).
  - Tint áp sau khi sample. Mipmap của osu! bị bỏ qua (`ponytail`: thêm nếu golden của client thật cho
    thấy lệch khi downscale).
- Mọi object được `CommandEvaluator.Prepare` một lần lúc dựng. Hết tình trạng bỏ qua children của loop.
- Constructor internal nhận `CanvasMapping` cùng W và H thay vì tự dựng mapping widescreen. Thêm
  property internal `SbObject.IntrinsicScale` (mặc định 1), nhân vào scale khi render.
- **Asset** đi qua `IAssetStorage` + `AssetPixelCache` (LRU, trần 256 MiB decode). Sprite ảo của
  verifier (mục 7) cho phép gắn texture trong memory qua một `Func<AssetId, SpriteFrame?>` override,
  chỉ internal.
- `RenderFrame(t, Canvas target)` tái dùng buffer, không cấp phát mới mỗi frame.
- `Pass` và `Fail`: vẽ `Pass` và bỏ `Fail` khi `passing = true`, ngược lại khi false.
- **Public `StoryboardRenderer`**: wrapper mỏng, dùng mapping widescreen chuẩn `CanvasMapping(w, h)`,
  convert object qua `OsuParsersConverter.ToIr`.
- Test (expected tính giải tích, **không** lấy từ chính renderer):
  - identity 1:1 copy chính xác từng byte;
  - phóng 2× ảnh 2×2: 16 pixel đích tính tay;
  - xoay 90° ảnh 3×3 căn tâm pixel ra đúng phép transpose;
  - property test: output luôn nằm trong [min, max] của texel;
  - `F 0.5` trên nền đen cho ra một nửa giá trị;
  - additive bão hòa ở 255;
  - `C 128,255,255` nhân kênh đỏ;
  - PNG có alpha 50%;
  - loop `L` hai vòng cho ra giá trị đúng ở vòng hai;
  - mapping auto-cover đặt tâm đúng;
  - LRU giải phóng khi vượt trần (đếm số lần decode bằng storage giả).
  - Golden handcrafted: `tests/fixtures/golden/basic.osb` đủ các lệnh M, MX, S, V, R, F, C, P và easing
    0, 2, 3, cùng asset PNG đơn sắc tạo trong test. Kiểm màu và vị trí tại 5 mốc bằng giá trị tính
    tay.

Commit: `feat(rendering): RGBA bilinear renderer, lazy bounded asset cache, public StoryboardRenderer`.

**S0.5. `WorkloadAnalyzer`** (`Verification/`).

- Input: IR đã `Prepare`, kích thước asset qua `AssetIndex.TryGet` (hoặc header PNG qua storage nếu
  thiếu), và số dòng `.osb` từ `StoryboardWriter` vào một `StringWriter` đếm dòng.
- Tại mỗi mốc 100 ms trong `[minStart, maxEnd]`:
  - object alive nếu `Lifetime` chứa t (bảo thủ, OQ-5);
  - diện tích = bbox của quad sau transform (evaluate tại t), tính trong storyboard units, chia
    `854 × 480`;
  - object có lệnh `Fade` đang nội suy tại t (`StartMs < t < EndMs` và `Start ≠ End`) tính vào
    **transient**; còn lại, nếu alpha > 0, tính vào **steady**. Alpha = 0 vẫn tính vào alive nhưng không
    tính vào load.
- `AnimationResidentBytes` = Σ `frames × w × h × 4`.
- `TotalAssetBytes` = Σ `Length` của các asset *khác nhau* được tham chiếu.
- Thuật toán quét sự kiện: sort theo start, dùng active set. Không duyệt O(N) object ở mỗi mốc.
- Test: tài liệu tay 3 sprite cho ra đúng các peak; fade window rơi vào transient. Test hiệu năng
  (`Category=Fixture`): 80 000 sprite tổng hợp chạy dưới 30 giây.
- Gắn vào `CompileAsync` để điền `Workload`.

Commit: `feat(verification): workload analyzer`.

**S0.6. `QualityVerifier` và gắn vào pipeline.**

- Hiện thực đúng mục 7. Decode lần hai bằng `FrameSource`. Tái dùng hai `Canvas` giữa các frame.
- Report theo mục 4.2. P1 là percentile 1% thấp nhất, tính trên toàn bộ frame (lưu mảng double là
  đủ).
- Public `OsbMpegCompiler.VerifyAsync` và bước verify trong `CompileAsync` (khi `Verify = true`), với
  progress `Verifying`.
- Test:
  - **positive control** (fixture nhẹ): mỗi frame i của `easy/sequence` 0 đến 1 giây thành một sprite
    full-frame, crop chính xác, tạo thủ công qua `AssetIndex`. Kết quả: `Passed`, `MinPsnr ≥ 99`.
  - **mutation:** như trên nhưng dịch một sprite 1 px, phải có vi phạm tại đúng frame đó.
  - **group transform:** positive control, nhưng group có `F` 0→1 và `M`. Expected và actual cùng
    transform nên vẫn pass. Sprite trong test được bake bằng `GroupTransformBaker`; test này đồng thời
    kiểm baker độc lập.
  - **negative control** (`Category=Fixture`): compile legacy trên clip `standard` của `medium/badapple`
    và `medium/fish`, ở `High` phải cho `ViolationCount > 0`. Chứng minh bộ máy bắt được lỗi mờ đang tồn
    tại.

Commit: `feat(verification): independent quality verifier wired into compile`.

**Acceptance M0:** mọi test xanh; positive control pass 100%; negative control có vi phạm; bảng verify
(min, p1, mean PSNR và TileSsim) của mọi fixture (nguồn theo mục 9.2) với encoder legacy được ghi vào
mục 11 của `direction.md`.

### 8.3 M0.5: calibrate preset (điểm dừng chờ người)

**S0.5.1. `ErrorMap`** (`Verification/ErrorMap.cs`).

```csharp
internal enum BlockState : byte { Clean, NearFloor, Violating }
internal sealed class ErrorMap
{
    ErrorMap(int width, int height, QualityPreset preset);   // block 16, tile 64, lưới giống TileSsim
    void Compute(ReadOnlySpan<byte> reconRgb, ReadOnlySpan<byte> sourceRgb);
    ReadOnlySpan<BlockState> Blocks { get; }                  // hàng trước, cột sau
    int BlocksX { get; } int BlocksY { get; }
    double ViolatingFraction { get; }
}
```

- `blockPSNR` tính từ MSE RGB của block (khớp `Metrics.Psnr`).
- Phân loại block:
  - `blockPSNR < P`: `Violating`;
  - `blockPSNR < P + HeadroomDb`: `NearFloor`;
  - còn lại: `Clean`.
- Theo tile 64 (luma):
  - `ssim < S`: mọi block của tile thành `Violating`;
  - `ssim < S + TileSsimHeadroom`: block đang `Clean` của tile thành `NearFloor`.
- Block mép (kích thước không chia hết cho 16) tính trên phần nằm trong frame.
- Viết scalar trước. `Parallel.For` theo hàng tile chỉ khi profile đòi.
- Test: ảnh giống hệt thì toàn `Clean`; nhiễu cục bộ đúng một block; hỏng SSIM một tile thì 16 block
  `Violating`; kích thước lẻ.

**S0.5.2. `DirtyRects`** (`Verification/DirtyRects.cs`):

- connected component 4-connectivity trên block `Violating`, cộng block `NearFloor` "đã chín" (do
  caller đánh dấu);
- lấy bbox;
- tỉ lệ lấp < 0,5 và diện tích > 64 block: split một lần theo hàng hoặc cột trống dài nhất;
- cap 1024 px mỗi chiều: vượt thì chia đều, căn theo bội 16;
- gộp rect nhỏ hơn 32×32 px vào rect kề cách dưới 32 px;
- output là rect pixel, cắt theo biên frame.

Test: hình L, hai đảo tách rời, full frame 1920×1080 (ra 4 rect ≤ 1024), một block lẻ ở mép.

**S0.5.3. `CalibrationSimulator`** (internal).

- Một lần decode, 24 simulator chạy lockstep: PSNR {30, 32, 34, 36, 38, 40} × SSIM {0,80; 0,85; 0,88;
  0,92}. Mỗi simulator giữ một canvas recon riêng.
- Mỗi frame, mỗi simulator:
  - `ErrorMap` → `DirtyRects` (chỉ block `Violating`, không headroom);
  - copy pixel nguồn của mỗi rect vào recon (hard swap);
  - cộng dồn số rect, byte PNG thật của từng rect (ImageSharp vào `MemoryStream`, không ghi đĩa) và %
    diện tích refresh.
- Dump recon PNG tại 25%, 50% và 75% cửa sổ vào thư mục output.

**S0.5.4. Harness** `CalibrationFixtureTests` (`Category=Fixture`): mọi fixture, trên nguồn ở mục 9.2. Nếu
feasibility V2 đã có bảng thì chỉ chạy lại để xác nhận. Ghi
`%TEMP%/osbmpeg-calibration/<fixture>.md` (bảng) và các PNG.

**S0.5.5. Dừng.** Agent chép bảng vào mục 11 của `direction.md`, rồi trình cho người dùng: bảng, đường
dẫn ảnh, và đề xuất ba bộ số High, Medium, Low kèm lý do (tránh bộ nào có % refresh gần 100% trên nội
dung sạch). **Chờ người dùng chốt.** Sau đó commit số vào `QualityPreset`, cập nhật mục 2.3 của
direction, cập nhật file snapshot API nếu cần.

Commit: `feat(verification): error map, dirty rects, calibration harness`, rồi
`feat: calibrated quality presets`.

### 8.4 M1: closed-loop encoder

Thứ tự:

0. S1.12 (fixture synthetic) trước tiên, vì test của S1.8, S1.10 và S1.11 cần chúng;
1. S1.1 và S1.2 (song song được), rồi S1.3 (cần `DisplayItem` và `Close` của S1.2);
2. S1.4, rồi S1.5, rồi S1.6;
3. S1.7, rồi S1.8 (chỉ patch), rồi S1.9;
4. acceptance 1 (chỉ patch);
5. S1.10, rồi S1.11;
6. S1.13;
7. acceptance đầy đủ.

**S1.1. `FrameWindow`** (`Encoder/FrameWindow.cs`).

- Ring buffer frame (`VideoFrame` giữ buffer thuê từ pool, trả khi bị đẩy ra). Capacity =
  `max(3, min(ceil(2·fps), floor(512 MiB / (W·H·3))))`.
- API: `Push(VideoFrame)`, `bool TryGet(int frameIndex, out ReadOnlySpan<byte>)`,
  `int OldestIndex`, `int NewestIndex`. Frame ngoài cửa sổ không truy cập được; đó cũng là trần tự nhiên
  của span crossfade.
- Test: đẩy quá capacity thì frame cũ nhất bị trả về pool; trần theo byte áp đúng ở 4K.

**S1.2. Display model và `ReconstructionState`** (`Encoder/`).

```csharp
internal sealed class DisplayItem
{
    int Id; int Order;                        // Order = thứ tự khai báo = z-order
    PixelRect Rect;                           // căn pixel, trong không gian video
    byte[] Pixels;                            // RGB24 Rect.W×Rect.H, sở hữu riêng
    double StartMs; double EndMs;             // EndMs = +∞ khi còn mở
    PiecewiseLinear Alpha;                    // mặc định hằng 1
    PiecewiseLinearRgb Tint;                  // mặc định hằng (255,255,255)
}
internal sealed class ReconstructionState
{
    ReconstructionState(int width, int height);
    ReadOnlySpan<byte> Canvas { get; }                    // RGB24, nền xám (128,128,128) theo I14
    double TimeMs { get; }
    void AdvanceTo(double tMs);
    void Add(DisplayItem item);                           // composite ngay vùng Rect tại TimeMs
    void Close(DisplayItem item, double endMs);           // endMs ≤ TimeMs thì composite lại vùng
    void AppendTrack(DisplayItem item, …);                // cho photometric và crossfade
    IReadOnlyList<DisplayItem> Query(PixelRect r);        // lưới bucket 128 px
    void RenderRegion(PixelRect r, double tMs, IReadOnlyList<DisplayItem> extra, Span<byte> scratch); // cho preview
    void Evict(double beforeMs);                          // bỏ item có EndMs < beforeMs, trả pixel
}
```

- **`Order` cho phép chèn ở dưới.** `Order` là khóa sắp xếp có khoảng trống (ví dụ `long` cấp theo bước
  lớn, chia đôi khi cần chèn), không phải chỉ số liên tiếp. M1 chỉ cấp tăng dần, nhưng API
  `Add(item, below: DisplayItem?)` có từ đầu, vì M2.5a phải đặt patch vá nền dưới sprite của X (D17), và
  sửa kiểu khóa sau khi đã có test tương đương là đắt. `EmitContext` xuất IR theo `Order`, nên chèn dưới
  trong display list cũng là chèn dưới trong `.osb`.
- **Công thức vẽ:** pixel = `src × tint / 255`, rồi `dst = dst·(1−a) + pixel·a`, làm tròn
  half-away-from-zero về byte. Item vẽ khi `StartMs ≤ t < EndMs`, theo `Order` tăng dần.
- **`AdvanceTo(t)`:** tập "động" gồm item có track alpha hoặc tint đang nội suy tại t, hoặc tại thời
  điểm trước đó, và item có `StartMs` hay `EndMs` nằm trong `(TimeMs, t]`. Hợp các `Rect` của chúng là
  vùng phải composite lại: clear về xám nền, rồi vẽ mọi item giao vùng.
- **Test quan trọng nhất của M1:** chuỗi ngẫu nhiên 1000+ thao tác (Add, Close, AppendTrack, AdvanceTo
  tiến lên) trên canvas 200×120. Cứ sau 10 thao tác, `Canvas` phải bằng từng byte với bản render lại
  toàn bộ tại `TimeMs`. Seed cố định, cộng 20 seed ngẫu nhiên in ra khi fail.

**S1.3. `BlockOwnership`** (`Encoder/`): dọn object bị che khuất hoàn toàn, để số sprite sống không phình
vô hạn.

- `owner[block]` = item trên cùng, *opaque*, phủ trọn phần trong frame của block. Mỗi item có biến đếm
  số block nó sở hữu.
- **Commit patch P** (opaque, alpha 1): với mỗi block P phủ trọn, giảm đếm của chủ cũ, rồi gán `owner = P`.
  Chủ cũ nào về 0 thì `Close(item, P.StartMs)`. P có thể được backdate, và việc đóng tại `P.StartMs` là
  đúng vì P phủ kín item đó.
- **Commit crossfade B:** nhận quyền sở hữu y như trên, nhưng chủ cũ về 0 được đóng tại *thời điểm kết
  thúc fade*.
- **Photometric:** không đổi quyền sở hữu.
- Item bị che một phần thì vẫn sống. Peak được report bởi `WorkloadAnalyzer`. Ngân sách là việc của M3.
- Test: patch phủ kín thì item cũ đóng đúng thời điểm; phủ một phần thì vẫn sống; hai patch lần lượt
  phủ hai nửa thì item cũ đóng sau patch thứ hai.

**S1.4. Khung candidate** (`Encoder/Candidates/`), internal.

```csharp
internal sealed record DirtyRegion(PixelRect Rect, int ViolationStartFrame, int NowFrame);
internal sealed class EncodeContext { FrameWindow Window; ReconstructionState Reconstruction; QualityPreset Preset;
    bool DisjointMode; double Fps; AssetIndex Assets; ErrorMap Scratch; }
internal interface IRepresentationCandidate
{
    CandidateKind Kind { get; }
    long EstimatedCostBytes { get; }
    int FirstAffectedFrame { get; }                       // frame đầu mà candidate làm đổi hiển thị
    void RenderPreview(int frameIndex, Span<byte> regionScratch);   // vùng Rect tại frame đó; không side effect
    Task<CommitResult> CommitAsync(EmitContext emit, CancellationToken ct);
}
internal interface ICandidateProvider
{
    int LadderRank { get; }
    IEnumerable<IRepresentationCandidate> Propose(DirtyRegion region, EncodeContext ctx);
}
```

- Danh sách provider nằm trong constructor của `ClosedLoopEncoder`, sort theo `LadderRank`. Tuyệt đối
  không viết thành chuỗi `if` (I7).
- Rank: `Photometric = 10`, `Crossfade = 30`, `Patch = int.MaxValue − 1`.
- **Evaluate (I4):** với mọi frame f trong `[FirstAffectedFrame, NowFrame]`: `RenderPreview`, rồi tính
  `blockPSNR` và `TileSsim` trên đúng vùng so với nguồn của frame f. Mọi frame đạt sàn thật thì candidate
  pass.
- Provider nào throw (trừ Patch) thì log, coi như trả rỗng. Exception của `PatchProvider` không bao giờ
  bị nuốt.

**S1.5. `EmitContext`** (`Encoder/EmitContext.cs`): người ghi duy nhất (I5).

- Giữ `OpenSprite` cho mỗi `DisplayItem`: asset entry, rect, track alpha và tint dạng đoạn tuyến tính,
  start và end.
- `Finalize(item, endMs)` và `FinalizeAll(videoEndMs)` sinh IR:
  - **Không có baker:** `SbSprite` với `Origin = TopLeft`, `X/Y = mapping.PixelToStoryboard(rect.X, rect.Y)`,
    một lệnh `S` hằng bằng `mapping.StoryboardScale` trên `[start, end]`, cộng lệnh `F` từ track alpha
    (nếu khác hằng 1) và lệnh `C` từ track tint (nếu khác hằng trắng). Thời gian = `StartTime + localMs`.
  - **Có baker:** `Origin = Centre`, tâm rect đổi qua mapping, `baker.Bake(center, baseScale, start,
    end, offset, contentTracks)`. Baker mở rộng như sau:
    - group không có `F`: nối track alpha của nội dung nguyên văn thành lệnh `F`;
    - group có `F`: disjoint mode bảo đảm alpha của nội dung luôn là 1, nên bỏ qua;
    - group không có `C`: nối track tint nguyên văn;
    - group có `C`: đi đường lấy mẫu theo từng frame, nhân `groupColour × contentTint / 255`.
    Test mới cho cả bốn nhánh.
- Output IR đi theo thứ tự `Order`.
- Asset được lấy qua `AssetIndex.GetOrCreateSpriteAsync` ngay lúc commit, rồi đếm `Created` và `Reused`.
- Test: sprite không baker cho đúng vị trí và scale (render lại bằng renderer IR, khớp pixel); tính tức
  thời của tint; baker cùng content tint.

**S1.6. `PatchProvider`** (terminal, I3).

- Propose theo thứ tự:
  1. **Backdated:** chỉ khi `ViolationStartFrame < NowFrame`. Crop vùng tại `NowFrame`, bắt đầu từ
     `ViolationStartFrame`.
  2. **Không backdate:** crop tại `NowFrame`, bắt đầu tại `NowFrame`. Candidate này luôn pass tại
     `NowFrame`, vì preview bằng nguồn từng byte.
- Item mở, `EndMs = +∞`; đóng lại bằng `BlockOwnership` hoặc ở cuối video.
- Test: region bất kỳ thì candidate 2 có preview bằng nguồn; candidate 1 bị loại khi frame giữa không đạt
  sàn.

**S1.7. Quy tắc chạy và disjoint mode.**

- **Disjoint mode** bật khi group commands (sau flatten, tính cả children của trigger) có bất kỳ `F` nào
  mang giá trị khác 1, hoặc có bất kỳ `P,A` nào. Lý do: mọi sprite sinh ra đều mang fade hoặc additive
  của group; sprite chồng nhau khi `α < 1` bị phơi sáng hai lần, và sprite additive thì cộng dồn.
- Trong disjoint mode:
  - `DirtyRects` được thay bằng lưới tile 128 px cố định: mỗi tile chứa block cần xử lý thành đúng một
    rect;
  - Crossfade trả rỗng;
  - `BlockOwnership` bảo đảm mỗi tile chỉ có một item sống (patch mới phủ trọn tile nên đóng item cũ).
- Log mode đã chọn, kèm lý do.

**S1.8. `ClosedLoopEncoder`** (`Encoder/ClosedLoopEncoder.cs`).

```text
frame 0: đánh dấu mọi block là Violating (I-frame), chỉ chạy PatchProvider
với mỗi frame i:
    Window.Push(frame i); R.AdvanceTo(localMs(i)); E.Compute(R.Canvas, frame i)
    nếu E.ViolatingFraction ≥ 0.7: cut frame
        → mọi block thành dirty, chỉ PatchProvider, không backdate
    ngược lại:
        dirty = Violating                                  # xử lý ngay, không debounce
        ripe  = NearFloor liên tục đủ HysteresisFrames frame
                → dirty, ViolationStartFrame = frame NearFloor đầu tiên
        (sàn thật không bao giờ bị debounce)
    rects = DirtyRects(dirty) (hoặc lưới tile trong disjoint mode), sort diện tích giảm dần
    mỗi rect: ladder (S1.4), commit candidate đầu tiên pass → EmitContext, BlockOwnership, R
    debug: tính lại ErrorMap trên các rect vừa sửa, phải không còn Violating
    R.Evict(localMs(i) − cửa sổ); progress mỗi 10 frame
kết thúc: EmitContext.FinalizeAll(localMs(FrameCount))
```

Ngưỡng cut 0,7 lấy từ lịch sử `CutThreshold` (mục 11.6 của direction). Test: hai ảnh tĩnh nối bằng hard
cut cho đúng một I-frame tại điểm cắt; video tĩnh cho đúng một lượt patch ở frame 0; bộ đếm hysteresis
đúng.

**S1.9. Gắn chỉ-patch vào facade.**

- `CompilePipeline` thay `LegacyTileEncoder` bằng `ClosedLoopEncoder` với danh sách provider
  `[Patch]`.
- Chạy acceptance 1, rồi ghi bảng A/B vào mục 11 của `direction.md`.

Commit: `feat(encoder): closed-loop patch encoder replaces legacy tile encoder`.

**S1.10. `PhotometricProvider`** (chỉ lệnh `C`).

- **Điều kiện propose:**
  - `Query(region)` trả đúng một item X phủ ≥ 90% region;
  - region phủ ≥ 90% `X.Rect`;
  - X opaque (alpha 1);
  - không có item động nào **khác X** giao region trong `[ViolationStartFrame, NowFrame]`. Track tint
    đang chạy của chính X không chặn, để các đoạn nối tiếp của một fade-out vẫn đi được.
- **Fit:** với A = pixel gốc của X (chưa tint), T = nguồn tại `NowFrame` trên `X.Rect`, tính theo từng
  kênh `c = 255·Σ(A·T)/Σ(A²)`. Kênh nào `c > 255` (cần làm sáng) thì trả rỗng, vì `C` chỉ nhân tối
  được. Clamp về [0, 255].
- **Candidate:** track tint của X đi tuyến tính từ tint hiện tại tại `ViolationStartFrame` tới `c` tại
  `NowFrame`. Preview = composite X với track mới. Evaluate trên toàn span.
- Commit bằng `AppendTrack`. Không tạo asset mới.
- Test: `fade_black_synthetic` không có asset mới trong đoạn fade; `tint_synthetic` sinh track `C`;
  trường hợp cần làm sáng trả rỗng.

**S1.11. `CrossfadeProvider`.**

- **Điều kiện propose:**
  - không ở disjoint mode;
  - region đã có item (không phải I-frame);
  - trong `HysteresisFrames + 2` frame gần nhất, region không bị vi phạm liên tục (liên tục thì là
    thrash, trả rỗng);
  - không có item động nào giao region trong cửa sổ.
- **Thuật toán:**

  ```text
  A = canvas hiện tại trên region (tĩnh trong cửa sổ, theo điều kiện trên)
  B = crop nguồn tại NowFrame
  thử t_s ∈ {Now−w, Now−w/2, Now−w/4, …} với w = Now − Window.OldestIndex, tối thiểu 2 frame, không sớm hơn Start của các item đang hiển thị region
      mọi f ∈ [t_s, Now]: lerp(A, B, (f − t_s)/(Now − t_s)) đạt sàn so với nguồn f?
      pass đầu tiên → candidate: item B khai báo sau, alpha 0→1 trên [t_s, Now]; chủ cũ đóng tại Now (S1.3)
  không span nào pass → rỗng
  ```

- Quyết định chỉ dựa trên frame thật trong cửa sổ, không bao giờ dựa trên mẫu.
- Test: `dissolve_synthetic` cho ra ít asset (acceptance 2); nội dung đổi đột ngột giữa lúc fade thì
  hard swap.

**S1.12. Fixture synthetic** (`tests/fixtures/make_synthetic.ps1`, commit cả script lẫn mp4, mỗi file
dưới 5 MB). Cả bốn file đều H.264 CRF 12, yuv420p.

- `static_synthetic.mp4`: 1280×720, 30 fps, 2 giây, nguồn `smptehdbars`.
- `dissolve_synthetic.mp4`: 1280×720, 30 fps, 4 giây. `smptehdbars` chuyển sang `rgbtestsrc`, scale bằng
  `xfade=transition=fade:duration=2:offset=1`.
- `fade_black_synthetic.mp4`: 1280×720, 30 fps, 4 giây, `smptehdbars` với `fade=t=out:st=1:d=2`.
- `tint_synthetic.mp4`: 1280×720, 30 fps, 4 giây, `smptehdbars` với
  `geq=r='r(X,Y)':g='g(X,Y)*max(0\,1-T/3)':b='b(X,Y)*max(0\,1-T/3)'`.

**S1.13. Dọn dẹp.** Xóa `LegacyTileEncoder`, `QuadtreeMerger` (và test), `AnimationDetector`, `TileGrid`,
`TileTimeline`/`TileRunTracker`, `ContentHasher`, `EncodeStageTimes`, cùng namespace
`OsbMpeg.Compilation.Legacy`. Cập nhật `CLAUDE.md`.

Commit: `refactor: remove legacy tile encoder`.

**Acceptance M1** (ghi mọi số vào mục 11 của direction):

1. Qua verifier độc lập, 100% frame đạt `Medium` (số của M0.5) trên mọi fixture thật (nguồn theo mục
   9.2) và 4 fixture synthetic. Report ở mức `High` được ghi lại nhưng không phải gate.
2. `dissolve_synthetic`: số asset không quá 1/5 so với chạy khi Crossfade bị tắt.
3. `fade_black_synthetic`: 0 asset mới trong đoạn `[1 s, 3 s)`.
4. `easy/complex` có group `F` 0→1 cộng `M`: disjoint mode được bật và verify pass.
5. Test đối chiếu `ReconstructionState` (1000+ thao tác) và toàn bộ unit test xanh.
6. Test đồng thời ở A5 chạy lại trên encoder mới, vẫn xanh.
7. **Tripwire hiệu năng:** wall time encode trên clip `standard` của `medium/fish` vượt 3 lần v1-baseline thì
   **dừng**, profile (OQ-1 của direction), báo người dùng trước khi tối ưu mò.
8. **Tripwire kích thước:** asset bytes của bất kỳ fixture thật nào vượt 3 lần v1-baseline thì báo
   người dùng kèm bảng. Không coi là fail, vì D2 chọn sàn cứng, nhưng người dùng phải biết trước khi
   đóng M1.
9. Bảng A/B đầy đủ so với v1-baseline: bytes (asset, `.osb`), số asset, sprite, command, wall time
   encode và verify, RAM đỉnh, `PeakAliveSprites`, `PeakSteadySbLoad`, min/p1 PSNR và TileSsim.

---

## 9. Harness đo, baseline và fixture

### 9.1 Harness

- Test nặng mang `[Trait("Category", "Fixture")]` và chỉ chạy khi biến môi trường `OSBMPEG_FIXTURES=1`.
  Thiếu biến thì `Assert.Skip`.
- Output ghi vào `%TEMP%/osbmpeg-<tên harness>/`, không bao giờ ghi vào repo.
- Mỗi harness in bảng markdown ra output của test, đồng thời ghi file `.md` cạnh output.
- Fixture nhẹ (≤ 2 giây decode) chạy mặc định, skip khi không tìm thấy ffmpeg.
- ffmpeg: tìm trên `PATH`, hoặc theo biến môi trường `OSBMPEG_FFMPEG_DIR` (truyền vào
  `FfmpegSettings.BinaryFolder`). Trên máy dev hiện tại, ffmpeg chỉ có trong thư mục của Krita, không nằm
  trên `PATH`; xem `tests/fixtures/README.md`.
- Helper `FixtureClips` (trong project test) sinh và cache clip dẫn xuất đúng theo lệnh và manifest trong
  `tests/fixtures/README.md`, dùng giá trị `round` của `FrameSource` (mục 6). Không có bản sinh clip thứ
  hai ở chỗ khác.

### 9.2 Fixture và nguồn chuẩn

Catalog, profile (`native`, `quick`, `standard`, `full`), quy tắc chọn cửa sổ và lệnh sinh clip nằm
**duy nhất** ở `tests/fixtures/README.md`.

**Quy tắc bắt buộc:** test và benchmark trên medium và hard **không bao giờ** chạy ở độ dài, độ phân giải
và fps gốc, trừ khi người dùng yêu cầu rõ. Chúng chạy trên clip dẫn xuất (cửa sổ ngắn, scale xuống, cap
fps), để không decode, render và verify một đống thông tin thừa. Clip dẫn xuất là **nguồn** cho mọi
phương án được so sánh (v1-baseline, encoder mới, verifier), nên A/B công bằng.

| Dùng cho | easy | medium, hard |
| --- | --- | --- |
| Unit test và fixture nhẹ | `native`, cắt ≤ 2 giây | không dùng |
| Harness `Category=Fixture` trong lúc phát triển | `native` | `quick` (3 s, cao 360, fps ≤ 24) |
| A1, acceptance M0, M0.5 và M1, bảng A/B | `native` | `standard` (10 s, cao 720, fps ≤ 30) |
| Kiểm dự phóng hiệu năng | không dùng | `full`, chỉ khi người dùng yêu cầu |

Cửa sổ `quick` và `standard` được chọn một lần (lần đầu sinh clip, thường trong feasibility), ghi vào
`tests/fixtures/README.md`, rồi **đóng băng** để A/B giữa các milestone còn so được.

`Fps = null` trong `VideoAnimation` (tức fps của nguồn chuẩn, đã cap sẵn trong clip) ở mọi nơi, trừ test
D7 riêng.

---

## 10. Edge case và rủi ro

### 10.1 Validate input (`ArgumentException` hoặc `ArgumentOutOfRangeException`, thông báo nêu rõ trường)

- `VideoPath` rỗng hoặc file không tồn tại: `FileNotFoundException`.
- File không có video stream: `VideoDecodeException`.
- `From < 0`; `To ≤ From`; `From ≥ Duration`.
- `To > Duration + 1 frame`: clamp về `Duration` và log cảnh báo. Không throw, vì metadata duration của
  ffprobe thường lệch vài ms.
- `Fps` ≤ 0, lớn hơn 1000, hoặc NaN.
- Group commands:
  - chồng lấn property: `ArgumentException` (S0.2);
  - trigger chứa lệnh ngoài F/C/P,A: `NotSupportedException` (baker hiện có);
  - loop flatten ra hơn 1 000 000 command: `ArgumentException`, để chặn nổ bộ nhớ.
- `AssetDirectory` sai quy tắc 5.7: `ArgumentException` ngay trong constructor.

### 10.2 Edge case phải có test

- Kích thước video không chia hết cho 16 (block mép) hoặc lẻ (ví dụ 1918×1078).
- Video nhỏ hơn 64 px.
- Video 4K: rect cap 1024 vẫn đúng; không asset nào vượt 3840.
- Video toàn đen: PSNR = 100; asset đơn sắc dedupe đúng theo kích thước.
- Một frame duy nhất.
- VFR: filter `fps=` chuẩn hóa sẵn.
- `Fps` lớn hơn nguồn: output giống hệt khi `Fps = null`, chỉ tốn thời gian (một test D7 trên
  `static_synthetic`).
- `Fps` nhỏ hơn nguồn (60 → 30): số frame giảm một nửa, thời gian các sprite rơi trên lưới 33,33 ms.
- Cancel giữa encode: `OperationCanceledException`, không còn `.tmp`.
- Disk full: `IOException` lan ra ngoài, không còn `.tmp`.
- Cùng một instance compile hai video khác nhau đồng thời.

### 10.3 Rủi ro đã biết

| Rủi ro | Mức | Giảm thiểu |
| --- | --- | --- |
| Nội dung nhiễu (`hard/birdbrain`, `medium/badapple`) cần patch gần như full refresh dưới sàn cứng, output có thể lớn hơn v1 | Cao | Tripwire kích thước (acceptance 8); M0.5 cho thấy đường cong trước; concept encoding và lookahead ở mục 12 |
| Hiệu năng: `RenderPreview` trên mọi frame bị ảnh hưởng, span search của crossfade, verify bằng decode và render lần hai | Cao | Tripwire hiệu năng; preview chỉ trên region; `Verify = false` khi cần; OQ-1 |
| Hành vi lazer lấy từ deepwiki (gap hold, S×V, chaining) chưa kiểm trên client thật | Trung bình | Golden test giải tích ở S0.1 và S0.4; tier vàng client thật là concept ở mục 12 |
| Mép patch lệch nhỏ khi group có rotate hoặc scale ≠ 1 (bilinear tại mép từng sprite so với một texture liền) | Thấp | Chấp nhận. Verify có thể báo vi phạm sát sàn; ghi vào report, không nới sàn |
| Sprite bị che một phần tích tụ, overdraw tăng | Trung bình | `BlockOwnership` đóng sprite bị che kín; peak có trong `WorkloadReport`; ngân sách ở M3 |
| Group có chuyển động: baker lấy mẫu mỗi frame cho *từng* sprite sống. Sprite của closed loop sống lâu và có thể chồng nhau, nên số command tăng theo số sprite sống × số frame | Trung bình | Đo `CommandCount` và `OsbLineCount` trong acceptance 9; concept "fit trajectory cho group" (R2) ở mục 12 |
| OsuParsers `StoryboardDecoder` không thread-safe | Thấp | Mọi lần gọi đi qua `StoryboardDecoderGate` |
| Antivirus hoặc indexer khóa file mới trên Windows | Thấp | Retry có backoff (mục 5.2) |
| Asset trong cache bị xóa từ bên ngoài giữa lúc chạy | Thấp | Verifier lộ lỗi; ghi trong doc comment |
| `FrameSource` có thể bỏ lại một reader mồ côi khi ffmpeg không mở pipe (doc comment hiện có) | Thấp | Giữ nguyên hành vi đã đo; bọc thành `VideoDecodeException` |

---

## 11. Tiêu chí hoàn thành tổng (hết tài liệu này)

- Một assembly `OsbMpeg` trên `net10.0`. Không còn project CLI, `.osbv`, tuner hay scene pre-pass.
- Public API đúng mục 4.2 và bị khóa bằng snapshot.
- `OsbMpegCompiler.CompileAsync` cho một video ra object OsuParsers, asset content-addressed trong
  thư mục dành riêng, `VerificationReport` và `WorkloadReport`.
- Cache: compile lại cùng nội dung tạo 0 asset mới. Nhiều instance hoặc tiến trình dùng chung thư mục
  không có lỗi, không ghi đè, không còn file tạm.
- Encoder closed loop với Patch, Photometric và Crossfade. Verify độc lập pass `Medium` trên mọi fixture.
- Mọi acceptance của Phase A, M0, M0.5 và M1 xanh, số liệu được ghi vào mục 11 của `direction.md`.
- `CLAUDE.md` và `README.md` phản ánh kiến trúc mới.

---

## 12. Lộ trình sau M1

Mỗi bước dưới đây có spec đủ để bắt đầu; chi tiết file và test được viết thêm khi tới lượt, cùng kiểu với
mục 8. Mọi số đo ghi vào mục 11 của `direction.md`; mọi A/B so với bước ngay trước và với `v1-baseline`.

### 12.1 Thứ tự và keep-gate

| Bước | Nội dung | Vào từ | Keep-gate (đo trên tier `authored` cộng `standard` của medium và hard) |
| --- | --- | --- | --- |
| R1 | Dataset có ground truth chương trình, tier `authored`, metric tầng hai (12.2) | ngay sau M1 | là instrument, không gate; sai lệch tier rẻ so với render thật là phát hiện |
| M2 | `TransformTrackProvider` K=1 cộng sub-segment theo motion (12.3) | sau R1 | acceptance trong 12.3 |
| R2 | Easing rời rạc cộng piecewise trong `TrajectoryFitter` (12.4) | ngay sau M2 | ≥ 30% command bytes trên `authored`, 0 verify regression; trượt thì giữ RDP linear |
| M2.5a | Layer nhân quả theo rect: X là một sprite di chuyển, nền lộ ra do closed loop vá và chèn dưới X (12.5) | sau R2 | trên `authored`: bytes ≤ 0,5× M2 và sprite ≤ 0,2× M2, verify 100%; trên medium và hard: bytes không tăng quá 2%, verify 100%; trượt thì quay về DEFER tới M3.5 |
| M2.5b | Mask và alpha cho X, dựng nền nhìn trước, khử lem màu (12.6) | sau M2.5a, chỉ khi bucket `rect-halo` hoặc `reveal-strip` ≥ 20% bytes trên `authored` | bytes ≤ 0,7× M2.5a trên fixture kích hoạt nó, verify 100%, không fixture nào tệ đi |
| M3 | Ngân sách workload, `Animation` có cap, encoding asset (12.7) | sau M2.5 (ngân sách và animation có thể sớm hơn nếu số của M1 đòi) | acceptance trong 12.7 |
| R3 | Oracle ILP đo oracle gap của ladder (12.8) | sau M3 | là instrument; gap ≥ 10% thì đề xuất nâng bộ chọn, dưới 10% thì đóng câu hỏi ladder |
| M3.5 | Attribution và quy trình quyết định (12.8) | sau M3 và R3 | không phải feature |
| Sidecar | `ILayerBoundaryRefiner` bằng model matting trong tiến trình Python (12.9) | sau M3.5 | chỉ xây khi bucket `layer-edge` ≥ 20% bytes hoặc verify fail tập trung ở biên layer trên 2D animation |

### 12.2 R1: dataset ground truth, tier `authored`, metric tầng hai

Theo `direction.md` 6.5. Ba nguồn, không nguồn nào commit video (xem `tests/fixtures/README.md`):

- `tools/OsbMpeg.DatasetGen` (project console riêng, không đụng assembly `OsbMpeg`): template theo
  category (tĩnh; một transform đơn `M`, `S`, `R`; từng họ easing; fade; colour; crossfade; pan; zoom; nhiều
  sprite; layered motion kèm occlusion; flipbook; loop; hỗn hợp; nhóm mập mờ), có seed, render bằng
  `StoryboardRenderer` ra frame lossless rồi ffmpeg x264 CRF 16, kèm `metadata.json` và `source.osb`.
- Loader Crello Animation (`cyberagent/crello-animation`): texture 256×256 RGBA, ma trận warp và opacity
  mỗi frame cho 2 đến 6 sprite, 50 frame ở 10 fps; render ra `.mp4` vào cache fixture. Ground truth là
  chương trình (texture + affine + opacity), chuyển về IR để tính metric tầng hai.
- Loader Animation2Code (`anim2code/dataset`): video cộng HTML/CSS/JS; chỉ dùng video và metadata, kiểm
  license trước khi tải.

Metric tầng hai (`complexity_ratio`, `representation_confusion`, `missing_primitive_attribution`,
`trajectory_rmse`, `asset_recall`, `occlusion_consistency`) theo bảng ở `direction.md` 6.5, giữ riêng
từng metric, không score tổng hợp, không metric nào làm gate. Harness `Category=Fixture` in bảng theo
category. Acceptance R1: generator sinh được mọi category với seed cố định và round-trip render khớp
golden; hai loader chạy được trên máy có mạng và skip sạch khi không; bảng metric tầng hai chạy trên
output M1 của toàn tier `authored`.

### 12.3 M2: motion K=1 dưới interface layer-set

Mục đích: hết nặng trên pan và zoom, vì nội dung do camera dịch chuyển ngừng bị re-emit mỗi frame.
Phụ thuộc M1 và R1. Không làm: K lớn hơn 1 (M2.5), panorama (DEFER tới M3.5), camera rotation (bỏ ở bản
này, thêm khi attribution đòi, `R` đã sẵn). Scene pre-pass không còn, nên M2 phải tự có segmenter (S2.3)
và dùng quy tắc cut của `ClosedLoopEncoder` (S1.8) làm ranh giới cứng.

**S2.R Mở rộng display model (MUST, làm đầu tiên).** `DisplayItem` của M1 là ảnh RGB24 căn pixel với
alpha toàn cục; sprite di chuyển không đặt vào được. Thêm: pixel RGBA (alpha theo pixel, mặc định 255),
track Sim(2) dạng đoạn tuyến tính `(x, y, s, θ)` quanh tâm, composite bilinear trên footprint đã
transform. Tập "động" của `AdvanceTo` gồm cả item có transform đang nội suy; vùng composite lại là hợp
footprint trước và sau. `BlockOwnership`: item có transform hoặc alpha theo pixel không bao giờ làm chủ
block và không bao giờ bị đóng bởi quyền sở hữu; nó đóng ở cuối đoạn hoặc khi gate gỡ nó. Item tĩnh bị
item di chuyển che một phần thì không bị đóng. Test: mở rộng test tương đương ngẫu nhiên của S1.2 với
transform và alpha, so với renderer IR, sai tối đa ±1 mỗi kênh.

**S2.G Quy tắc commit trước, vá sau (MUST).** Candidate sống dài hơn `FrameWindow` (I4) được commit ở đầu
đoạn khi qua gate theo đoạn: pass A downscale render composite dự kiến cho mọi frame của đoạn và đo diện
tích phần dư; nhận khi lợi ích ròng ước lượng dương và phần dư dưới ngưỡng (gate hai tầng của S2.2 là một
trường hợp). Sau commit, closed loop kiểm full-res từng frame và vá như thường. Nếu bytes vá thực tế trong
đoạn vượt bytes ước lượng của phương án không có candidate đó, ghi vào report để hiệu chỉnh gate; không
quay lui trong đoạn.

**S2.0 Spike ước lượng (MUST, trước khi xây phần còn lại).** Block matching chỉ translation trên
`pan_synthetic` và fish_spinning 5 giây, đo inlier ratio và residual. Quyết: block matching đủ, hay
cần pyramidal, hay cần DIS flow, theo đúng thứ tự đó (learned flow đã REJECT, `direction.md` 5.6). Phạm vi an
toàn: code spike ném đi được, chỉ số liệu ghi vào mục 11 của `direction.md` mới là deliverable.

**S2.1 FlowField (MUST, thuật toán theo kết quả S2.0).** Phương án mặc định: downscale luma xuống tối
đa 256px (decode pass A riêng), mỗi block 16×16 tìm SAD trong bán kính ±12px, 3 mức pyramid nếu S2.0
đòi. Output là vector thưa kèm confidence theo SAD.

**S2.2 MotionModelFitter (MUST).** RANSAC similarity không rotation: model `(tx,ty,s)`, 2 điểm mỗi
mẫu, inlier là residual dưới 1px ở full-res, 64 vòng lặp. Interface trả
`IReadOnlyList<MotionLayer>`, bản này luôn 0 hoặc 1 phần tử, nhưng là layer-set ngay từ ngày đầu.
Gate hai tầng, cả hai phải đạt: inlier ít nhất 60% trong ít nhất 70% số frame; và diện tích residual
dự kiến (phần camera model không giải thích được) không quá 25% diện tích frame, tính trung bình trên
scene. Gate thứ hai tồn tại vì riêng inlier ratio cho phép accept một layer mà 40% frame vẫn phải
patch mỗi frame *chồng lên* một base sprite full-res cũng đang trả tiền; gate này đo thẳng cái quyết
định net win. Không đạt gate nào thì trả list rỗng và scene chạy thuần patch cộng crossfade, an toàn
tuyệt đối vì motion layer chỉ là optimization.

**S2.3 MotionSegmenter (MUST).** Từ chuỗi model mỗi frame: đoạn có `|v| < ε` kéo dài ít nhất 500ms
thành segment tĩnh, chuyển tiếp có hysteresis 300ms. Output là scene chia thành các segment
`[(t0, t1, MotionLayer?)]`. Encoder chạy theo từng segment, và I-frame ở đầu segment rơi ra tự nhiên
từ quy tắc I-frame của `ClosedLoopEncoder` (S1.8), chính là keyframe tại điểm đổi chế độ. Không tách segment cho thay đổi motion nhẹ, vì
overhead keyframe tự trả giá trong cost và ladder tự quyết.

**S2.4 TrajectoryFitter (MUST).** RDP joint: điểm là `(tx,ty,s)` mỗi frame, hàm khoảng cách là *dịch
chuyển lớn nhất của 4 góc frame* dưới transform đã compose, so giữa đường nội suy và giá trị thật,
tính bằng osu!px, epsilon tạm 0,3. Không chạy RDP riêng từng thành phần với epsilon theo đơn vị vị
trí: sai số scale Δs tốn `Δs·|x−c|` px, lớn nhất ở góc ảnh và gần như vô hình với epsilon đo trên
(tx, ty). Simplify joint cũng giữ breakpoint của `MX`, `MY`, `S` đồng bộ, nên ba track đọc được như một.

Ghi chú toán: với `p(u) = p₀ + uΔp` và `s(u) = s₀ + uΔs`, vị trí màn hình của một điểm ảnh cố định x
là `y(u) = p₀ + s₀(x−c) + u·[Δp + Δs(x−c)]`, tức tuyến tính theo u trên mỗi segment. Nên M và S nội
suy tuyến tính đồng thời là đúng, chỉ có phép đo sai số phải nằm trong screen space.

**S2.5 TransformTrackProvider và fold vào baker (MUST).** Không có baker: sprite base full-frame (crop
tại keyframe đầu segment), origin Centre, emit `MX`, `MY`, `S` keyframe linear từ trajectory, giải
`p(t)` và `s(t)` từ `screen = p(t) + s(t)·(x−c)`.

Có baker thì phải giữ single-writer (I5): `GroupTransformBaker.BakeTracked(cameraSamples, ...)` compose
bằng cách lấy mẫu. Với mỗi frame nguồn t: lấy group transform tại t (logic `SampleAt` sẵn có) rồi
compose với camera `(p(t), s(t))`, ra chuỗi state cuối (x, y, sx, sy, α), rồi chạy RDP joint screen
space *một lần* trên kết quả và emit. Lý do lấy mẫu thay vì đại số: tích của hai track tuyến tính là
bậc hai, nên đừng làm đại số piecewise.

Đây chính là chỗ dễ sai nhất của toàn bộ thiết kế. `SbCommandKind` chỉ có một slot cho mỗi
MoveX, MoveY, Scale, VectorScaleX, VectorScaleY trên mỗi object, mà `GroupTransformBaker.Bake` đã phát
`MoveX`, `MoveY`, `VectorScaleX`, `VectorScaleY` hằng số lên *mọi* tile khi `positionConstant` đúng,
tức trường hợp thường gặp. Nếu motion layer phát track riêng cạnh track của baker thì đó là vi phạm
R5 và không tầng nào bắt được nếu thiếu `PropertyOverlapValidator`: `MergeAdjacentCommands` chỉ kiểm
tính liền kề, `OsbValidator` chỉ đếm object, còn `CommandEvaluator` giải quyết chồng lấn theo kiểu
"cái cuối trong danh sách thắng", một cách thứ ba khác cả stable lẫn lazer, nghĩa là verify có thể
pass sạch trong khi client thật render khác đi.

Base keyframe refresh: khi `ErrorMap` cho thấy vùng base (không bị patch đè) suy thoái tới ngưỡng cận
sàn trên hơn 40% diện tích thì mở keyframe mới, chuyển bằng `CrossfadeProvider` trên vùng full-frame,
tức tái dùng cơ chế chứ không viết riêng.

Trên `authored`, flow block matching có thể không dùng được vì nội dung ít texture (OQ-2): S2.0 phải đo
thêm hai đường cho Sim(2) là KLT trong support khi có texture và ECC trên pixel khi không, khởi tạo bằng
flow block cộng RANSAC. Kết quả S2.0 quyết định `FlowField` có một hay hai backend.

Provider rank 20, chỉ propose khi `ctx.Motion` có layer phủ region và region chưa có base đúng. Mép lộ
ra khi pan là coverage gap có chủ đích, không thuộc layer nên đi theo ladder thường và thành patch,
đúng như mong đợi. Guard 4K: nguồn có chiều nào vượt 3840 thì `SceneMotionInfo` trả rỗng và ghi lý do
vào log, vì base sprite ở độ phân giải nguồn sẽ chạm `MaxTextureSize`, mà cap diện tích 17Mpx² không
bắt được vi phạm theo từng chiều.

**S2.6 Fixture (MUST).** Tier `authored` đã có từ R1; thêm hai fixture synthetic: `pan_synthetic` (ffmpeg zoompan trên ảnh tĩnh lớn) và
`pan_foreground_synthetic` (pan cộng một sprite chuyển động độc lập chiếm khoảng 20% frame, thiết kế
để vừa lọt gate 60/70), script tạo trong `tests/fixtures/make_synthetic.ps1` (S1.12). Thêm luôn
`object_reveal_flat` và `object_reveal_textured` dùng cho M2.5: camera tĩnh, một vật X rộng khoảng 15%
khung đi ngang qua nền phẳng hoặc nền có texture trong 3 giây, làm lộ dần phần nền bị che.

Acceptance M2: (1) `pan_synthetic` asset bytes không quá 1/5 so với M1; (2) minecraft 10 giây không
quá 60% bytes so với M1; (3) `pan_foreground_synthetic` cho thấy gate hai tầng từ chối hoặc nhận đúng
theo net win, kiểm cả hai nhánh bằng cách chỉnh kích thước foreground; (4) 100% frame vẫn đạt sàn qua
verify độc lập; (5) scene bị từ chối motion cho output trùng byte với M1.

Thứ tự M2: S2.R → S2.0 → S2.1 → S2.2 → S2.G → S2.4 → S2.5 (nhánh không baker) → S2.3 → S2.5 (fold
baker) → S2.6 → acceptance.

### 12.4 R2: easing rời rạc trong `TrajectoryFitter`

Theo `direction.md` 6.3. Tập hàm là 35 easing của osu! (0 tới 34), không tham số tự do ngoài `(v0, v1)`
của segment; cubic Bézier, spring hay curve tùy ý không tồn tại trong `.osb` nên không nằm trong tập.
Bắt đầu với chín hàm (Linear, Quad và Cubic In/Out/InOut, Sine In/Out), thêm Elastic, Back, Bounce khi
dataset R1 chỉ ra.

Thuật toán: với mỗi track vô hướng đã có breakpoint từ RDP joint, với mỗi đoạn giữa hai breakpoint ứng
viên, đánh giá mọi easing trong tập với residual dạng đóng trong screen space (cùng metric góc frame như
S2.4); nhận easing đạt epsilon thay vì chèn keyframe. Piecewise bằng changepoint tối ưu (PELT hoặc DP
trên số sample mỗi track, cost là residual của easing tốt nhất trên đoạn, penalty là một command); chọn
theo `Error + λ·Complexity` với fidelity là ràng buộc cứng. Không easing nào đạt epsilon thì fallback RDP
linear; trên quỹ đạo nhiễu kết quả phải trùng RDP hiện tại. Áp cho mọi track: x, y, scale, alpha,
colour (Photometric và Crossfade dùng chung fitter).

Acceptance R2: trên `authored`, command bytes giảm ít nhất 30% so với M2 với 0 verify regression; trên
case generator thuộc "từng họ easing", `representation_confusion` chỉ ra easing đúng ở ít nhất 90% case
không mập mờ; trên medium và hard, output trùng byte với M2 hoặc nhỏ hơn.

### 12.5 M2.5a: layer nhân quả theo rect

Mục đích: nội dung được author (D14) là nhiều vật độc lập di chuyển trên nền; K=1 không bắt được. Theo D12,
không có bước định danh object: layer được *đề xuất* bằng motion và được *nhận* chỉ khi đo thấy rẻ hơn.
Theo D17, nấc này không dựng nền và không có alpha. Phụ thuộc M2 (S2.R, S2.G) và R2.

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
MovingRectProvider     : ICandidateProvider, rank 25, propose ở đầu đoạn theo S2.G
```

Candidate: sprite là ảnh cắt bounding rect của support tại frame chuẩn (frame trong đoạn có support lớn
nhất và ít bị che nhất theo sự kiện occlusion), pixel đặc, track `MX`, `MY`, `S`, `R` từ quỹ đạo qua
`TrajectoryFitter` và easing R2, ghép qua baker (I5). Chỉ cần rect của frame trước và frame hiện tại để
dựng quỹ đạo; không lưu lịch sử rect, vì vùng lộ do `ErrorMap` tìm.

**Quy tắc z cho patch vá (điểm dễ sai nhất).** Trong `.osb` object khai báo sau nằm trên. Patch vá nền sinh
sau sprite X, nếu đặt mặc định sẽ nằm trên X; khi X quay lại, patch che X, vùng X bẩn và patch chồng patch.
Quy tắc: patch có rect *không giao* footprint của bất kỳ layer di chuyển nào trong khoảng sống của nó được
chèn ngay dưới layer thấp nhất (`Add(item, below)` ở S1.2); patch giao footprint thì nằm trên như thường.

| Vấn đề | Cách xử lý ở M2.5a |
| --- | --- |
| Đếm layer không cần người | matching pursuit (`direction.md` 5.7 tầng 1), K ≤ 4 |
| Di chuyển nhanh, rect cũ và mới không giao | không ảnh hưởng tới vùng lộ; chỉ cần flow bắt kịp (S2.0) |
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
| A1 | Layer rect từ `ILayerProposer` classical có thắng M2 cộng R2 không | bytes ≤ 0,7× và verify 100% trên ít nhất 2 trong 3 nhóm |
| A2 | KLT trong support so với ECC trên pixel cho Sim(2) | residual screen-space dưới 0,5 px ở ≥ 90% frame trên `authored` |

Acceptance M2.5a: keep-gate ở 12.1; `object_reveal_flat` cho đúng một asset X và không patch nào trong
footprint của X; `object_reveal_textured` hoặc được nhận với bytes thấp hơn M2, hoặc bị gate từ chối với
output trùng byte M2; case generator "layered motion kèm occlusion" cho `occlusion_consistency` ≥ 0,9;
bucket `rect-halo` và `reveal-strip` được báo cáo cho mọi fixture.

### 12.6 M2.5b: mask, alpha và dựng nền nhìn trước (có gate)

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
kiểm thử khô       như S2.G, trên composite các layer cộng nền
pass B, encode     đặt nền và layer vào display list ở đầu đoạn, closed loop vá phần dư
```

Dựng nền và dựng sprite X là cùng một phép toán: tích lũy trong hệ quy chiếu của một layer theo quỹ đạo và
mask của chính nó; panorama khi camera động là trường hợp riêng. Lấp lỗ là bắt buộc chứ không phải tùy chọn:
ở mép X alpha nhỏ hơn 1, nên lỗ trong suốt của nền sẽ lộ qua mép và vi phạm I14. Glow và flash dùng additive
layer T9 trong cùng bước: trong hệ đã bù chuyển động, `Base` là percentile thấp theo thời gian, `Light` là max
delta, `α(t)` đóng dạng, gate clamp ≤ 2% và SSIM. Chi phí thêm: một lần decode full-res mỗi đoạn có layer.

Spike A4 (additive trên glow thật) và A5 (chi phí GPU của sidecar, chỉ khi gate 12.9 mở) thuộc nấc này.
Acceptance M2.5b: keep-gate ở 12.1; trên `object_reveal_textured` bucket `reveal-strip` gần 0 và verify 100%.

### 12.7 M3: ngân sách, animation, encoding asset

Phụ thuộc M1 cho ngân sách và encoding, M2 không bắt buộc cho encoding.

**S3.1 Thực thi WorkloadBudgets (MUST).** Bộ đếm cập nhật tại `Commit` (timeline SB load steady và
transient, alive count, animation resident). `Allow(candidate)` kiểm dự phóng sau commit so với ngân
sách ở `direction.md` 2.2. Không đạt thì encoder thử candidate hoặc provider tiếp theo, tức đổi representation chứ
không hạ sàn (I6). PatchProvider bị ngân sách chặn về lý thuyết không xảy ra vì patch rời nhau về thời
gian nên không tăng alive dài hạn; nếu vẫn xảy ra thì vẫn commit và ghi violation, vì fidelity thắng.
Test: ngân sách giả nhỏ làm crossfade bị chặn và rơi về patch, bản ghi violation đúng.

**S3.1b Giải ngân sách bằng solver (INVESTIGATE, theo D18).** Khi một đoạn vượt ngân sách, chọn đổi
representation ở vùng nào là multiple-choice knapsack với ràng buộc dung lượng theo từng lát thời gian:
biến `x[r,k]` chọn candidate k cho vùng r trong số candidate *đã đo*, `Σk x[r,k] = 1`, tối thiểu tổng bytes,
với mọi lát t: `AliveSprites(t) ≤ 500`, `SbLoad(t) ≤ 4.0`, resident ≤ trần. Chỉ giải trên đoạn vượt nên tập
nhỏ. Chỉ xây khi S3.1 greedy để lại vi phạm hoặc R3 cho thấy greedy ở đoạn vượt bỏ phí từ 10%.

**S3.2 AnimationProvider và ThrashPlanner (MUST).** Pass hậu kỳ mỗi scene, không nằm trong vòng lặp
mỗi frame. Quét lịch sử commit theo vùng: vùng có ít nhất 4 hard swap liên tiếp cách nhau đúng một
frame và tổng span tuân cap (tối đa khoảng 120 frame, diện tích không quá tile 256px, tổng resident
không quá 200MB mỗi scene) thì gom chuỗi patch thành một `SbAnimation` LoopOnce và xóa các sprite
patch tương ứng khỏi document. Thao tác trên IR trước các pass, nên `EmitContext` cần hỗ trợ thay thế.
Gate uniqueness như bản cũ (ít nhất 80% frame khác nội dung), logic đọc lại từ git history rồi viết
gọn lại. Test: bad_apple 5 giây có animation resident trong ngân sách và giữ sàn; vùng nhấp nháy A/B
chỉ có hai nội dung thì *không* được thành animation, vì dedupe sprite thắng.

**S3.3 Tìm encoding asset (MUST cho palette có dither, INVESTIGATE cho JPEG).** JPEG đã đo *tệ hơn*
PNG một lần (+6,5% trên anime, +62,6% trên bad_apple, doc comment `AssetStore.cs`), nên chỉ được giữ nếu
hai điều cùng đúng: OQ-4 pass (tự tay bỏ một `.jpg` vào storyboard test, load trong osu! stable và lazer,
xác nhận render) và đo trên `standard` của footage cho thấy giảm từ 30% mà không thủng gate. Một trong
hai fail thì bỏ JPEG, chỉ giữ dither palette.

`AssetStore.GetOrAdd(pixels, w, h, consumer, EncodingPolicy)` thử các ứng viên gồm PNG thô, PNG palette
256/128/64 kèm Floyd-Steinberg, và JPEG q95/q90/q85 chỉ khi vùng opaque; decode lại, tính `TileSsim` so
với gốc, yêu cầu đạt `preset.TileSsimFloor + 0.02`, rồi chọn bytes nhỏ nhất. Key = hash pixel *sau* lossy (mục 5.3, D9), nên vẫn chỉ phụ thuộc nội dung; tăng tag lên `V2` nếu đổi
quy tắc; phần mở rộng file theo format. Quyết định được cache theo content-hash nên asset đã gặp không phải tìm lại. Test:
gradient thì palette bị loại bởi gate SSIM, ảnh chụp thì JPEG thắng, và dedupe vẫn đúng khi hai policy
cùng chạy trong một compile.

Acceptance M3: bad_apple giữ sàn với animation trong ngân sách; palette có dither giảm asset bytes mà
không thủng gate trên `authored` và `fish-short`; JPEG được giữ hay bỏ theo đúng hai điều kiện ở S3.3,
ghi số; không fixture nào vượt ngân sách mặc định hoặc có violation report rõ ràng.

### 12.8 M3.5: checkpoint attribution

Mục đích: quyết định ưu tiên nghiên cứu tiếp theo bằng số liệu (I9). Đây không phải milestone feature.

**S3.5.1 AttributionAnalyzer (MUST).** Chạy trên kết quả compile toàn corpus cộng dữ liệu trung gian
của encoder (flow, model, commit log, serialize ra `attribution.json` khi `CompileOptions.Attribution` bật).

| Bucket | Cách đo |
| --- | --- |
| `local-coherent-motion` | bytes của patch trong vùng mà sequential RANSAC pass 2 (chạy offline trong analyzer trên flow đã lưu) tìm được model K=2 giải thích ít nhất 70% block vùng đó |
| `pan-reveal` | bytes của patch nằm trong dải mép ngược hướng camera translation, bề rộng dải bằng `\|v\|·Δt` |
| `perspective-residual` | bytes patch ở vùng inlier ở tâm nhưng residual tăng theo bán kính (fit tuyến tính residual theo khoảng cách tâm, độ dốc vượt ngưỡng) |
| `near-duplicate-assets` | bytes của asset có coarse hash (aHash 8×8) trùng asset khác nhưng exact hash khác |
| `deformation` | patch còn lại có SSIM giữa hai lần refresh liên tiếp cùng vùng ít nhất 0,7, tức đổi có cấu trúc |
| `noise` | phần còn lại (SSIM giữa hai lần refresh dưới 0,7), bucket không cứu được |

Thêm ba bucket cho layer: `rect-halo` (bytes patch nằm trong footprint của một layer rect đã nhận ở M2.5a, tức
nền bị kéo theo trong rect), `reveal-strip` (bytes patch chèn dưới layer để vá nền vừa lộ), và `layer-edge`
(bytes patch trong dải biên dưới 8px của layer có alpha ở M2.5b). Output
là bảng phần trăm bytes theo bucket, theo từng fixture và trên toàn corpus, ghi vào mục 11 của `direction.md`.

**S3.5.2 Quy trình quyết định (MUST, đây là quy trình chứ không phải code).** Bucket lớn nhất từ
khoảng 25 đến 30% trở lên thì mở chu trình research, prototype, benchmark cho hướng tương ứng trong
menu: Panorama, CanonicalAssetReuse, PerspectiveStrips, và sidecar `ILayerBoundaryRefiner` khi `layer-edge`
từ 20% (mục 12.9). Bucket `noise` trội
thì dừng, ship, và ghi giới hạn vào README. Không đặt tên M4 hay M5 trước; milestone sau M3.5 chỉ tồn
tại sau khi attribution và benchmark của prototype thắng.

**R3 Oracle ILP (instrument, theo D18).** Trong project đo, trên cửa sổ ngắn của tier `authored` và clip
`standard`: chế độ exhaustive sinh mọi candidate, encode và kiểm pixel từng cái, rồi giải bài toán phủ có
trọng số bằng CP-SAT:

```text
ô    = (block, frame) vi phạm;  x_c ∈ {0,1} cho mỗi candidate c đã đo là đạt sàn
min  Σ bytes_c · x_c
s.t. mỗi ô được phủ ít nhất một lần; candidate đã chọn không chồng nhau trong cùng ô
     AliveSprites(t) ≤ 500, SbLoad(t) ≤ 4.0  với mọi t
```

So nghiệm với lựa chọn của ladder ra oracle gap. Ràng buộc không chồng nhau bỏ sót tổ hợp chồng hợp lệ như
crossfade lồng transform, nên kết quả là cận dưới của gap; báo cáo kèm ghi chú này. Gap dưới 10% thì đóng
câu hỏi ladder và dòng bác bỏ tối ưu toàn cục giữ nguyên bằng số; từ 10% trở lên thì đề xuất best-among-passing
và mở S3.1b.

Metric tầng hai của R1 (`representation_confusion`, `missing_primitive_attribution`) và oracle gap của R3
cũng đổ vào checkpoint này bên cạnh các bucket trên.

---

### 12.9 Sidecar `ILayerBoundaryRefiner` (có gate)

Chỉ xây sau M2.5b, khi M3.5 cho bucket `layer-edge` ≥ 20% bytes hoặc verify fail tập trung ở biên layer
trên 2D animation. Tiến trình Python riêng (`tools/aurbey-sidecar`), vào là thư mục frame cộng support mask theo
thời gian, ra là alpha 8-bit trong dải biên theo thời gian; lõi .NET đọc file và coi kết quả là *đề
xuất* (I4), vẫn verify. Model chọn tại thời điểm gate mở (MatAnyone, SAM 2 hoặc SAM 3 cộng matting, hay
thứ kế tiếp): contract không nêu tên model. Không có GPU hay không có sidecar thì pipeline chạy classical
không đổi output. Chi phí ước lượng khi phải chạy per-object per-frame: cỡ 24 đến 47 fps mỗi object trên
A100 cho SAM 2, chậm 3 đến 10 lần trên GPU tiêu dùng; A5 đo thật trước khi nhận.

### 12.10 Concept còn lại (chỉ ý tưởng; mỗi mục sẽ có plan riêng khi tới lượt)

| Hướng | Ý tưởng | Khả thi | Điều kiện để bắt đầu |
| --- | --- | --- | --- |
| **R4 loop synthesis** | Autocorrelation ra `L` hoặc `Animation` LoopForever. | Khả thi. | R1 hoặc attribution thấy chu kỳ phổ biến. |
| **Panorama T6 cho pan dài** | Strip ≤ 1024 chung một trajectory (`direction.md` 5.5). | Khả thi. | Bucket `pan-reveal` ≥ 25% ở M3.5. |
| **Perspective strips** | Chia strip, mỗi strip một similarity. | Khả thi. | Bucket `perspective-residual` ≥ 25%. |
| **Lookahead** | Encoder trễ K frame để thấy tương lai (fade-in, chọn nội dung sáng nhất cho patch). | Khả thi; `FrameWindow` đã có. Tốn RAM theo K. | Attribution thấy nhiều patch lặp trong fade hoặc chuyển tiếp chậm. |
| **Tier vàng client thật** | 5 đến 10 case chụp từ osu!lazer để ghim renderer. | Khả thi nhưng cần thao tác tay của người dùng. | Khi golden giải tích không còn đủ, ví dụ nghi ngờ lệch mipmap. |
| **Nhiều video dùng chung decode** | API nhận nhiều `VideoAnimation` cùng file và cùng cửa sổ, decode một lần, N encoder. | Khả thi; RAM tỉ lệ N × cửa sổ. | Có consumer thật cần. |
| **Nguồn frame tùy biến** | Overload public nhận `IAsyncEnumerable` frame RGB24 kèm metadata, bỏ phụ thuộc ffmpeg. | Khả thi; encoder đã tách khỏi ffmpeg. | Có consumer thật cần. |
| **Memo toàn kết quả** | Compile lại cùng (nội dung video, tham số) trả ngay kết quả cũ mà không decode. | Khả thi. Cần key cho input và một nơi lưu object ngoài asset, nên phải cân nhắc với D9. | Người dùng xác nhận cần "compile lại tức thì". |
| **Đóng gói NuGet** | `PackageId`, versioning, README trong gói. | Dễ. | Khi người dùng muốn phát hành. |

---

## Phụ lục A: sự thật bên ngoài và trạng thái xác minh

**osu!lazer** [deepwiki, `ppy/osu`, cần golden client thật để khẳng định]:

- `LegacyStoryboardDecoder` chỉ đọc cặp giá trị đầu của mỗi command, bỏ qua các giá trị chaining.
- `DrawableStoryboardSprite.DrawScale = base.DrawScale (có flip) × VectorScale`, tức S nhân V.
- Trong khoảng trống giữa hai command cùng loại, property giữ `EndValue` của command trước.

**OsuParsers** [deepwiki, `mrflashstudio/OsuParsers`, **A0 phải xác nhận trên 1.7.2**]:

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
  `Save(string path)`. Encoder công khai hay không thì chưa rõ; A0 sẽ trả lời.
- `StoryboardDecoder.Decode(string path | IEnumerable<string> | Stream)`. Không thread-safe (state
  static).

**FFMpegCore 5.x** [context7]: `FFOptions` được truyền theo từng lần gọi qua
`ProcessAsynchronously(throwOnError, ffOptions)` và qua mọi method của `FFProbe`. Không cần
`GlobalFFOptions`.

Kết quả A0: *(agent điền sau khi chạy A0)*.
