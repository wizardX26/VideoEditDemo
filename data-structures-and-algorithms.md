# Data Structures & Algorithms — Video Editor iOS

> Thiết kế chi tiết cấu trúc dữ liệu, thuật toán, và implementation guide.  
> Dựa trên phân tích kỹ thuật từ `video-editor-architecture.md` và `swiftui-vs-uikit-video-editor.md`.

---

## Mục lục

1. [Core Data Structures](#1-core-data-structures)
2. [Derived Types & Supporting Structures](#2-derived-types--supporting-structures)
3. [Key Algorithms](#3-key-algorithms)
4. [Editor State Machine](#4-editor-state-machine)
5. [Concurrency Model](#5-concurrency-model)
6. [Memory & Performance](#6-memory--performance)
7. [Edge Cases & Invariants](#7-edge-cases--invariants)
8. [File & Module Structure](#8-file--module-structure)

---

## 1. Core Data Structures

### 1.1 `MediaAsset` — Wrapper quanh AVAsset

`AVAsset` là lazy-loading object — nhiều thuộc tính (duration, tracks, naturalSize) cần async load riêng. `MediaAsset` wrap AVAsset và cache những thứ đã được resolve.

```swift
struct MediaAsset: Identifiable, Equatable {
    let id: UUID
    let url: URL
    let avAsset: AVAsset

    // Cached sau khi load — không truy cập trực tiếp từ AVAsset
    let duration: CMTime
    let naturalSize: CGSize           // sau khi apply preferredTransform
    let preferredTransform: CGAffineTransform
    let hasAudioTrack: Bool
    let nominalFrameRate: Float       // 24, 29.97, 30, 60...
    let videoCodec: CMFormatDescription? // H.264 vs H.265
}

extension MediaAsset {
    // Factory — load async, cache synchronously
    static func load(from url: URL) async throws -> MediaAsset {
        let asset = AVURLAsset(url: url)

        // Load tất cả cùng lúc — tránh nhiều round trip
        let (duration, tracks) = try await (
            asset.load(.duration),
            asset.loadTracks(withMediaType: .video)
        )

        guard let videoTrack = tracks.first else {
            throw MediaError.noVideoTrack
        }

        let (naturalSize, transform, frameRate, formatDescriptions) = try await (
            videoTrack.load(.naturalSize),
            videoTrack.load(.preferredTransform),
            videoTrack.load(.nominalFrameRate),
            videoTrack.load(.formatDescriptions)
        )

        // Tính corrected size sau orientation
        let corrected = naturalSize.applying(transform)
        let correctedSize = CGSize(
            width: abs(corrected.width),
            height: abs(corrected.height)
        )

        let audioTracks = try await asset.loadTracks(withMediaType: .audio)

        return MediaAsset(
            id: UUID(),
            url: url,
            avAsset: asset,
            duration: duration,
            naturalSize: correctedSize,
            preferredTransform: transform,
            hasAudioTrack: !audioTracks.isEmpty,
            nominalFrameRate: frameRate,
            videoCodec: formatDescriptions.first
        )
    }
}
```

**Tại sao không dùng `AVAsset` trực tiếp:**
- `AVAsset.duration`, `.tracks` là synchronous deprecated APIs từ iOS 16
- Async load tập trung một chỗ giúp error handling rõ ràng
- `naturalSize` raw không reflect orientation — phải apply `preferredTransform` trước

---

### 1.2 `Clip` — Unit of Edit

`Clip` là value type. Mọi edit (trim, crop, filter) tạo ra một `Clip` mới thay vì mutate in-place — đây là nền tảng của undo/redo.

```swift
struct Clip: Identifiable, Equatable {
    let id: UUID
    let asset: MediaAsset

    // --- Trim ---
    // Luôn nằm trong [.zero, asset.duration]
    // Invariant: trimRange.duration > CMTime.zero
    var trimRange: CMTimeRange

    // --- Crop ---
    // Normalized [0, 1] space. origin = top-left.
    // Invariant: origin.x + size.width  <= 1.0
    // Invariant: origin.y + size.height <= 1.0
    // Invariant: size.width > 0.05, size.height > 0.05 (min 5%)
    var cropRect: CGRect

    // --- Color grading ---
    var adjustments: AdjustmentParams
    var filter: FilterType

    // --- Computed ---
    var duration: CMTime { trimRange.duration }
}

extension Clip {
    // Default clip từ asset — toàn bộ duration, không crop, không filter
    static func from(_ asset: MediaAsset) -> Clip {
        Clip(
            id: UUID(),
            asset: asset,
            trimRange: CMTimeRange(start: .zero, duration: asset.duration),
            cropRect: CGRect(x: 0, y: 0, width: 1, height: 1),
            adjustments: .identity,
            filter: .none
        )
    }

    // Validate invariants — gọi sau mọi mutation
    var isValid: Bool {
        let minDuration = CMTime(value: 1, timescale: 600) // ~1 frame
        guard trimRange.duration >= minDuration else { return false }
        guard trimRange.start >= .zero else { return false }
        guard (trimRange.start + trimRange.duration) <= asset.duration else { return false }
        guard cropRect.origin.x >= 0, cropRect.origin.y >= 0 else { return false }
        guard cropRect.origin.x + cropRect.size.width  <= 1.0 else { return false }
        guard cropRect.origin.y + cropRect.size.height <= 1.0 else { return false }
        guard cropRect.size.width >= 0.05, cropRect.size.height >= 0.05 else { return false }
        return true
    }
}
```

---

### 1.3 `AdjustmentParams` & `FilterType`

```swift
struct AdjustmentParams: Equatable {
    var brightness: Float  // -1.0 → 1.0, identity = 0.0
    var contrast: Float    //  0.0 → 2.0, identity = 1.0
    var saturation: Float  //  0.0 → 2.0, identity = 1.0

    static let identity = AdjustmentParams(
        brightness: 0.0,
        contrast: 1.0,
        saturation: 1.0
    )

    // Không có effect nào đáng kể
    var isIdentity: Bool {
        abs(brightness) < 0.001
        && abs(contrast  - 1.0) < 0.001
        && abs(saturation - 1.0) < 0.001
    }
}

enum FilterType: String, CaseIterable, Equatable {
    case none
    case blackAndWhite
    case sepia
    case noir
    case fade

    // CIFilter name tương ứng — dùng khi apply
    var ciFilterName: String? {
        switch self {
        case .none:         return nil
        case .blackAndWhite: return "CIPhotoEffectMono"
        case .sepia:         return "CISepiaTone"
        case .noir:          return "CIPhotoEffectNoir"
        case .fade:          return "CIPhotoEffectFade"
        }
    }

    var displayName: String {
        switch self {
        case .none:          return "Original"
        case .blackAndWhite: return "B&W"
        case .sepia:         return "Sepia"
        case .noir:          return "Noir"
        case .fade:          return "Fade"
        }
    }
}
```

---

### 1.4 `Timeline` — Toàn bộ trạng thái edit

```swift
struct Timeline: Equatable {
    var clips: [Clip]
    var aspectRatio: AspectRatio
    var outputFPS: Float  // 30 hoặc 60

    // Computed
    var outputResolution: CGSize { aspectRatio.resolution }
    var totalDuration: CMTime {
        clips.reduce(CMTime.zero) { $0 + $1.duration }
    }
    var isEmpty: Bool { clips.isEmpty }
}

enum AspectRatio: String, CaseIterable, Equatable {
    case widescreen = "16:9"
    case square     = "1:1"
    case portrait   = "9:16"
    case standard   = "4:3"

    var resolution: CGSize {
        switch self {
        case .widescreen: return CGSize(width: 1920, height: 1080)
        case .square:     return CGSize(width: 1080, height: 1080)
        case .portrait:   return CGSize(width: 1080, height: 1920)
        case .standard:   return CGSize(width: 1440, height: 1080)
        }
    }

    var ratio: CGFloat {
        resolution.width / resolution.height
    }
}
```

---

### 1.5 `EditCommand` — Undo/Redo Stack

Dùng **Command Pattern** với value-type snapshot. Mỗi edit snapshot toàn bộ `Timeline` — đơn giản, an toàn với value semantics của Swift.

```swift
// Toàn bộ undo/redo state chỉ là một mảng Timeline snapshots
struct EditHistory {
    private var past: [Timeline]    // [oldest ... newest]
    private var future: [Timeline]  // [newest ... oldest] — reversed for O(1) pop
    private let maxHistory = 30

    var current: Timeline

    init(initial: Timeline) {
        self.current = initial
        self.past    = []
        self.future  = []
    }

    // Gọi trước mỗi mutation
    mutating func push(_ newState: Timeline) {
        past.append(current)
        if past.count > maxHistory {
            past.removeFirst()
        }
        current = newState
        future.removeAll()  // clear redo stack sau edit mới
    }

    var canUndo: Bool { !past.isEmpty }
    var canRedo: Bool { !future.isEmpty }

    mutating func undo() {
        guard let prev = past.popLast() else { return }
        future.append(current)
        current = prev
    }

    mutating func redo() {
        guard let next = future.popLast() else { return }
        past.append(current)
        current = next
    }
}
```

**Tại sao snapshot thay vì delta commands:**
- `Timeline` là value type, copy rẻ (struct, không heap)
- `AVAsset` bên trong `Clip` không được copy — chỉ reference được share, không duplicate data thật
- Không cần implement inverse của từng command — đơn giản hơn nhiều
- 30 snapshots × ~1KB metadata = ~30KB RAM — không đáng kể

---

### 1.6 `RenderParams` — Per-frame Render Instruction

`RenderParams` là derived type, tính từ `Clip`. Renderer nhận `RenderParams`, không nhận `Clip` trực tiếp — tách biệt data model khỏi rendering layer.

```swift
struct RenderParams: Equatable {
    var cropOrigin: SIMD2<Float>    // top-left của crop rect, normalized
    var cropSize:   SIMD2<Float>    // size của crop rect, normalized
    var brightness: Float
    var contrast:   Float
    var saturation: Float
    var filterType: FilterType
    var outputSize: CGSize

    static func from(_ clip: Clip, outputSize: CGSize) -> RenderParams {
        RenderParams(
            cropOrigin: SIMD2<Float>(
                Float(clip.cropRect.origin.x),
                Float(clip.cropRect.origin.y)
            ),
            cropSize: SIMD2<Float>(
                Float(clip.cropRect.size.width),
                Float(clip.cropRect.size.height)
            ),
            brightness: clip.adjustments.brightness,
            contrast:   clip.adjustments.contrast,
            saturation: clip.adjustments.saturation,
            filterType: clip.filter,
            outputSize: outputSize
        )
    }

    // Identity params — passthrough, không thay đổi frame
    static let identity = RenderParams(
        cropOrigin: .zero,
        cropSize:   SIMD2<Float>(1, 1),
        brightness: 0.0,
        contrast:   1.0,
        saturation: 1.0,
        filterType: .none,
        outputSize: CGSize(width: 1920, height: 1080)
    )
}
```

---

### 1.7 `ExportConfig`

```swift
struct ExportConfig {
    var outputURL: URL
    var fileType: AVFileType       // .mp4
    var videoCodec: AVVideoCodecType  // .h264 (tương thích rộng)
    var videoBitrate: Int?         // nil = auto
    var outputResolution: CGSize
    var outputFPS: Float

    static func `default`(outputURL: URL, timeline: Timeline) -> ExportConfig {
        ExportConfig(
            outputURL: outputURL,
            fileType: .mp4,
            videoCodec: .h264,
            videoBitrate: nil,
            outputResolution: timeline.outputResolution,
            outputFPS: timeline.outputFPS
        )
    }
}
```

---

## 2. Derived Types & Supporting Structures

### 2.1 `ClipOffset` — Timeline Position Mapping

```swift
// Map từ global timeline time → clip index + local time trong clip
struct ClipOffset {
    let clipIndex: Int
    let localTime: CMTime       // time trong source asset (đã apply trimRange.start)
    let globalStart: CMTime     // thời điểm clip này bắt đầu trong timeline
}

extension Timeline {
    // O(n) — n = số clips. Acceptable vì n thường < 20
    func clipOffset(at globalTime: CMTime) -> ClipOffset? {
        var cursor = CMTime.zero
        for (i, clip) in clips.enumerated() {
            let end = cursor + clip.duration
            if globalTime < end || i == clips.count - 1 {
                let localOffset = globalTime - cursor
                let sourceTime  = clip.trimRange.start + localOffset
                return ClipOffset(
                    clipIndex: i,
                    localTime: sourceTime,
                    globalStart: cursor
                )
            }
            cursor = end
        }
        return nil
    }

    // Tính global start time của từng clip — dùng khi build AVMutableComposition
    func globalStartTimes() -> [CMTime] {
        var times: [CMTime] = []
        var cursor = CMTime.zero
        for clip in clips {
            times.append(cursor)
            cursor = cursor + clip.duration
        }
        return times
    }
}
```

---

### 2.2 `ThumbnailCache`

```swift
// Thread-safe thumbnail cache dùng NSCache (auto-evict dưới memory pressure)
final class ThumbnailCache {
    private let cache = NSCache<NSString, UIImage>()

    init() {
        cache.countLimit = 100       // tối đa 100 thumbnails
        cache.totalCostLimit = 50 * 1024 * 1024  // 50 MB
    }

    func thumbnail(for clipID: UUID, time: CMTime) -> UIImage? {
        cache.object(forKey: key(clipID, time))
    }

    func store(_ image: UIImage, for clipID: UUID, time: CMTime) {
        let cost = Int(image.size.width * image.size.height * 4) // RGBA bytes
        cache.setObject(image, forKey: key(clipID, time), cost: cost)
    }

    private func key(_ id: UUID, _ time: CMTime) -> NSString {
        "\(id.uuidString)-\(time.value)-\(time.timescale)" as NSString
    }
}
```

---

## 3. Key Algorithms

### 3.1 Timeline → CMTime Mapping

Bài toán: khi AVPlayer ở `globalTime`, cần biết đang ở clip nào và frame nào của clip đó.

```
Timeline: [Clip A (5s)] [Clip B (3s)] [Clip C (7s)]
Global:    0─────────5  5────────8   8──────────15
```

```swift
// Ví dụ: globalTime = 6.5s
// → cursor sau Clip A = 5s
// → 6.5 < 5+3 = 8 → đang ở Clip B
// → localOffset = 6.5 - 5.0 = 1.5s
// → sourceTime = clip.trimRange.start + 1.5s
```

**Complexity:** O(n) với n = số clips. Với n < 20 (thực tế), không cần optimize.

**Tại sao không dùng binary search:** Clips có thể bị xóa, reorder, thêm vào — maintain sorted array thêm complexity không cần thiết cho n nhỏ.

---

### 3.2 Crop Rect Normalization & Aspect Ratio Constraint

```swift
struct CropAlgorithm {

    // Clamp crop rect vào valid bounds [0,1]
    static func clamp(_ rect: CGRect) -> CGRect {
        let x = max(0, min(rect.origin.x, 1 - rect.size.width))
        let y = max(0, min(rect.origin.y, 1 - rect.size.height))
        let w = max(0.05, min(rect.size.width,  1 - x))
        let h = max(0.05, min(rect.size.height, 1 - y))
        return CGRect(x: x, y: y, width: w, height: h)
    }

    // Constrain crop rect theo target aspect ratio
    // Giữ center cố định, điều chỉnh width hoặc height
    static func constrain(_ rect: CGRect, to ratio: CGFloat) -> CGRect {
        let currentRatio = rect.width / rect.height
        var result = rect

        if currentRatio > ratio {
            // Quá rộng → giảm width
            result.size.width = result.size.height * ratio
        } else {
            // Quá cao → giảm height
            result.size.height = result.size.width / ratio
        }

        // Re-center theo midpoint của rect gốc
        result.origin.x = rect.midX - result.size.width  / 2
        result.origin.y = rect.midY - result.size.height / 2

        return clamp(result)
    }

    // Apply pinch gesture: scale crop rect quanh gesture center
    // scale > 1: zoom in → crop rect nhỏ hơn (lấy vùng nhỏ hơn)
    // scale < 1: zoom out → crop rect lớn hơn
    static func applyPinch(
        current: CGRect,
        scale: CGFloat,
        anchor: CGPoint,           // normalized [0,1] trong view
        aspectRatioConstraint: CGFloat?
    ) -> CGRect {
        // Anchor trong crop-rect space
        let anchorInCrop = CGPoint(
            x: current.origin.x + anchor.x * current.size.width,
            y: current.origin.y + anchor.y * current.size.height
        )

        let newWidth  = current.size.width  / scale
        let newHeight = current.size.height / scale

        // Giữ anchor point cố định
        var result = CGRect(
            x: anchorInCrop.x - anchor.x * newWidth,
            y: anchorInCrop.y - anchor.y * newHeight,
            width: newWidth,
            height: newHeight
        )

        if let ratio = aspectRatioConstraint {
            result = constrain(result, to: ratio)
        } else {
            result = clamp(result)
        }

        return result
    }

    // Apply pan gesture: translate crop rect
    static func applyPan(
        current: CGRect,
        delta: CGVector            // normalized delta trong view space
    ) -> CGRect {
        var result = current
        result.origin.x += delta.dx * current.size.width
        result.origin.y += delta.dy * current.size.height
        return clamp(result)
    }
}
```

**Lưu ý anchor point:** Khi pinch zoom, user expect vùng dưới ngón tay không bị dịch chuyển. Algorithm tính `anchorInCrop` (điểm trong crop rect tương ứng gesture center) và giữ cố định trong khi scale.

---

### 3.3 Fit Transform — naturalSize → outputResolution

Vấn đề: video từ Photos có thể là 4K portrait (2160×3840), output là 1080×1920. Cần tính `CGAffineTransform` để:
1. Correct orientation (apply `preferredTransform`)
2. Scale để fit/fill output
3. Center trong output frame

```swift
enum FitMode {
    case fit   // toàn bộ video nằm trong output, có letterbox
    case fill  // video fill toàn bộ output, crop nếu cần
}

struct TransformAlgorithm {

    static func fitTransform(
        naturalSize: CGSize,
        preferredTransform: CGAffineTransform,
        outputSize: CGSize,
        mode: FitMode = .fit
    ) -> CGAffineTransform {

        // Bước 1: Tính corrected size sau orientation
        let transformed  = naturalSize.applying(preferredTransform)
        let correctedSize = CGSize(
            width:  abs(transformed.width),
            height: abs(transformed.height)
        )

        // Bước 2: Scale factor
        let scaleX = outputSize.width  / correctedSize.width
        let scaleY = outputSize.height / correctedSize.height
        let scale: CGFloat = mode == .fit ? min(scaleX, scaleY) : max(scaleX, scaleY)

        // Bước 3: Translation để center
        let scaledWidth  = correctedSize.width  * scale
        let scaledHeight = correctedSize.height * scale
        let tx = (outputSize.width  - scaledWidth)  / 2
        let ty = (outputSize.height - scaledHeight) / 2

        // Compose: preferredTransform → scale → translate
        // Thứ tự quan trọng — CGAffineTransform multiply là right-to-left
        var transform = preferredTransform
        transform = transform.scaledBy(x: scale, y: scale)

        // Translation cần adjust vì preferredTransform có thể có rotation
        // Tính translation trong output space
        let originAfterTransform = CGPoint.zero.applying(preferredTransform).applying(
            CGAffineTransform(scaleX: scale, y: scale)
        )
        transform.tx = tx - originAfterTransform.x
        transform.ty = ty - originAfterTransform.y

        return transform
    }
}
```

**Edge case quan trọng:** `CGAffineTransform` của video portrait quay từ iPhone thường là:
```
[ 0, -1, 0 ]
[ 1,  0, 0 ]   // rotation 90°
[ 0, w, 1 ]    // translation để compensate rotation origin
```
Sau khi apply transform, origin có thể là negative — phải tính lại translation trong output coordinate space.

---

### 3.4 Trim Seek Strategy

Có hai chế độ seek với tradeoff khác nhau:

```swift
enum SeekMode {
    case scrubbing   // user đang drag — ưu tiên speed, chấp nhận approximate
    case committed   // user thả tay — ưu tiên accuracy
}

final class TrimSeekController {
    private weak var player: AVPlayer?
    private var pendingSeek: CMTime?
    private var isSeeking = false

    // Gọi liên tục khi drag — debounce để tránh queue overflow
    func seek(to time: CMTime, mode: SeekMode) {
        pendingSeek = time

        guard !isSeeking else { return }  // ignore nếu seek đang chạy
        performSeek(mode: mode)
    }

    private func performSeek(mode: SeekMode) {
        guard let time = pendingSeek, let player = player else { return }
        pendingSeek = nil
        isSeeking = true

        let (before, after): (CMTime, CMTime) = switch mode {
        case .scrubbing:
            // Tolerance lớn = snap sang keyframe gần nhất = nhanh
            (CMTime(value: 2, timescale: 1), CMTime(value: 2, timescale: 1))
        case .committed:
            // Zero tolerance = frame accurate = chậm hơn nhưng đúng
            (.zero, CMTime(value: 1, timescale: 600))
        }

        player.seek(to: time, toleranceBefore: before, toleranceAfter: after) { [weak self] _ in
            self?.isSeeking = false
            // Nếu có pending seek mới trong lúc chờ, thực hiện tiếp
            if self?.pendingSeek != nil {
                self?.performSeek(mode: mode)
            }
        }
    }
}
```

**Tại sao cần debounce:** `AVPlayer.seek` callback-based. Nếu gọi 60 lần/giây khi drag, mỗi seek queue vào sau seek trước → lag tích lũy. Pattern trên chỉ giữ seek mới nhất, bỏ tất cả intermediate seeks.

---

### 3.5 Metal Shader — Filter + Adjustment trong Single Pass

Thay vì hai render pass riêng (filter pass → adjustment pass), chain trong một fragment shader duy nhất.

```metal
// CombinedShader.metal

#include <metal_stdlib>
using namespace metal;

struct VertexIn {
    float4 position [[attribute(0)]];
    float2 texCoord [[attribute(1)]];
};

struct VertexOut {
    float4 position [[position]];
    float2 texCoord;
};

struct RenderUniforms {
    float2 cropOrigin;
    float2 cropSize;
    float  brightness;
    float  contrast;
    float  saturation;
    int    filterType;  // 0=none, 1=bw, 2=sepia, 3=noir, 4=fade
};

// Sepia matrix constants
constant float3x3 sepiaMatrix = float3x3(
    float3(0.393, 0.349, 0.272),
    float3(0.769, 0.686, 0.534),
    float3(0.189, 0.168, 0.131)
);

float3 applyFilter(float3 rgb, int filterType) {
    if (filterType == 0) return rgb;  // none

    float luma = dot(rgb, float3(0.2126, 0.7152, 0.0722));

    if (filterType == 1) {  // black & white
        return float3(luma);
    }
    if (filterType == 2) {  // sepia
        return clamp(sepiaMatrix * rgb, 0.0, 1.0);
    }
    if (filterType == 3) {  // noir — high contrast B&W
        float noir = pow(luma, 1.6) * 1.1;
        return float3(clamp(noir, 0.0, 1.0));
    }
    if (filterType == 4) {  // fade — lift blacks, reduce whites
        return mix(float3(0.08), float3(0.85), rgb);
    }
    return rgb;
}

float3 applyAdjustment(float3 rgb, float brightness, float contrast, float saturation) {
    // 1. Brightness
    rgb = rgb + float3(brightness);

    // 2. Contrast — scale quanh midpoint 0.5
    rgb = (rgb - 0.5) * contrast + 0.5;

    // 3. Saturation — mix với luminance
    float luma = dot(rgb, float3(0.2126, 0.7152, 0.0722));
    rgb = mix(float3(luma), rgb, saturation);

    return clamp(rgb, 0.0, 1.0);
}

vertex VertexOut vertexShader(
    VertexIn in [[stage_in]]
) {
    VertexOut out;
    out.position = in.position;
    out.texCoord = in.texCoord;
    return out;
}

fragment float4 fragmentShader(
    VertexOut in            [[stage_in]],
    texture2d<float> tex    [[texture(0)]],
    constant RenderUniforms& u [[buffer(0)]]
) {
    constexpr sampler s(address::clamp_to_edge, filter::linear);

    // Step 1: Crop — remap UV vào crop region
    float2 uv = u.cropOrigin + in.texCoord * u.cropSize;
    float4 color = tex.sample(s, uv);

    // Step 2: Filter
    float3 rgb = applyFilter(color.rgb, u.filterType);

    // Step 3: Adjustment
    rgb = applyAdjustment(rgb, u.brightness, u.contrast, u.saturation);

    return float4(rgb, color.a);
}
```

**Tại sao single pass quan trọng:**
- Mỗi render pass = 1 texture write + 1 texture read = bandwidth overhead
- Với 30fps preview, 2 pass = 60 texture ops/giây thay vì 30
- Single pass không ảnh hưởng chất lượng (toán học tương đương)

---

### 3.6 Export Frame Loop

Đây là thuật toán trung tâm của export — xử lý tuần tự từng frame.

```swift
final class ExportPipeline {

    func export(
        timeline: Timeline,
        config: ExportConfig,
        renderer: MetalFrameRenderer,
        onProgress: @escaping (Float) -> Void
    ) async throws {

        // Bước 1: Build AVMutableComposition từ tất cả clips
        let (composition, videoComposition) = try buildComposition(from: timeline)

        // Bước 2: Setup AVAssetWriter
        let writer = try AVAssetWriter(outputURL: config.outputURL, fileType: config.fileType)

        let videoSettings: [String: Any] = [
            AVVideoCodecKey: config.videoCodec,
            AVVideoWidthKey:  config.outputResolution.width,
            AVVideoHeightKey: config.outputResolution.height,
        ]
        let videoInput = AVAssetWriterInput(mediaType: .video, outputSettings: videoSettings)
        videoInput.expectsMediaDataInRealTime = false

        let adaptor = AVAssetWriterInputPixelBufferAdaptor(
            assetWriterInput: videoInput,
            sourcePixelBufferAttributes: pixelBufferAttributes(config.outputResolution)
        )
        writer.add(videoInput)

        // Bước 3: Setup AVAssetReader
        let reader = try AVAssetReader(asset: composition)
        let readerOutput = AVAssetReaderVideoCompositionOutput(
            videoTracks: composition.tracks(withMediaType: .video),
            videoSettings: [kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA]
        )
        readerOutput.videoComposition = videoComposition
        reader.add(readerOutput)

        // Bước 4: Audio passthrough
        let audioInput = setupAudioInput(writer: writer)
        let audioOutput = setupAudioOutput(reader: reader, composition: composition)

        // Bước 5: Start
        reader.startReading()
        writer.startWriting()
        writer.startSession(atSourceTime: .zero)

        let totalSeconds = CMTimeGetSeconds(timeline.totalDuration)

        // Bước 6: Frame loop — video
        // Chạy trên background queue để không block main thread
        try await withThrowingTaskGroup(of: Void.self) { group in

            // Video track
            group.addTask {
                let queue = DispatchQueue(label: "export.video")
                try await withCheckedThrowingContinuation { continuation in
                    videoInput.requestMediaDataWhenReady(on: queue) {
                        while videoInput.isReadyForMoreMediaData {
                            // Đọc next decoded frame
                            guard let sample = readerOutput.copyNextSampleBuffer() else {
                                videoInput.markAsFinished()
                                continuation.resume()
                                return
                            }

                            let pts = CMSampleBufferGetPresentationTimeStamp(sample)
                            guard let pixelBuffer = CMSampleBufferGetImageBuffer(sample) else {
                                continue
                            }

                            // Tìm RenderParams cho frame này
                            let params = self.renderParams(
                                at: pts,
                                timeline: timeline,
                                outputSize: config.outputResolution
                            )

                            // Apply Metal render
                            if let outputBuffer = renderer.render(pixelBuffer, params: params) {
                                adaptor.append(outputBuffer, withPresentationTime: pts)
                            }

                            // Progress
                            let elapsed = CMTimeGetSeconds(pts)
                            onProgress(Float(elapsed / totalSeconds))
                        }
                    }
                }
            }

            // Audio track (passthrough — không re-encode)
            group.addTask {
                let queue = DispatchQueue(label: "export.audio")
                try await withCheckedThrowingContinuation { continuation in
                    audioInput?.requestMediaDataWhenReady(on: queue) {
                        while audioInput?.isReadyForMoreMediaData == true {
                            guard let sample = audioOutput?.copyNextSampleBuffer() else {
                                audioInput?.markAsFinished()
                                continuation.resume()
                                return
                            }
                            audioInput?.append(sample)
                        }
                    }
                }
            }

            try await group.waitForAll()
        }

        // Bước 7: Finish
        await writer.finishWriting()

        guard writer.status == .completed else {
            throw writer.error ?? ExportError.writeFailed
        }
    }

    // Tìm RenderParams tại một global timestamp
    private func renderParams(
        at globalTime: CMTime,
        timeline: Timeline,
        outputSize: CGSize
    ) -> RenderParams {
        guard let offset = timeline.clipOffset(at: globalTime) else {
            return .identity
        }
        let clip = timeline.clips[offset.clipIndex]
        return RenderParams.from(clip, outputSize: outputSize)
    }

    private func pixelBufferAttributes(_ size: CGSize) -> [String: Any] {
        [
            kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA,
            kCVPixelBufferWidthKey  as String: Int(size.width),
            kCVPixelBufferHeightKey as String: Int(size.height),
            kCVPixelBufferIOSurfacePropertiesKey as String: [:]  // GPU-accessible
        ]
    }
}
```

**Vấn đề quan trọng — `renderParams(at:)` và clip boundary:**

Khi export, `AVAssetReaderVideoCompositionOutput` trả về frames trong global timeline time. Frame tại `pts = 5.000s` có thể là frame cuối của Clip A hoặc frame đầu của Clip B tùy rounding. Cần logic `clipOffset(at:)` robust để handle boundary chính xác.

```swift
// Trong clipOffset — handle boundary: frame đúng cuối clip thuộc clip đó
func clipOffset(at globalTime: CMTime) -> ClipOffset? {
    var cursor = CMTime.zero
    for (i, clip) in clips.enumerated() {
        let end = cursor + clip.duration
        // Strict less-than — frame đúng tại boundary thuộc clip TIẾP THEO
        // Ngoại trừ clip cuối
        let isLastClip = (i == clips.count - 1)
        if globalTime < end || isLastClip {
            let localOffset = CMTimeMinimum(globalTime - cursor, clip.duration)
            let sourceTime  = clip.trimRange.start + localOffset
            return ClipOffset(clipIndex: i, localTime: sourceTime, globalStart: cursor)
        }
        cursor = end
    }
    return nil
}
```

---

### 3.7 `AVMutableComposition` Build Algorithm

```swift
func buildComposition(
    from timeline: Timeline
) throws -> (AVMutableComposition, AVMutableVideoComposition) {

    let composition = AVMutableComposition()

    guard let videoTrack = composition.addMutableTrack(
        withMediaType: .video,
        preferredTrackID: kCMPersistentTrackID_Invalid
    ) else { throw CompositionError.trackCreationFailed }

    let audioTrack = composition.addMutableTrack(
        withMediaType: .audio,
        preferredTrackID: kCMPersistentTrackID_Invalid
    )

    var insertTime = CMTime.zero
    var layerInstructions: [(AVMutableVideoCompositionLayerInstruction, CMTimeRange)] = []

    for clip in timeline.clips {
        let asset = clip.asset.avAsset
        let assetVideoTracks = try await asset.loadTracks(withMediaType: .video)

        guard let sourceVideoTrack = assetVideoTracks.first else {
            throw CompositionError.noVideoTrack(clipID: clip.id)
        }

        // Insert video segment
        try videoTrack.insertTimeRange(
            clip.trimRange,
            of: sourceVideoTrack,
            at: insertTime
        )

        // Insert audio (skip nếu không có — composition tự fill silence)
        if clip.asset.hasAudioTrack {
            let audioTracks = try await asset.loadTracks(withMediaType: .audio)
            if let sourceAudioTrack = audioTracks.first {
                try? audioTrack?.insertTimeRange(
                    clip.trimRange,
                    of: sourceAudioTrack,
                    at: insertTime
                )
            }
        }

        // Build layer instruction cho clip này
        let instruction = AVMutableVideoCompositionLayerInstruction(
            assetTrack: videoTrack
        )
        let transform = TransformAlgorithm.fitTransform(
            naturalSize: clip.asset.naturalSize,
            preferredTransform: clip.asset.preferredTransform,
            outputSize: timeline.outputResolution,
            mode: .fit
        )
        // Set transform chỉ trong thời gian của clip này
        instruction.setTransform(transform, at: insertTime)

        layerInstructions.append(
            (instruction, CMTimeRange(start: insertTime, duration: clip.duration))
        )

        insertTime = insertTime + clip.duration
    }

    // Build AVMutableVideoComposition
    // Một instruction cho mỗi clip, chứa transform tương ứng
    let compositionInstructions = layerInstructions.map { (layerInstr, timeRange) -> AVMutableVideoCompositionInstruction in
        let instr = AVMutableVideoCompositionInstruction()
        instr.timeRange = timeRange
        instr.layerInstructions = [layerInstr]
        return instr
    }

    let videoComposition = AVMutableVideoComposition()
    videoComposition.instructions = compositionInstructions
    videoComposition.renderSize   = timeline.outputResolution
    videoComposition.frameDuration = CMTime(
        value: 1,
        timescale: CMTimeScale(timeline.outputFPS)
    )

    return (composition, videoComposition)
}
```

---

## 4. Editor State Machine

### 4.1 `EditorState` Enum

```swift
enum EditorState: Equatable {
    case idle
    case loadingAsset(URL)
    case editing(selectedClipIndex: Int, activePanel: EditPanel)
    case exporting(progress: Float)
    case exportComplete(outputURL: URL)
    case error(EditorError)
}

enum EditPanel: Equatable {
    case none
    case trim
    case crop
    case adjustment
    case filter
    case ratio
    case concatenate
}

enum EditorError: Error, Equatable {
    case assetLoadFailed(String)
    case compositionFailed(String)
    case exportFailed(String)
    case invalidClip(UUID)
}
```

### 4.2 State Transitions

```
idle
  ──[selectVideo]──► loadingAsset
                          │
                    [loadSuccess]──► editing(panel: .none)
                          │
                    [loadFailed] ──► error
                          
editing
  ──[selectTrim]    ──► editing(panel: .trim)
  ──[selectCrop]    ──► editing(panel: .crop)
  ──[selectFilter]  ──► editing(panel: .filter)
  ──[selectAdj]     ──► editing(panel: .adjustment)
  ──[selectRatio]   ──► editing(panel: .ratio)
  ──[addClip]       ──► editing(panel: .concatenate)
  ──[tapExport]     ──► exporting(progress: 0)

exporting
  ──[progressUpdate]──► exporting(progress: n)
  ──[exportDone]    ──► exportComplete(url)
  ──[exportFailed]  ──► error

exportComplete
  ──[previewDone]   ──► editing (về lại)
  ──[saveToPhotos]  ──► idle
```

### 4.3 `EditorViewModel` — Owns State Machine

```swift
@MainActor
final class EditorViewModel: ObservableObject {
    @Published private(set) var state: EditorState = .idle
    @Published private(set) var history: EditHistory

    private let renderer: MetalFrameRenderer
    private let exportPipeline: ExportPipeline
    private let thumbnailCache = ThumbnailCache()

    var timeline: Timeline { history.current }
    var canUndo: Bool { history.canUndo }
    var canRedo: Bool { history.canRedo }

    // State transitions — gọi từ UI
    func selectVideo(_ url: URL) {
        state = .loadingAsset(url)
        Task {
            do {
                let asset = try await MediaAsset.load(from: url)
                let clip  = Clip.from(asset)
                let newTimeline = Timeline(
                    clips: [clip],
                    aspectRatio: .widescreen,
                    outputFPS: 30
                )
                history = EditHistory(initial: newTimeline)
                state = .editing(selectedClipIndex: 0, activePanel: .none)
            } catch {
                state = .error(.assetLoadFailed(error.localizedDescription))
            }
        }
    }

    // Mọi edit đều đi qua đây — tự động push vào history
    func applyEdit(_ transform: (inout Timeline) -> Void) {
        var newTimeline = timeline
        transform(&newTimeline)

        // Validate trước khi commit
        let allValid = newTimeline.clips.allSatisfy { $0.isValid }
        guard allValid else { return }

        history.push(newTimeline)
    }

    func undo() { history.undo() }
    func redo() { history.redo() }

    func startExport() {
        guard case .editing = state else { return }
        state = .exporting(progress: 0)

        Task {
            do {
                let outputURL = FileManager.default
                    .temporaryDirectory
                    .appendingPathComponent("\(UUID().uuidString).mp4")

                let config = ExportConfig.default(outputURL: outputURL, timeline: timeline)

                try await exportPipeline.export(
                    timeline: timeline,
                    config: config,
                    renderer: renderer
                ) { [weak self] progress in
                    Task { @MainActor in
                        self?.state = .exporting(progress: progress)
                    }
                }

                state = .exportComplete(outputURL: outputURL)
            } catch {
                state = .error(.exportFailed(error.localizedDescription))
            }
        }
    }
}
```

---

## 5. Concurrency Model

```
┌─────────────────────────────────────────────────────────┐
│                    Main Thread (UI)                     │
│  EditorViewModel, UIKit gesture, state updates,         │
│  AVPlayer control, CADisplayLink callback               │
└──────────────┬─────────────────────────┬────────────────┘
               │                         │
               ▼                         ▼
┌──────────────────────┐   ┌─────────────────────────────┐
│  Metal Command Queue │   │   Export DispatchQueue      │
│  (GPU thread)        │   │   (background, QoS: utility) │
│                      │   │                             │
│  MTLCommandBuffer    │   │  AVAssetReader.copyNext...  │
│  render per frame    │   │  MetalFrameRenderer.render  │
│  30-60 fps           │   │  AVAssetWriter.append       │
└──────────────────────┘   └─────────────────────────────┘
               │                         │
               ▼                         ▼
┌──────────────────────┐   ┌─────────────────────────────┐
│  Thumbnail Queue     │   │   Asset Load Queue          │
│  (background, QoS:   │   │   (async/await Task)        │
│   background)        │   │                             │
│  AVAssetImageGen     │   │  AVAsset.load(.duration)    │
│  NSCache write       │   │  AVAsset.loadTracks(...)    │
└──────────────────────┘   └─────────────────────────────┘
```

### Threading Rules

```swift
// Rule 1: State mutations chỉ trên Main thread
// Dùng @MainActor cho ViewModel

// Rule 2: Metal render có thể gọi từ bất kỳ thread nào
// MTLCommandQueue là thread-safe

// Rule 3: AVAssetReader/Writer trên background queue
// Dùng riêng cho mỗi export session (không share)

// Rule 4: CVPixelBuffer từ AVPlayerItemVideoOutput — main thread safe
// copyPixelBuffer(forItemTime:) gọi từ CADisplayLink callback (main)

// Rule 5: NSCache thread-safe — thumbnail read/write từ bất kỳ thread
```

### Preview Loop

```swift
final class PreviewController {
    private var displayLink: CADisplayLink?
    private var videoOutput: AVPlayerItemVideoOutput?
    private weak var mtkView: MTKView?
    private weak var renderer: MetalFrameRenderer?
    private var currentClipIndex: Int = 0

    func startPreview(player: AVPlayer, timeline: Timeline) {
        let output = AVPlayerItemVideoOutput(pixelBufferAttributes: [
            kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA
        ])
        player.currentItem?.add(output)
        videoOutput = output

        displayLink = CADisplayLink(target: self, selector: #selector(tick))
        displayLink?.preferredFrameRateRange = CAFrameRateRange(minimum: 30, maximum: 60)
        displayLink?.add(to: .main, forMode: .common)
    }

    @objc private func tick(_ link: CADisplayLink) {
        let hostTime = link.timestamp + link.duration  // next frame time
        let itemTime = videoOutput?.itemTime(forHostTime: hostTime) ?? .zero

        guard
            videoOutput?.hasNewPixelBuffer(forItemTime: itemTime) == true,
            let pixelBuffer = videoOutput?.copyPixelBuffer(
                forItemTime: itemTime,
                itemTimeForDisplay: nil
            )
        else { return }

        // Tìm clip và render params cho thời điểm này
        // (timeline được pass từ ngoài — không store ở đây)
        // ...
        renderer?.renderToView(pixelBuffer, params: currentParams, view: mtkView)
    }
}
```

---

## 6. Memory & Performance

### 6.1 `CVPixelBuffer` Pool — Tránh Alloc trong Export Loop

```swift
// Export loop alloc một pixel buffer mới mỗi frame → GC pressure cao
// Giải pháp: dùng pool từ adaptor

// Khi setup adaptor:
let adaptor = AVAssetWriterInputPixelBufferAdaptor(
    assetWriterInput: videoInput,
    sourcePixelBufferAttributes: [
        kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA,
        kCVPixelBufferWidthKey  as String: Int(outputSize.width),
        kCVPixelBufferHeightKey as String: Int(outputSize.height),
        kCVPixelBufferIOSurfacePropertiesKey as String: [:]
    ]
)

// Trong export loop — lấy buffer từ pool thay vì alloc mới:
func getPooledBuffer(from adaptor: AVAssetWriterInputPixelBufferAdaptor) -> CVPixelBuffer? {
    guard let pool = adaptor.pixelBufferPool else { return nil }
    var buffer: CVPixelBuffer?
    let status = CVPixelBufferPoolCreatePixelBuffer(nil, pool, &buffer)
    guard status == kCVReturnSuccess else { return nil }
    return buffer
}
```

### 6.2 `CVMetalTextureCache` — Tránh CPU Round-trip

```swift
final class MetalTextureCache {
    private var cache: CVMetalTextureCache?

    init(device: MTLDevice) {
        CVMetalTextureCacheCreate(nil, nil, device, nil, &cache)
    }

    // Convert CVPixelBuffer → MTLTexture mà không copy data về CPU
    // DMA trực tiếp GPU ↔ IOSurface
    func texture(from pixelBuffer: CVPixelBuffer) -> MTLTexture? {
        let width  = CVPixelBufferGetWidth(pixelBuffer)
        let height = CVPixelBufferGetHeight(pixelBuffer)

        var cvTexture: CVMetalTexture?
        let status = CVMetalTextureCacheCreateTextureFromImage(
            nil,
            cache!,
            pixelBuffer,
            nil,
            .bgra8Unorm,
            width, height,
            0,
            &cvTexture
        )

        guard status == kCVReturnSuccess, let cv = cvTexture else { return nil }
        return CVMetalTextureGetTexture(cv)

        // NOTE: MTLTexture này share memory với CVPixelBuffer
        // KHÔNG retain CVPixelBuffer sau khi texture hết scope
    }

    func flush() {
        CVMetalTextureCacheFlush(cache!, 0)
    }
}
```

**Tại sao quan trọng:** Nếu không dùng `CVMetalTextureCache`, cần copy pixel data từ GPU → CPU → GPU mỗi frame (2 copy/frame × 30fps × 8MB per 1080p frame = ~480MB bandwidth/giây). Với texture cache, zero copy — IOSurface được share trực tiếp.

### 6.3 Thumbnail Generation

```swift
final class ThumbnailGenerator {
    private let cache: ThumbnailCache
    private let queue = DispatchQueue(label: "thumbnail.gen", qos: .background)

    // Tạo thumbnails bất đồng bộ, cache kết quả
    func generateThumbnails(
        for clip: Clip,
        count: Int,
        size: CGSize
    ) async -> [CMTime: UIImage] {
        let asset = clip.asset.avAsset
        let generator = AVAssetImageGenerator(asset: asset)
        generator.appliesPreferredTrackTransform = true
        generator.maximumSize = size
        generator.requestedTimeToleranceBefore = CMTime(value: 1, timescale: 4)
        generator.requestedTimeToleranceAfter  = CMTime(value: 1, timescale: 4)

        // Tính các timestamp cần generate
        let duration = clip.trimRange.duration
        let interval = CMTimeMultiplyByFloat64(duration, multiplier: 1.0 / Double(count))
        let times = (0..<count).map { i -> CMTime in
            clip.trimRange.start + CMTimeMultiply(interval, multiplier: Int32(i))
        }

        var results: [CMTime: UIImage] = [:]

        // Batch generate — hiệu quả hơn từng cái một
        let values = times.map { NSValue(time: $0) }
        for await result in generator.images(for: values) {
            switch result {
            case let .success(requestedTime: reqTime, image: cgImage, actualTime: _):
                let image = UIImage(cgImage: cgImage)
                cache.store(image, for: clip.id, time: reqTime)
                results[reqTime] = image
            case .failure:
                break
            }
        }

        return results
    }
}
```

---

## 7. Edge Cases & Invariants

### 7.1 Empty Audio Track

```swift
// Khi clip không có audio, AVMutableCompositionTrack có gap (silence)
// KHÔNG cần handle thủ công — AVFoundation fill silence tự động

// Ngoại lệ: clip đầu không có audio nhưng clip sau có
// Composition track sẽ có silence từ 0 đến start của clip có audio → OK

// CHỈ cần skip insertTimeRange cho audio nếu clip không có track:
if clip.asset.hasAudioTrack {
    // insert audio
}
// Không insert gì cả nếu false — composition tự xử lý
```

### 7.2 Zero-duration Trim

```swift
// Guard trong applyEdit để prevent zero-duration clip
func applyTrim(_ range: CMTimeRange, to clipIndex: Int) {
    let minDuration = CMTime(value: 1, timescale: 600) // ~1 frame ở 60fps
    guard range.duration >= minDuration else { return }

    applyEdit { timeline in
        timeline.clips[clipIndex].trimRange = range
    }
}
```

### 7.3 Orientation Mismatch trong Concatenate

```swift
// Video portrait + video landscape trong cùng timeline
// FitTransform.fit → cả hai đều fit vào outputResolution với letterbox
// Kết quả: portrait video có pillarbox, landscape có letterbox → visually inconsistent

// Options:
// A) Luôn dùng .fill → crop phần thừa, không letterbox
// B) Normalize orientation theo clip đầu tiên khi add clip mới
// C) Warn user khi orientation mismatch (recommended cho 1 tuần)

func addClip(_ clip: Clip, to timeline: inout Timeline) {
    if let first = timeline.clips.first {
        let firstIsPortrait  = first.asset.naturalSize.height > first.asset.naturalSize.width
        let newIsPortrait    = clip.asset.naturalSize.height > clip.asset.naturalSize.width
        if firstIsPortrait != newIsPortrait {
            // Log warning — UI nên hiển thị cho user
            print("[Warning] Orientation mismatch: adding \(newIsPortrait ? "portrait" : "landscape") to \(firstIsPortrait ? "portrait" : "landscape") timeline")
        }
    }
    timeline.clips.append(clip)
}
```

### 7.4 Clip với không có Video Track

```swift
// Load từ Photos — thường không xảy ra nhưng cần guard
guard let _ = try? await asset.loadTracks(withMediaType: .video).first else {
    throw MediaError.noVideoTrack
}
```

### 7.5 Export File đã tồn tại

```swift
// AVAssetWriter fail nếu outputURL đã tồn tại
func prepareOutputURL() -> URL {
    let url = FileManager.default.temporaryDirectory
        .appendingPathComponent("\(UUID().uuidString).mp4")
    // UUID guarantee unique — không cần check tồn tại
    return url
}

// Cleanup temp files sau khi save to Photos
func cleanup(url: URL) {
    try? FileManager.default.removeItem(at: url)
}
```

### 7.6 Background Export Bị Interrupt

```swift
final class BackgroundExportManager {
    private var taskID: UIBackgroundTaskIdentifier = .invalid

    func beginBackgroundTask() {
        taskID = UIApplication.shared.beginBackgroundTask { [weak self] in
            // System sắp kill — cancel export gracefully
            self?.cancelExport()
            UIApplication.shared.endBackgroundTask(self?.taskID ?? .invalid)
        }
    }

    func endBackgroundTask() {
        UIApplication.shared.endBackgroundTask(taskID)
        taskID = .invalid
    }

    // Cancel: AVAssetReader.cancelReading() + AVAssetWriter.cancelWriting()
    func cancelExport() {
        // Signal export pipeline để dừng
    }
}
```

---

## 8. File & Module Structure

```
VideoEditor/
├── App/
│   ├── AppDelegate.swift
│   └── SceneDelegate.swift
│
├── Models/                          ← Pure data, no UIKit/AVFoundation side effects
│   ├── MediaAsset.swift             (1.1)
│   ├── Clip.swift                   (1.2)
│   ├── AdjustmentParams.swift       (1.3)
│   ├── FilterType.swift             (1.3)
│   ├── Timeline.swift               (1.4)
│   ├── RenderParams.swift           (1.6)
│   ├── ExportConfig.swift           (1.7)
│   └── EditHistory.swift            (1.5)
│
├── Algorithms/                      ← Pure functions, testable without device
│   ├── CropAlgorithm.swift          (3.2)
│   ├── TransformAlgorithm.swift     (3.3)
│   ├── TrimSeekController.swift     (3.4)
│   └── TimelineMapping.swift        (3.1)
│
├── Metal/                           ← GPU layer
│   ├── Shaders/
│   │   └── CombinedShader.metal     (3.5)
│   ├── MetalFrameRenderer.swift
│   ├── MetalTextureCache.swift      (6.2)
│   └── PixelBufferPool.swift        (6.1)
│
├── AVFoundation/                    ← AVFoundation layer
│   ├── CompositionBuilder.swift     (3.7)
│   ├── ExportPipeline.swift         (3.6)
│   ├── ThumbnailGenerator.swift     (6.3)
│   ├── ThumbnailCache.swift         (2.2)
│   └── PreviewController.swift      (5)
│
├── ViewModels/
│   └── EditorViewModel.swift        (4.3)
│
├── ViewControllers/                 ← UIKit
│   ├── EditorViewController.swift
│   ├── TrimViewController.swift
│   ├── CropViewController.swift
│   └── ExportViewController.swift
│
└── Views/                           ← SwiftUI panels (nhúng qua UIHostingController)
    ├── AdjustmentPanelView.swift
    ├── FilterPanelView.swift
    └── RatioPanelView.swift
```

### Dependency Rules (tránh circular imports)

```
Models          ←── không import gì từ app
Algorithms      ←── chỉ import Models + Foundation
Metal           ←── import Models + MetalKit
AVFoundation    ←── import Models + Algorithms + AVFoundation
ViewModels      ←── import Models + AVFoundation + Metal
ViewControllers ←── import ViewModels + UIKit
Views (SwiftUI) ←── import ViewModels + SwiftUI
```

---

## Tóm tắt Design Decisions

| Quyết định | Lý do |
|---|---|
| `Clip` là value type | Undo/redo bằng snapshot — không cần inverse command |
| `AVAsset` không copy trong snapshot | Lazy-loading object — reference share là đúng |
| Single Metal pass (filter + adjustment) | Giảm bandwidth, không ảnh hưởng chất lượng |
| `CVMetalTextureCache` | Zero-copy GPU ↔ IOSurface — critical cho 30fps preview |
| `TrimSeekController` debounce | Tránh seek queue overflow khi drag |
| Anchor-point crop scaling | Gesture feel tự nhiên — vùng dưới ngón tay không dịch |
| `clipOffset` tại boundary thuộc clip tiếp theo | Đảm bảo render params nhất quán tại transition points |
| Audio passthrough (không re-encode) | Nhanh hơn, không mất chất lượng |
| FitMode thay vì FillMode mặc định | Không crop nội dung — user quyết định qua Crop feature |
| UUID cho temp export file | Tránh conflict, không cần check tồn tại |

---

_Data Structures & Algorithms Design — Video Editor iOS_
