# Metal Render Approach for Realtime Video Editing on iOS

## 1. Overview

Nếu đã có một realtime camera filter pipeline sử dụng Metal, thì phần khó nhất của một video editor realtime thực tế đã được giải quyết.

Bởi vì:

- Camera realtime processing
- Video frame realtime processing
- Video export rendering

đều có cùng bản chất:

```text
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

Điểm khác biệt chủ yếu nằm ở:
- frame source
- timing
- synchronization
- composition orchestration

KHÔNG nằm ở:
- rendering engine
- shader processing
- GPU filter architecture

---

# 2. Core Insight

## Camera Pipeline

```text
Camera
↓
CMSampleBuffer
↓
CVPixelBuffer
↓
Metal Texture
↓
Shader
↓
MTKView
```

## Video Editing Pipeline

```text
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

Sau bước convert sang `CVPixelBuffer`:

```text
CVPixelBuffer
↓
Metal Texture
↓
Shader
```

pipeline gần như identical.

Điều này có nghĩa:
- Metal shader
- filter logic
- GPU processing
- render pipeline

có thể reuse gần như nguyên vẹn.

---

# 3. Reuse Percentage

| Layer | Reuse |
|---|---|
| Metal shader | 90–100% |
| Filter pipeline | 80–90% |
| Texture processing | 90% |
| CVPixelBuffer handling | 80% |
| GPU render flow | 70–90% |
| Playback synchronization | cần xây mới |
| Export pipeline | cần xây mới |

---

# 4. Camera vs Video Editing

## Camera App

Camera app thực chất là:

```text
Live Frame Producer
```

## Video Editor

Video editor thực chất là:

```text
Deterministic Frame Producer
```

Frame được:
- seek
- pause
- replay
- render theo timestamp cụ thể

---

# 5. AVVideoCompositing

`AVVideoCompositing` là custom frame renderer của AVFoundation.

Apple gọi renderer theo từng frame video:

```swift
startRequest(_:)
```

và cung cấp source frame để xử lý.

---

# 6. AVAsynchronousVideoCompositionRequest

API:

```swift
request.sourceFrame(byTrackID:)
```

trả về:

```swift
CVPixelBuffer
```

Điều này cực kỳ quan trọng.

## Camera Pipeline

```swift
CMSampleBuffer
→ CVPixelBuffer
```

## Video Composition Pipeline

```swift
AVAsynchronousVideoCompositionRequest
→ CVPixelBuffer
```

Sau bước này:
pipeline xử lý gần như identical.

---

# 7. Realtime Video Filtering

## Có thể apply filter realtime lên video đang play không?

Hoàn toàn có thể.

Video playback realtime có thể xử lý:
- filter
- adjustment
- shader effects
- distortion
- LUT
- blur
- bloom
- overlay

giống camera realtime pipeline.

---

# 8. Three Main Realtime Rendering Approaches

## Approach 1 — CoreImage Handler

### API

```swift
AVVideoComposition(asset:) { request in
}
```

### Pros

- đơn giản
- Apple-managed
- ít code
- phù hợp MVP

### Cons

- khó custom Metal sâu
- limited control
- khó tối ưu realtime phức tạp

---

## Approach 2 — AVPlayerItemVideoOutput (Recommended)

### Core Idea

```text
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

### Why This Approach is Powerful

Nếu đã có:
- Metal renderer
- shader pipeline
- realtime frame loop

thì:
có thể reuse gần như toàn bộ.

### AVPlayerItemVideoOutput

API:

```swift
copyPixelBuffer(
    forItemTime: itemTime,
    itemTimeForDisplay: nil
)
```

trả về:

```swift
CVPixelBuffer
```

### Điều này có nghĩa

Camera pipeline:

```text
Camera
↓
CVPixelBuffer
↓
Metal
```

Video playback pipeline:

```text
Video
↓
CVPixelBuffer
↓
Metal
```

=> renderer layer có thể shared.

---

## Approach 3 — AVVideoCompositing

### Use Cases

Phù hợp cho:
- export
- timeline rendering
- transition
- multi-track composition
- deterministic rendering

### Not Ideal For

Không tối ưu cho:
- interactive slider
- gesture editing
- low latency preview

---

# 9. Professional Architecture

## Interactive Preview

```text
AVPlayerItemVideoOutput
↓
Metal Renderer
↓
MTKView
```

## Final Export

```text
AVMutableComposition
↓
AVVideoCompositing
↓
Metal Renderer
↓
AVAssetExportSession
```

---

# 10. Best Architectural Direction

## Separate Frame Source from Renderer

### Frame Provider

```swift
protocol FrameProvider {
    func nextPixelBuffer() -> CVPixelBuffer?
}
```

### Frame Processor

```swift
protocol FrameProcessor {
    func process(
        _ pixelBuffer: CVPixelBuffer
    ) -> MTLTexture
}
```

---

# 11. Benefits

| Source | Reuse Renderer |
|---|---|
| Camera | ✅ |
| Video Playback | ✅ |
| Video Export | ✅ |

Renderer trở thành:
- reusable
- stateless
- independent

---

# 12. Stateless Renderer

## Recommended

```swift
render(
    texture,
    parameters,
    time
)
```

## Avoid

```swift
cameraRenderer.render()
```

Renderer nên:
- independent
- pure rendering
- không phụ thuộc source

---

# 13. Export Pipeline

## Realtime Preview != Export

### Realtime Preview

Tập trung:
- low latency
- smooth FPS
- responsiveness

### Export

Tập trung:
- deterministic rendering
- exact output
- frame accuracy

---

# 14. Major Challenges When Moving to Video Editing

## A. Seeking

```text
Seek
↓
Jump Frame
↓
Resync Renderer
↓
Invalidate Cache
```

## B. Variable FPS

Video có thể:
- 24 FPS
- 29.97 FPS
- 60 FPS
- Variable FPS

## C. Orientation

Imported video:
- transform matrix phức tạp
- preferredTransform
- aspect ratio

## D. Decode Bottleneck

Video:
- cần decode H264/H265 trước

Điều này ảnh hưởng:
- frame timing
- buffering
- memory
- latency

---

# 15. Recommended Production Architecture

```text
                 ┌─────────────────┐
                 │ Frame Provider  │
                 └─────────────────┘
                           ↓
        ┌─────────────────────────────────┐
        │                                 │
        │  Camera / Video / Export Source │
        │                                 │
        └─────────────────────────────────┘
                           ↓
                   CVPixelBuffer
                           ↓
                 ┌─────────────────┐
                 │ Metal Renderer  │
                 └─────────────────┘
                           ↓
                    MTLTexture
                           ↓
                 ┌─────────────────┐
                 │ Output Surface  │
                 └─────────────────┘
                           ↓
        ┌─────────────────────────────────┐
        │                                 │
        │ MTKView / Export / Composition  │
        │                                 │
        └─────────────────────────────────┘
```

---

# 16. Recommended Tech Stack

| Layer | Technology |
|---|---|
| Playback | AVPlayer |
| Frame Extraction | AVPlayerItemVideoOutput |
| Export | AVAssetExportSession |
| Composition | AVMutableComposition |
| Custom Render | AVVideoCompositing |
| GPU Processing | Metal |
| Frame Type | CVPixelBuffer |
| Display | MTKView |

---

# 17. Strategic Insight

Nếu đã có:
- Metal shader pipeline
- realtime camera rendering
- CVPixelBuffer processing
- GPU effects

thì:
core technology của realtime video editor đã tồn tại.

Thứ còn thiếu chủ yếu là:
- timeline architecture
- playback synchronization
- export orchestration
- AVFoundation composition
- seeking management

KHÔNG phải:
- rendering engine
- shader system
- GPU processing layer

---

# 18. Final Conclusion

Realtime camera filtering và realtime video editing:
- không phải hai domain khác nhau
- chúng là hai frame source khác nhau

Cùng chia sẻ:
- GPU pipeline
- shader architecture
- Metal renderer
- texture processing
- CVPixelBuffer flow

Điều này cho phép:
- reuse render engine mạnh
- xây dựng unified media processing architecture
- scale từ camera app sang video editor tương đối tự nhiên
