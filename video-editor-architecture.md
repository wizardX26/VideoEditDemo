# Video Editor iOS — Tài Liệu Kiến Trúc Hoàn Chỉnh

> Phiên bản hoàn thiện, bổ sung đầy đủ các điểm còn thiếu so với tài liệu gốc `metal-render-approach.md`.

---

## Mục lục

1. [Overview & Core Insight](#1-overview--core-insight)
2. [Pipeline Reuse — Camera vs Video](#2-pipeline-reuse--camera-vs-video)
3. [Reuse Percentage](#3-reuse-percentage)
4. [Ba Rendering Approach Chính](#4-ba-rendering-approach-chính)
5. [Timeline & Clip Data Model](#5-timeline--clip-data-model) ⭐ _Mới_
6. [Trim Video](#6-trim-video) ⭐ _Mới_
7. [Crop Video](#7-crop-video) ⭐ _Mới_
8. [Concatenate Videos](#8-concatenate-videos) ⭐ _Mới_
9. [Adjustments — Brightness / Contrast / Saturation](#9-adjustments--brightness--contrast--saturation) ⭐ _Mới_
10. [Filter — B&W / Sepia / Noir / Fade](#10-filter--bw--sepia--noir--fade) ⭐ _Mới_
11. [Ratio Feature](#11-ratio-feature) ⭐ _Mới_
12. [Export Pipeline Đầy Đủ](#12-export-pipeline-đầy-đủ) ⭐ _Mới_
13. [Preview / Export Consistency (WYSIWYG)](#13-preview--export-consistency-wysiwyg) ⭐ _Mới_
14. [Orientation & Transform Normalization](#14-orientation--transform-normalization) ⭐ _Mới_
15. [Kiến Trúc Renderer Stateless](#15-kiến-trúc-renderer-stateless)
16. [Production Architecture Tổng Thể](#16-production-architecture-tổng-thể)
17. [Tech Stack](#17-tech-stack)
18. [Major Challenges](#18-major-challenges)
19. [Tổng Kết](#19-tổng-kết)

---

## 1. Overview & Core Insight

Nếu đã có một realtime camera filter pipeline sử dụng Metal, thì phần khó nhất của một video editor realtime thực tế đã được giải quyết.

Camera realtime processing, video frame realtime processing, và video export rendering đều có cùng bản chất:

```
Frame Source
↓
CVPixelBuffer
↓
GPU Texture
↓
Metal Shader
↓
Render Target
```

Điểm khác biệt chủ yếu nằm ở **frame source**, **timing**, **synchronization**, và **composition orchestration** — KHÔNG nằm ở rendering engine, shader processing, hay GPU filter architecture.

---

## 2. Pipeline Reuse — Camera vs Video

### Camera Pipeline

```
Camera
↓
CMSampleBuffer → CVPixelBuffer
↓
Metal Texture
↓
Shader
↓
MTKView
```

### Video Editing Pipeline

```
AVAsset Frame
↓
CVPixelBuffer
↓
Metal Texture
↓
Shader
↓
Display / Export
```

Sau bước convert sang `CVPixelBuffer`, pipeline gần như **identical**. Metal shader, filter logic, GPU processing, và render pipeline có thể reuse gần như nguyên vẹn.

---

## 3. Reuse Percentage

| Layer | Reuse |
|---|---|
| Metal shader | 90–100% |
| Filter pipeline | 80–90% |
| Texture processing | 90% |
| CVPixelBuffer handling | 80% |
| GPU render flow | 70–90% |
| Playback synchronization | Cần xây mới |
| Export pipeline | Cần xây mới |
| Timeline / Clip model | Cần xây mới |
| Seek & frame accuracy | Cần xây mới |

---

## 4. Ba Rendering Approach Chính

### Approach 1 — CoreImage Handler

```swift
AVVideoComposition(asset:) { request in
    // apply CIFilter
}
```

**Pros:** Đơn giản, Apple-managed, ít code, phù hợp MVP.

**Cons:** Khó custom Metal sâu, limited control, khó tối ưu realtime phức tạp.

---

### Approach 2 — AVPlayerItemVideoOutput ✅ Recommended cho Interactive Preview

```
AVPlayer
↓
AVPlayerItemVideoOutput
↓
CVPixelBuffer
↓
Metal Renderer
↓
MTKView
```

API lấy frame:

```swift
let pixelBuffer = videoOutput.copyPixelBuffer(
    forItemTime: itemTime,
    itemTimeForDisplay: nil
)
```

Renderer layer hoàn toàn có thể shared với camera pipeline.

---

### Approach 3 — AVVideoCompositing

Phù hợp cho export, timeline rendering, transition, multi-track composition, deterministic rendering.

Không tối ưu cho interactive slider, gesture editing, hay low-latency preview.

Apple gọi renderer theo từng frame qua:

```swift
func startRequest(_ request: AVAsynchronousVideoCompositionRequest) {
    let pixelBuffer = request.sourceFrame(byTrackID: trackID)
    // → xử lý Metal → finishRequest
}
```

---

## 5. Timeline & Clip Data Model

> ⭐ **Đây là phần hoàn toàn thiếu trong tài liệu gốc và là nền tảng của toàn bộ editor.**

Không có data model này, không thể đảm bảo preview và export nhất quán với nhau.

### Clip Model

```swift
struct AdjustmentParams: Equatable {
    var brightness: Float = 0.0   // -1.0 → 1.0
    var contrast: Float   = 1.0   // 0.0 → 2.0
    var saturation: Float = 1.0   // 0.0 → 2.0
}

enum FilterType: String, CaseIterable {
    case none
    case blackAndWhite
    case sepia
    case noir
    case fade
}

struct Clip: Identifiable {
    let id: UUID
    let asset: AVAsset
    var trimRange: CMTimeRange          // start/end trong source asset
    var cropRect: CGRect                // normalized 0–1 (x, y, width, height)
    var adjustments: AdjustmentParams
    var filter: FilterType
    var preferredTransform: CGAffineTransform  // orientation từ asset
}
```

### Timeline Model

```swift
struct Timeline {
    var clips: [Clip]
    var outputResolution: CGSize        // ví dụ: 1920×1080, 1080×1080
    var outputFPS: Float                // 30, 60...
    var outputAspectRatio: AspectRatio  // 16:9, 1:1, 9:16, 4:3
}

enum AspectRatio: String, CaseIterable {
    case widescreen  = "16:9"
    case square      = "1:1"
    case portrait    = "9:16"
    case standard    = "4:3"

    var size: CGSize {
        switch self {
        case .widescreen: return CGSize(width: 1920, height: 1080)
        case .square:     return CGSize(width: 1080, height: 1080)
        case .portrait:   return CGSize(width: 1080, height: 1920)
        case .standard:   return CGSize(width: 1440, height: 1080)
        }
    }
}
```

### Tại sao quan trọng

- Mỗi `Clip` lưu đầy đủ trạng thái: trim, crop, adjustment, filter, orientation.
- `Timeline` định nghĩa output cuối cùng.
- Preview và Export cùng đọc từ model này → đảm bảo WYSIWYG.
- Apply per-video (như Notes khuyến khích) trở nên tự nhiên: mỗi Clip có params riêng.

---

## 6. Trim Video

> ⭐ **Tài liệu gốc đề cập seeking như challenge nhưng chưa nói implementation.**

### Core API

Trim dùng `CMTimeRange` để xác định đoạn cần giữ lại:

```swift
let startTime = CMTime(seconds: 2.5, preferredTimescale: 600)
let endTime   = CMTime(seconds: 8.0, preferredTimescale: 600)
let trimRange = CMTimeRange(start: startTime, end: endTime)

// Lưu vào Clip model
clip.trimRange = trimRange
```

### Preview Trim

Khi scrubbing thanh trim, dùng `AVPlayer.seek` với tolerance thấp:

```swift
player.seek(
    to: targetTime,
    toleranceBefore: .zero,
    toleranceAfter: CMTime(value: 1, timescale: 600)
)
```

Tolerance `.zero` đảm bảo frame-accurate preview nhưng chậm hơn. Tolerance nhỏ là compromise tốt.

### Export Trim

Khi export, dùng `AVAssetExportSession` với `timeRange`:

```swift
let exporter = AVAssetExportSession(asset: asset, presetName: AVAssetExportPresetHighestQuality)!
exporter.timeRange = clip.trimRange
exporter.outputURL = outputURL
exporter.exportAsynchronously { ... }
```

Hoặc khi dùng `AVMutableComposition` (multi-clip):

```swift
let compositionTrack = composition.addMutableTrack(
    withMediaType: .video,
    preferredTrackID: kCMPersistentTrackID_Invalid
)
try compositionTrack?.insertTimeRange(
    clip.trimRange,
    of: videoTrack,
    at: insertTime
)
```

### Frame Accuracy & Keyframe Issue

H.264/H.265 encode theo keyframe interval (thường mỗi 2–3 giây). Seek đến non-keyframe có thể snap sang keyframe gần nhất nếu không xử lý đúng.

**Giải pháp:**
- Dùng `preferredTimescale: 600` (hoặc 90000 cho video FPS cao) thay vì 1.
- Khi export cần frame-accurate trim với `AVAssetWriter` (xem phần Export), decode frame-by-frame thay vì dùng seek.

---

## 7. Crop Video

> ⭐ **Hoàn toàn thiếu trong tài liệu gốc.**

### Core Concept

Crop không resize frame thật trong preview — thay vào đó điều chỉnh **UV coordinate** trong Metal shader để chỉ sample vùng crop.

### Metal Shader — UV Crop

```metal
fragment float4 cropFragment(
    VertexOut in [[stage_in]],
    texture2d<float> inputTexture [[texture(0)]],
    constant CropParams& params [[buffer(0)]]
) {
    constexpr sampler s(address::clamp_to_edge, filter::linear);

    // Remap UV vào vùng crop
    float2 croppedUV = params.cropOrigin + in.texCoord * params.cropSize;
    return inputTexture.sample(s, croppedUV);
}

struct CropParams {
    float2 cropOrigin;  // normalized (x, y) của top-left crop rect
    float2 cropSize;    // normalized (width, height) của crop rect
};
```

### Crop Params từ CGRect

```swift
struct CropParams {
    var cropOrigin: SIMD2<Float>
    var cropSize: SIMD2<Float>

    init(from rect: CGRect) {
        cropOrigin = SIMD2<Float>(Float(rect.origin.x), Float(rect.origin.y))
        cropSize   = SIMD2<Float>(Float(rect.width), Float(rect.height))
    }
}
```

### UI — Pinch & Pan Gesture

```swift
class CropViewController: UIViewController {
    var cropRect: CGRect = CGRect(x: 0, y: 0, width: 1, height: 1) // normalized

    @objc func handlePinch(_ gesture: UIPinchGestureRecognizer) {
        let scale = gesture.scale
        gesture.scale = 1.0
        // Thu nhỏ crop rect (zoom in = crop rect nhỏ hơn)
        let newWidth  = max(0.1, cropRect.width / scale)
        let newHeight = max(0.1, cropRect.height / scale)
        cropRect = centeredCropRect(width: newWidth, height: newHeight)
        updatePreview()
    }

    @objc func handlePan(_ gesture: UIPanGestureRecognizer) {
        let translation = gesture.translation(in: view)
        gesture.setTranslation(.zero, in: view)
        let dx = Float(translation.x) / Float(view.bounds.width)
        let dy = Float(translation.y) / Float(view.bounds.height)
        cropRect.origin.x = clamp(cropRect.origin.x + CGFloat(dx), 0, 1 - cropRect.width)
        cropRect.origin.y = clamp(cropRect.origin.y + CGFloat(dy), 0, 1 - cropRect.height)
        updatePreview()
    }
}
```

### Crop với Aspect Ratio Constraint

Khi crop cần giữ ratio (ví dụ 16:9), clamp crop rect:

```swift
func constrainedCropRect(
    current: CGRect,
    ratio: CGSize,
    anchorPoint: CGPoint
) -> CGRect {
    let targetRatio = ratio.width / ratio.height
    var rect = current

    // Giữ width, điều chỉnh height theo ratio
    rect.size.height = rect.size.width / targetRatio

    // Clamp trong bounds [0, 1]
    rect.origin.x = clamp(rect.origin.x, 0, 1 - rect.width)
    rect.origin.y = clamp(rect.origin.y, 0, 1 - rect.height)

    return rect
}
```

### Export Crop

Khi export, crop được apply trong cùng Metal render pass — không cần xử lý riêng. `FrameProcessor` nhận `CropParams` từ `Clip.cropRect` và apply trong shader.

---

## 8. Concatenate Videos

> ⭐ **Tài liệu gốc chỉ đề cập AVMutableComposition trong tech stack nhưng không giải thích.**

### Core — AVMutableComposition

```swift
func buildComposition(from clips: [Clip]) throws -> AVMutableComposition {
    let composition = AVMutableComposition()

    guard
        let videoTrack = composition.addMutableTrack(
            withMediaType: .video,
            preferredTrackID: kCMPersistentTrackID_Invalid
        ),
        let audioTrack = composition.addMutableTrack(
            withMediaType: .audio,
            preferredTrackID: kCMPersistentTrackID_Invalid
        )
    else { throw CompositionError.trackCreationFailed }

    var insertTime = CMTime.zero

    for clip in clips {
        let asset = clip.asset

        // Insert video track
        if let sourceVideoTrack = asset.tracks(withMediaType: .video).first {
            try videoTrack.insertTimeRange(
                clip.trimRange,
                of: sourceVideoTrack,
                at: insertTime
            )
        }

        // Insert audio track (optional — skip nếu clip không có audio)
        if let sourceAudioTrack = asset.tracks(withMediaType: .audio).first {
            try? audioTrack.insertTimeRange(
                clip.trimRange,
                of: sourceAudioTrack,
                at: insertTime
            )
        }

        insertTime = insertTime + clip.trimRange.duration
    }

    return composition
}
```

### Resolution Mismatch — Vấn Đề Thực Tế

Khi concatenate, các video có thể có resolution khác nhau (ví dụ: 1920×1080 và 1280×720). Cần normalize về `outputResolution` của Timeline.

```swift
func buildVideoComposition(
    from composition: AVMutableComposition,
    clips: [Clip],
    outputResolution: CGSize
) -> AVMutableVideoComposition {
    let videoComposition = AVMutableVideoComposition()
    videoComposition.renderSize = outputResolution
    videoComposition.frameDuration = CMTime(value: 1, timescale: 30)

    var instructions: [AVMutableVideoCompositionInstruction] = []
    var currentTime = CMTime.zero

    for clip in clips {
        let instruction = AVMutableVideoCompositionInstruction()
        instruction.timeRange = CMTimeRange(start: currentTime, duration: clip.trimRange.duration)

        let layerInstruction = AVMutableVideoCompositionLayerInstruction(
            assetTrack: composition.tracks(withMediaType: .video).first!
        )

        // Apply transform để fit video vào outputResolution
        let transform = fitTransform(
            from: clip.asset.tracks(withMediaType: .video).first!.naturalSize,
            to: outputResolution,
            preferredTransform: clip.preferredTransform
        )
        layerInstruction.setTransform(transform, at: currentTime)

        instruction.layerInstructions = [layerInstruction]
        instructions.append(instruction)

        currentTime = currentTime + clip.trimRange.duration
    }

    videoComposition.instructions = instructions
    return videoComposition
}
```

### Audio Handling

- Nếu một clip không có audio track, `AVMutableCompositionTrack` sẽ có khoảng trống (silence) tại đó — đây là behavior đúng.
- Sample rate khác nhau giữa các clips (44100 Hz vs 48000 Hz): `AVMutableComposition` tự handle khi export qua `AVAssetExportSession`.
- Nếu export qua `AVAssetWriter`, cần resample audio thủ công bằng `AVAudioConverter`.

---

## 9. Adjustments — Brightness / Contrast / Saturation

> ⭐ **Tài liệu gốc đề cập nhưng thiếu hoàn toàn implementation detail.**

### Metal Shader

```metal
struct AdjustmentParams {
    float brightness;  // -1.0 → 1.0, default 0.0
    float contrast;    // 0.0 → 2.0, default 1.0
    float saturation;  // 0.0 → 2.0, default 1.0
};

fragment float4 adjustmentFragment(
    VertexOut in [[stage_in]],
    texture2d<float> inputTexture [[texture(0)]],
    constant AdjustmentParams& params [[buffer(0)]]
) {
    constexpr sampler s(address::clamp_to_edge, filter::linear);
    float4 color = inputTexture.sample(s, in.texCoord);

    // Brightness: shift toàn bộ giá trị
    float3 rgb = color.rgb + float3(params.brightness);

    // Contrast: scale quanh midpoint 0.5
    rgb = (rgb - 0.5) * params.contrast + 0.5;

    // Saturation: mix với luminance
    float luminance = dot(rgb, float3(0.2126, 0.7152, 0.0722));
    rgb = mix(float3(luminance), rgb, params.saturation);

    return float4(clamp(rgb, 0.0, 1.0), color.a);
}
```

### Chaining Filter + Adjustment trong Một Pass

Để tránh redundant texture copy, chain filter và adjustment trong cùng một fragment shader:

```metal
fragment float4 combinedFragment(
    VertexOut in [[stage_in]],
    texture2d<float> inputTexture [[texture(0)]],
    texture3d<float> lutTexture   [[texture(1)]],  // cho filter
    constant CombinedParams& params [[buffer(0)]]
) {
    constexpr sampler s(address::clamp_to_edge, filter::linear);
    float4 color = inputTexture.sample(s, in.texCoord);

    // 1. Apply LUT filter
    if (params.filterEnabled) {
        color = applyLUT(color, lutTexture);
    }

    // 2. Apply adjustment
    color.rgb = applyAdjustment(color.rgb, params.adjustment);

    return color;
}
```

### Per-Video vs Global Apply

Như Notes khuyến khích, apply per-video: mỗi `Clip` trong Timeline model lưu `AdjustmentParams` riêng. Renderer nhận params tương ứng với clip đang render.

---

## 10. Filter — B&W / Sepia / Noir / Fade

> ⭐ **Tài liệu gốc đề cập filter như use case nhưng thiếu implementation.**

### Approach 1 — LUT Texture (Recommended)

LUT (Look-Up Table) là 3D texture 64×64×64 encode mapping màu. Đây là cách implement chuyên nghiệp nhất — reuse từ camera pipeline nếu đã có.

```metal
float4 applyLUT(
    float4 inputColor,
    texture3d<float> lutTexture
) {
    constexpr sampler lutSampler(
        address::clamp_to_edge,
        filter::linear
    );

    // Normalize về [0, 1] range của LUT
    float3 lutCoord = clamp(inputColor.rgb, 0.001, 0.999);
    return float4(lutTexture.sample(lutSampler, lutCoord).rgb, inputColor.a);
}
```

Mỗi filter là một file `.png` LUT (hoặc `.cube` convert sang texture). Không cần viết shader riêng cho từng filter.

### Approach 2 — CoreImage Filters (Đơn Giản Hơn)

Phù hợp cho MVP nếu chưa có LUT pipeline:

```swift
func applyFilter(_ type: FilterType, to image: CIImage) -> CIImage {
    switch type {
    case .none:
        return image
    case .blackAndWhite:
        return image.applyingFilter("CIPhotoEffectMono")
    case .sepia:
        return image.applyingFilter("CISepiaTone", parameters: ["inputIntensity": 0.8])
    case .noir:
        return image.applyingFilter("CIPhotoEffectNoir")
    case .fade:
        return image.applyingFilter("CIPhotoEffectFade")
    }
}
```

**Lưu ý:** CoreImage filter khó chain với Metal pipeline — cần convert `CIImage → CVPixelBuffer → MTLTexture` mỗi frame. Với LUT approach, toàn bộ xử lý ở trong Metal pass.

### Filter State

Filter được lưu per-clip trong `Clip.filter: FilterType`. Nếu chọn apply cho tất cả clips, set đồng loạt qua Timeline:

```swift
mutating func applyFilterToAll(_ filter: FilterType) {
    clips = clips.map { var c = $0; c.filter = filter; return c }
}
```

---

## 11. Ratio Feature

> ⭐ **Hoàn toàn thiếu trong tài liệu gốc.**

Ratio là preset crop kết hợp với `outputResolution` thay đổi.

### Ratio Selection Flow

```
User chọn ratio (16:9, 1:1, 9:16, 4:3)
↓
Timeline.outputAspectRatio = ratio
↓
Timeline.outputResolution = ratio.size
↓
Crop rect của mỗi Clip được constrain theo ratio mới
↓
Preview MTKView re-render với aspect ratio mới
```

### Crop Rect Constrain khi Đổi Ratio

```swift
mutating func applyAspectRatio(_ ratio: AspectRatio) {
    outputAspectRatio = ratio
    outputResolution = ratio.size

    // Constrain crop rect của từng clip theo ratio mới
    clips = clips.map { clip in
        var updated = clip
        updated.cropRect = constrainedCropRect(
            current: clip.cropRect,
            ratio: ratio.size
        )
        return updated
    }
}

func constrainedCropRect(current: CGRect, ratio: CGSize) -> CGRect {
    let targetRatio = ratio.width / ratio.height
    let currentRatio = current.width / current.height

    var rect = current

    if currentRatio > targetRatio {
        // Quá rộng → giảm width
        rect.size.width = rect.size.height * targetRatio
    } else {
        // Quá cao → giảm height
        rect.size.height = rect.size.width / targetRatio
    }

    // Center lại và clamp
    rect.origin.x = clamp(
        current.midX - rect.width / 2,
        0, 1 - rect.width
    )
    rect.origin.y = clamp(
        current.midY - rect.height / 2,
        0, 1 - rect.height
    )

    return rect
}
```

---

## 12. Export Pipeline Đầy Đủ

> ⭐ **Tài liệu gốc đề cập AVAssetExportSession và AVVideoCompositing nhưng thiếu path quan trọng nhất: AVAssetWriter.**

### Ba Export Path

| Scenario | Path |
|---|---|
| Trim only, no custom shader | `AVAssetExportSession` + `timeRange` |
| Filter / Adjustment / Crop với single clip | `AVAssetExportSession` + `AVVideoCompositing` |
| Multi-clip (concatenate) + custom Metal render | `AVAssetWriter` + `AVAssetReader` ✅ |

### AVAssetWriter Export — Full Implementation

Đây là path dùng khi cần custom Metal render mỗi frame khi export.

```swift
class VideoExporter {

    func export(
        timeline: Timeline,
        outputURL: URL,
        progress: @escaping (Float) -> Void,
        completion: @escaping (Result<Void, Error>) -> Void
    ) {
        Task.detached {
            do {
                try await self.exportInternal(
                    timeline: timeline,
                    outputURL: outputURL,
                    progress: progress
                )
                completion(.success(()))
            } catch {
                completion(.failure(error))
            }
        }
    }

    private func exportInternal(
        timeline: Timeline,
        outputURL: URL,
        progress: @escaping (Float) -> Void
    ) async throws {

        // 1. Build composition từ clips
        let composition = try buildComposition(from: timeline.clips)
        let totalDuration = composition.duration
        let outputSize = timeline.outputResolution

        // 2. Setup AVAssetWriter
        let writer = try AVAssetWriter(outputURL: outputURL, fileType: .mp4)

        let videoSettings: [String: Any] = [
            AVVideoCodecKey: AVVideoCodecType.h264,
            AVVideoWidthKey: outputSize.width,
            AVVideoHeightKey: outputSize.height,
        ]
        let videoInput = AVAssetWriterInput(
            mediaType: .video,
            outputSettings: videoSettings
        )
        videoInput.expectsMediaDataInRealTime = false

        let pixelBufferAdaptor = AVAssetWriterInputPixelBufferAdaptor(
            assetWriterInput: videoInput,
            sourcePixelBufferAttributes: [
                kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA,
                kCVPixelBufferWidthKey as String: outputSize.width,
                kCVPixelBufferHeightKey as String: outputSize.height,
            ]
        )
        writer.add(videoInput)

        // 3. Setup AVAssetReader
        let reader = try AVAssetReader(asset: composition)
        let readerOutput = AVAssetReaderVideoCompositionOutput(
            videoTracks: composition.tracks(withMediaType: .video),
            videoSettings: [
                kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA
            ]
        )

        // Apply video composition (resolution normalization, orientation)
        let videoComposition = buildVideoComposition(
            from: composition,
            clips: timeline.clips,
            outputResolution: outputSize
        )
        readerOutput.videoComposition = videoComposition
        reader.add(readerOutput)

        // 4. Setup Audio
        let audioInput = AVAssetWriterInput(
            mediaType: .audio,
            outputSettings: nil  // passthrough
        )
        writer.add(audioInput)

        if let audioTrack = composition.tracks(withMediaType: .audio).first {
            let audioOutput = AVAssetReaderTrackOutput(
                track: audioTrack,
                outputSettings: nil  // passthrough
            )
            reader.add(audioOutput)
        }

        // 5. Start
        reader.startReading()
        writer.startWriting()
        writer.startSession(atSourceTime: .zero)

        // 6. Metal renderer (shared với preview pipeline)
        let renderer = MetalFrameRenderer()

        // 7. Write video frames
        let videoQueue = DispatchQueue(label: "video.export.queue")
        await withCheckedContinuation { continuation in
            videoInput.requestMediaDataWhenReady(on: videoQueue) {
                while videoInput.isReadyForMoreMediaData {
                    guard
                        let sampleBuffer = readerOutput.copyNextSampleBuffer(),
                        let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer)
                    else {
                        videoInput.markAsFinished()
                        continuation.resume()
                        return
                    }

                    let presentationTime = CMSampleBufferGetPresentationTimeStamp(sampleBuffer)

                    // Tìm clip tương ứng với thời điểm này
                    let clipParams = self.renderParams(
                        for: presentationTime,
                        in: timeline
                    )

                    // Apply Metal render
                    if let outputBuffer = renderer.render(
                        pixelBuffer,
                        params: clipParams
                    ) {
                        pixelBufferAdaptor.append(outputBuffer, withPresentationTime: presentationTime)
                    }

                    // Progress
                    let elapsed = CMTimeGetSeconds(presentationTime)
                    let total = CMTimeGetSeconds(totalDuration)
                    progress(Float(elapsed / total))
                }
            }
        }

        // 8. Finish
        audioInput.markAsFinished()
        await writer.finishWriting()

        if writer.status == .failed {
            throw writer.error ?? ExportError.unknown
        }
    }
}
```

### Progress Tracking

`AVAssetWriter` không có built-in progress. Tính thủ công từ `presentationTime / totalDuration` như trên.

### Background Export

```swift
var backgroundTaskID: UIBackgroundTaskIdentifier = .invalid

func startExport() {
    backgroundTaskID = UIApplication.shared.beginBackgroundTask {
        // Timeout handler — cleanup nếu cần
        UIApplication.shared.endBackgroundTask(self.backgroundTaskID)
    }

    exporter.export(timeline: timeline, outputURL: outputURL) { result in
        UIApplication.shared.endBackgroundTask(self.backgroundTaskID)
        // Handle result
    }
}
```

### Lưu vào Photos

```swift
PHPhotoLibrary.shared().performChanges({
    PHAssetChangeRequest.creationRequestForAssetFromVideo(atFileURL: outputURL)
}) { success, error in
    // Handle result
}
```

---

## 13. Preview / Export Consistency (WYSIWYG)

> ⭐ **Thiếu trong tài liệu gốc — đây là vấn đề thực tế quan trọng.**

### Vấn Đề

Preview dùng `AVPlayerItemVideoOutput` (realtime), Export dùng `AVAssetWriter` (offline). Nếu hai path dùng shader/params khác nhau, output export sẽ trông khác preview.

### Giải Pháp — Shared FrameProcessor

```swift
// Protocol dùng chung cho cả Preview và Export
protocol FrameProcessor {
    func process(
        _ pixelBuffer: CVPixelBuffer,
        params: RenderParams
    ) -> MTLTexture?
}

struct RenderParams {
    var cropRect: CGRect
    var adjustments: AdjustmentParams
    var filter: FilterType
    var outputSize: CGSize
}

// Cùng một MetalFrameRenderer được dùng cho cả hai path
class MetalFrameRenderer: FrameProcessor {
    func process(
        _ pixelBuffer: CVPixelBuffer,
        params: RenderParams
    ) -> MTLTexture? {
        // Metal shader với cùng logic crop + filter + adjustment
        // ...
    }
}
```

### Preview Path

```swift
// CADisplayLink → lấy frame từ AVPlayerItemVideoOutput → MetalFrameRenderer → MTKView
displayLink.add(to: .main, forMode: .common)

@objc func displayLinkFired(_ link: CADisplayLink) {
    let itemTime = videoOutput.itemTime(forHostTime: link.timestamp)
    guard
        videoOutput.hasNewPixelBuffer(forItemTime: itemTime),
        let pixelBuffer = videoOutput.copyPixelBuffer(forItemTime: itemTime, itemTimeForDisplay: nil)
    else { return }

    let params = currentClipRenderParams()  // từ Timeline model
    if let texture = renderer.process(pixelBuffer, params: params) {
        mtkView.draw(texture: texture)
    }
}
```

### Export Path

```swift
// AVAssetReader → MetalFrameRenderer → AVAssetWriter
// Cùng renderer, cùng params từ Timeline model
if let outputBuffer = renderer.render(pixelBuffer, params: clipParams) {
    pixelBufferAdaptor.append(outputBuffer, withPresentationTime: presentationTime)
}
```

---

## 14. Orientation & Transform Normalization

> ⭐ **Tài liệu gốc liệt kê như challenge nhưng không giải quyết.**

### Vấn Đề

Video quay bằng iPhone thường có `preferredTransform` là rotation 90° hoặc 270°. Nếu không xử lý, video sẽ hiển thị ngang/lộn ngược.

### Đọc Transform khi Load Video

```swift
func loadClip(from url: URL) async -> Clip {
    let asset = AVURLAsset(url: url)
    let videoTrack = asset.tracks(withMediaType: .video).first!
    let preferredTransform = videoTrack.preferredTransform

    // Natural size sau khi apply transform
    let naturalSize = videoTrack.naturalSize.applying(preferredTransform)
    let correctedSize = CGSize(
        width: abs(naturalSize.width),
        height: abs(naturalSize.height)
    )

    return Clip(
        id: UUID(),
        asset: asset,
        trimRange: CMTimeRange(start: .zero, duration: asset.duration),
        cropRect: CGRect(x: 0, y: 0, width: 1, height: 1),
        adjustments: AdjustmentParams(),
        filter: .none,
        preferredTransform: preferredTransform
    )
}
```

### Apply Transform khi Export

```swift
func fitTransform(
    from sourceSize: CGSize,
    to outputSize: CGSize,
    preferredTransform: CGAffineTransform
) -> CGAffineTransform {
    // 1. Correct orientation
    var transform = preferredTransform

    // 2. Natural size sau orientation
    let transformedSize = sourceSize.applying(preferredTransform)
    let correctedWidth  = abs(transformedSize.width)
    let correctedHeight = abs(transformedSize.height)

    // 3. Scale để fit outputSize (aspect fit)
    let scaleX = outputSize.width  / correctedWidth
    let scaleY = outputSize.height / correctedHeight
    let scale  = min(scaleX, scaleY)

    transform = transform.scaledBy(x: scale, y: scale)

    // 4. Center
    let scaledWidth  = correctedWidth  * scale
    let scaledHeight = correctedHeight * scale
    let offsetX = (outputSize.width  - scaledWidth)  / 2
    let offsetY = (outputSize.height - scaledHeight) / 2
    transform = transform.translatedBy(x: offsetX / scale, y: offsetY / scale)

    return transform
}
```

### Apply Transform trong AVMutableVideoCompositionLayerInstruction

```swift
let layerInstruction = AVMutableVideoCompositionLayerInstruction(
    assetTrack: videoTrack
)
let transform = fitTransform(
    from: videoTrack.naturalSize,
    to: outputResolution,
    preferredTransform: clip.preferredTransform
)
layerInstruction.setTransform(transform, at: insertTime)
```

---

## 15. Kiến Trúc Renderer Stateless

### Frame Provider Protocol

```swift
protocol FrameProvider {
    func nextPixelBuffer() -> CVPixelBuffer?
}

// Camera source
class CameraFrameProvider: FrameProvider { ... }

// Video playback source
class VideoPlaybackFrameProvider: FrameProvider { ... }

// Video export source (AVAssetReader)
class VideoExportFrameProvider: FrameProvider { ... }
```

### Frame Processor Protocol

```swift
protocol FrameProcessor {
    func process(
        _ pixelBuffer: CVPixelBuffer,
        params: RenderParams
    ) -> MTLTexture?
}
```

### Renderer nên Stateless

```swift
// ✅ Recommended — stateless, pure rendering
renderer.render(texture, parameters: params, time: time)

// ❌ Avoid — coupled với source cụ thể
cameraRenderer.render()
```

### Benefits

| Source | Reuse Renderer |
|---|---|
| Camera | ✅ |
| Video Playback | ✅ |
| Video Export | ✅ |

---

## 16. Production Architecture Tổng Thể

```
                    ┌──────────────────────┐
                    │   Timeline Model     │
                    │  [Clip, Clip, Clip]  │
                    └──────────────────────┘
                               ↓
          ┌────────────────────┴────────────────────┐
          │                                         │
          ▼                                         ▼
   ┌─────────────┐                        ┌──────────────────┐
   │   PREVIEW   │                        │     EXPORT       │
   └─────────────┘                        └──────────────────┘
          │                                         │
          ▼                                         ▼
   AVPlayerItemVideoOutput              AVAssetReader
   + CADisplayLink                      + AVAssetReaderOutput
          │                                         │
          └─────────────┬───────────────────────────┘
                        ▼
                 CVPixelBuffer
                        ↓
              ┌─────────────────┐
              │ MetalFrameRenderer │  ← Shared, Stateless
              │  - Crop (UV)    │
              │  - Filter (LUT) │
              │  - Adjustment   │
              └─────────────────┘
                        ↓
                   MTLTexture
                        ↓
          ┌─────────────┴──────────────┐
          ▼                            ▼
       MTKView                 AVAssetWriterInput
   (Preview Display)         + PixelBufferAdaptor
                                       ↓
                               AVAssetWriter
                                       ↓
                              MP4 File → Photos
```

### Interactive Preview vs Final Export

| | Realtime Preview | Final Export |
|---|---|---|
| Frame Source | `AVPlayerItemVideoOutput` | `AVAssetReader` |
| Render Loop | `CADisplayLink` | Frame-by-frame loop |
| Priority | Low latency, smooth FPS | Deterministic, frame accuracy |
| Renderer | `MetalFrameRenderer` (shared) | `MetalFrameRenderer` (shared) |
| Output | `MTKView` | `AVAssetWriter` → `.mp4` |

---

## 17. Tech Stack

| Layer | Technology |
|---|---|
| Playback | `AVPlayer` |
| Frame Extraction | `AVPlayerItemVideoOutput` |
| Seek | `AVPlayer.seek(to:toleranceBefore:toleranceAfter:)` |
| Composition | `AVMutableComposition` |
| Orientation Fix | `AVMutableVideoCompositionLayerInstruction` |
| Custom Render (Export) | `AVVideoCompositing` hoặc `AVAssetWriter` |
| GPU Processing | Metal |
| Filter | Metal LUT Texture hoặc `CIFilter` |
| Frame Buffer | `CVPixelBuffer` |
| Display | `MTKView` |
| Export Encode | `AVAssetWriter` + `AVAssetWriterInput` |
| Export Decode | `AVAssetReader` + `AVAssetReaderOutput` |
| Audio | `AVAssetWriterInput` (audio passthrough) |
| Save to Photos | `PHPhotoLibrary` |
| Background Task | `UIBackgroundTaskIdentifier` |

---

## 18. Major Challenges

### A. Seeking & Frame Accuracy

```
Seek
↓
Jump Frame (có thể snap sang keyframe)
↓
Resync Renderer
↓
Invalidate Cache
```

Giải pháp: Dùng `toleranceBefore: .zero` cho frame-accurate seek khi cần, tolerance nhỏ cho scrubbing mượt.

### B. Variable FPS

Video có thể là 24, 29.97, 60, hoặc Variable FPS. `AVMutableVideoComposition.frameDuration` cần set phù hợp với FPS cao nhất trong timeline.

### C. Orientation

Xem Section 14. Luôn đọc `preferredTransform` khi load video và apply khi render.

### D. Decode Bottleneck

Video H.264/H.265 cần decode trước khi render. Ảnh hưởng frame timing, buffering, memory, latency. `AVPlayerItemVideoOutput` có internal buffer — không cần handle thủ công khi preview.

### E. Memory — Large Video Files

Không load toàn bộ video vào memory. `AVAssetReader` đọc frame-by-frame theo yêu cầu. Khi export nhiều clip, process tuần tự theo từng clip.

### F. Concatenate với Different Formats

Nếu các clips có codec khác nhau (H.264 vs H.265) hoặc color space khác nhau, `AVMutableComposition` vẫn hoạt động nhưng export qua `AVAssetWriter` cần normalize về cùng một output format.

---

## 19. Tổng Kết

### Tài liệu gốc — Đúng

| Nội dung | Trạng thái |
|---|---|
| Metal pipeline reuse insight | ✅ Đúng và quan trọng |
| AVPlayerItemVideoOutput cho preview | ✅ Đúng |
| AVVideoCompositing cho export | ✅ Đúng |
| FrameProvider abstraction | ✅ Tốt |
| Stateless renderer design | ✅ Tốt |

### Bổ Sung Trong Tài Liệu Này

| Nội dung | Trạng thái |
|---|---|
| Timeline & Clip Data Model | ⭐ Mới — nền tảng của editor |
| Trim (CMTimeRange, keyframe) | ⭐ Mới |
| Crop (UV shader, gesture UI) | ⭐ Mới |
| Concatenate (resolution normalize, audio) | ⭐ Mới |
| Adjustment Metal shader | ⭐ Mới |
| Filter LUT + CoreImage | ⭐ Mới |
| Ratio feature | ⭐ Mới |
| AVAssetWriter export path | ⭐ Mới — quan trọng nhất |
| Preview/Export consistency (WYSIWYG) | ⭐ Mới |
| Orientation normalization | ⭐ Mới |
| Background export | ⭐ Mới |
| Progress tracking | ⭐ Mới |

### Insight Chính

> Realtime camera filtering và realtime video editing không phải hai domain khác nhau — chúng là hai **frame source** khác nhau, cùng chia sẻ GPU pipeline, shader architecture, Metal renderer, và CVPixelBuffer flow.
>
> Thứ cần xây thêm cho video editor là: **Timeline/Clip data model**, **playback synchronization**, **export orchestration** (AVAssetWriter), và **AVFoundation composition** — KHÔNG phải rendering engine hay shader system.

---

_Tài liệu hoàn thiện — phiên bản bổ sung từ `metal-render-approach.md`_
