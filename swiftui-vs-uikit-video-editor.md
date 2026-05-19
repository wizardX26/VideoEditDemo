# SwiftUI vs UIKit — So Sánh Implementation Video Editor iOS

> Phân tích mức độ đáp ứng yêu cầu khi implement Video Editor Mini App với time bound 1 tuần.

---

## Tổng quan nhanh

| Chỉ số | Giá trị |
|---|---|
| Features feasible trong 1 tuần | 7/8 |
| High-risk items | 3 |
| Framework recommended | UIKit (core) + SwiftUI (UI panels) |
| Feature nên cắt nếu trễ | Crop |

---

## Mức độ đáp ứng từng feature — SwiftUI vs UIKit

| Feature | SwiftUI | UIKit | Effort |
|---|---|---|---|
| **Select Video** (PHPickerViewController) | ✅ Dễ | ✅ Dễ | 0.5 ngày |
| **Preview video** (AVPlayer + MTKView) | ⚠️ Cần `UIViewRepresentable` wrapper | ✅ Native | 0.5 ngày |
| **Trim** (CMTimeRange + scrubber UI) | ⚠️ Scrubber cần custom | ✅ Tốt hơn | 1 ngày |
| **Crop** 🔴 HIGH RISK (Pinch+pan gesture + UV shader) | ❌ Gesture conflict với SwiftUI system | ⚠️ Manageable | 1.5 ngày |
| **Adjustment** (Metal shader uniforms) | ⚠️ Slider OK, Metal wrap cần | ✅ Tốt | 0.5 ngày |
| **Filter** (CIFilter hoặc LUT) | ✅ CIFilter dễ | ✅ CIFilter dễ | 0.5 ngày |
| **Ratio** (Preset crop + output size) | ✅ Đơn giản | ✅ Đơn giản | 0.5 ngày |
| **Concatenate** 🔴 HIGH RISK (AVMutableComposition + resolution normalize) | ⚠️ Logic giống UIKit | ⚠️ Logic giống SwiftUI | 1.5 ngày |
| **Export** 🔴 HIGH RISK (AVAssetWriter + background task) | ⚠️ Nặng, manageable | ✅ Tốt hơn | 1 ngày |

**Tổng ước tính:** ~7.5 ngày làm việc → cần discipline về scope.

---

## SwiftUI — Phân tích chi tiết

**Khả thi nhưng có friction với AVFoundation-heavy features.**

### Điểm mạnh

- Edit tool UI (sliders, buttons, picker) — native SwiftUI, viết nhanh
- Timeline clip list dùng `ScrollView + HStack` tốt
- `PHPickerViewController`, Sheet, `NavigationStack` — smooth
- Boilerplate ít hơn UIKit cho UI thuần

### Điểm yếu

- `MTKView` cần `UIViewRepresentable` wrapper — không quá khó nhưng tốn boilerplate debug khi có vấn đề timing/layout
- `AVPlayerLayer` cần `UIViewRepresentable` — thêm một lớp abstraction
- **Crop gesture là vấn đề nghiêm trọng nhất**: `UIPinchGestureRecognizer` conflict với SwiftUI gesture system. SwiftUI intercept gesture trước UIKit và behavior không predictable khi hai hệ thống gesture chạy đồng thời → debugging tốn thời gian không cần thiết trong 1 tuần
- Không có SwiftUI equivalent cho custom gesture + Metal preview đồng thời trên cùng một view

---

## UIKit — Phân tích chi tiết

**Recommended cho time bound 1 tuần với AVFoundation-heavy workload.**

### Điểm mạnh

- `MTKView` drop-in, không cần wrapper
- `UIPinchGestureRecognizer` + `UIPanGestureRecognizer` — full control, predictable
- `AVPlayerLayer` trong `UIView` — standard pattern, nhiều tài liệu
- Custom scrubber UI bằng `UISlider` + `UIControl` — predictable behavior
- `UIScrollView` cho clip timeline — tested in production
- Toàn bộ AVFoundation APIs được thiết kế xung quanh UIKit lifecycle

### Điểm yếu

- Boilerplate nhiều hơn SwiftUI
- Layout code bằng tay hoặc Auto Layout — chậm hơn khi viết

---

## Verdict — Hybrid Approach

> **Dùng UIKit làm core** (MTKView, gesture, AVPlayer, Export pipeline) + nhúng **SwiftUI cho UI thuần** (settings panel, slider controls, PHPicker) qua `UIHostingController`. Đây là hybrid approach thực tế nhất trong 1 tuần.

```swift
// Ví dụ nhúng SwiftUI panel vào UIKit view controller
let adjustmentView = AdjustmentPanelView(params: $clipParams)
let hostingVC = UIHostingController(rootView: adjustmentView)
addChild(hostingVC)
view.addSubview(hostingVC.view)
hostingVC.didMove(toParent: self)
```

---

## Gap Analysis — Những gì cần xây mới

Những phần dưới đây **không reuse được** từ Metal camera pipeline và phải xây từ đầu.

### 🔴 Clip Data Model + Timeline State Management

Phải xây từ đầu. Không có sẵn trong AVFoundation hay Metal. Nếu model sai, toàn bộ preview/export consistency sẽ fail. Ước tính 0.5 ngày nhưng ảnh hưởng toàn bộ app.

```swift
struct Clip: Identifiable {
    let id: UUID
    let asset: AVAsset
    var trimRange: CMTimeRange
    var cropRect: CGRect           // normalized 0–1
    var adjustments: AdjustmentParams
    var filter: FilterType
    var preferredTransform: CGAffineTransform
}

struct Timeline {
    var clips: [Clip]
    var outputResolution: CGSize
    var outputAspectRatio: AspectRatio
}
```

### 🔴 Crop Gesture + UV Shader Integration

`UIPinchGestureRecognizer` + `UIPanGestureRecognizer` trên `MTKView` preview, normalize về crop rect `[0,1]`, pass vào Metal shader uniform. Gesture recognizer conflict là rủi ro lớn nhất trong 1 tuần — đặc biệt nếu dùng SwiftUI.

```swift
// UIKit approach — predictable
let pinch = UIPinchGestureRecognizer(target: self, action: #selector(handlePinch))
let pan   = UIPanGestureRecognizer(target: self, action: #selector(handlePan))
mtkView.addGestureRecognizer(pinch)
mtkView.addGestureRecognizer(pan)
pinch.require(toFail: pan) // chain gesture recognizers
```

### 🔴 AVAssetWriter Export Pipeline đầy đủ

`AVAssetReader → Metal render per frame → AVAssetWriterInputPixelBufferAdaptor`. Audio passthrough riêng. Dễ có edge case (H.265 decode, orientation, empty audio track). Cần test kỹ ngay từ ngày 5.

```
AVAssetReader (decode source frames)
↓ CVPixelBuffer per frame
MetalFrameRenderer (apply filter/adjustment/crop)
↓ CVPixelBuffer output
AVAssetWriterInputPixelBufferAdaptor
↓
AVAssetWriter → .mp4 → PHPhotoLibrary
```

### 🟡 Concatenate — Resolution Normalization

Video từ Photos có thể 4K, 1080p, 720p mix lẫn nhau. `AVMutableComposition` + `AVMutableVideoCompositionLayerInstruction` để fit về `outputResolution`. Orientation mismatch giữa các clips (portrait + landscape) là gotcha hay gặp nhất.

```swift
// Normalize mỗi clip về outputResolution
let transform = fitTransform(
    from: videoTrack.naturalSize,
    to: timeline.outputResolution,
    preferredTransform: clip.preferredTransform
)
layerInstruction.setTransform(transform, at: insertTime)
```

### 🟡 Preview / Export WYSIWYG Consistency

Shared `MetalFrameRenderer` phải nhận cùng `RenderParams` từ `Clip` model cho cả preview lẫn export. Nếu hai path dùng renderer khác nhau, output export sẽ trông khác preview — khó debug cuối tuần.

```swift
// Protocol dùng chung cho Preview và Export
protocol FrameProcessor {
    func process(_ pixelBuffer: CVPixelBuffer, params: RenderParams) -> MTLTexture?
}

// Preview: AVPlayerItemVideoOutput → MetalFrameRenderer → MTKView
// Export:  AVAssetReader           → MetalFrameRenderer → AVAssetWriter
// Cùng một instance renderer, cùng RenderParams từ Clip model
```

### 🟢 Filter (CIFilter) + Adjustment (Metal Uniforms) — Reuse tốt nhất

Phần này reuse Metal pipeline tốt nhất. `CIFilter` cho filter (B&W, Sepia, Noir, Fade) không cần LUT. Adjustment shader viết một lần, dùng cho cả preview lẫn export. Ít rủi ro nhất, nên implement sớm (ngày 2).

```swift
// Filter — CIFilter approach (đơn giản hơn LUT cho MVP)
func applyFilter(_ type: FilterType, to image: CIImage) -> CIImage {
    switch type {
    case .blackAndWhite: return image.applyingFilter("CIPhotoEffectMono")
    case .sepia:         return image.applyingFilter("CISepiaTone", parameters: ["inputIntensity": 0.8])
    case .noir:          return image.applyingFilter("CIPhotoEffectNoir")
    case .fade:          return image.applyingFilter("CIPhotoEffectFade")
    case .none:          return image
    }
}
```

### 🟢 Trim UI — Custom Scrubber

`UISlider` + thumbnail strip. Frame-accurate seek với `toleranceBefore: .zero` chậm hơn nhưng đúng. Dùng tolerance nhỏ để scrub mượt và seek chính xác khi thả tay (`.touchUpInside`).

```swift
player.seek(
    to: targetTime,
    toleranceBefore: .zero,                          // frame accurate
    toleranceAfter: CMTime(value: 1, timescale: 600) // nhỏ thôi
)
```

---

## Kế hoạch 7 ngày — Time-bound Implementation

| Ngày | Tasks | Loại |
|---|---|---|
| **Ngày 1** | Project setup + UIKit skeleton, PHPicker select video, Clip + Timeline model, AVPlayer preview (MTKView) | Core |
| **Ngày 2** | Metal renderer (stateless), Adjustment shader, CIFilter pipeline, Filter + Adjustment UI (sliders) | Core + UI |
| **Ngày 3** | Trim scrubber UI, CMTimeRange + seek, **Crop gesture** (pinch+pan), UV crop shader | ⚠️ Risk |
| **Ngày 4** | Ratio presets, **Concatenate** (AVMutableComposition), Resolution normalize, Orientation fix | ⚠️ Risk |
| **Ngày 5** | **AVAssetWriter export**, Audio passthrough, Progress tracking, Save to Photos | ⚠️ Risk |
| **Ngày 6** | Edit UI polish, Export preview, WYSIWYG verify, Edge case testing | Polish + Test |
| **Ngày 7** | Bug fix buffer, Orientation test với nhiều loại video, Export quality check, Final polish | Test + Buffer |

### Nguyên tắc quản lý thời gian

- Hoàn thành **Filter + Adjustment trước** (ngày 2) — ít rủi ro, xây được foundation renderer
- Setup **shared `MetalFrameRenderer`** ngay từ đầu — không làm preview path riêng rồi copy sang export
- **Test export với nhiều loại video** từ ngày 5, không đợi đến ngày 7
- Giới hạn concatenate **2–3 clips** thay vì không giới hạn để tránh edge case
- **Ngày 7 là buffer** — không schedule feature mới vào ngày này

---

## Scope Cut Theo Priority

### Phải có (non-negotiable)

- Select video từ Photos
- Preview video realtime
- Filter (B&W, Sepia, Noir, Fade)
- Adjustment (Brightness, Contrast, Saturation)
- Trim (start/end time)
- Export cơ bản
- Save to Photos

### Nên có (nếu đủ thời gian)

- Ratio preset (0.5 ngày, đơn giản — nên làm)
- Concatenate 2–3 video (1.5 ngày, cần test kỹ)
- Preview video trước khi lưu

### Cắt nếu bị trễ

> **Crop** — complex nhất, gesture rủi ro nhất. Thay bằng **fixed center-crop theo ratio** (1 dòng code) và note rõ limitation. Đừng để crop block các feature quan trọng hơn.

```swift
// Fixed center crop thay thế — safe, 1 dòng
let centerCrop = CGRect(x: 0.1, y: 0.1, width: 0.8, height: 0.8) // 80% center
clip.cropRect = centerCrop
```

---

## Architectural Direction — Tóm tắt

```
┌─────────────────────────────────────────────┐
│              UIKit Core Layer               │
│  MTKView / AVPlayer / Gesture Recognizers   │
│  AVMutableComposition / AVAssetWriter       │
└──────────────────────┬──────────────────────┘
                       │ UIHostingController
┌──────────────────────▼──────────────────────┐
│           SwiftUI UI Panels (nhúng)         │
│  AdjustmentPanel / FilterPanel / RatioPicker│
│  PHPickerViewController wrapper             │
└─────────────────────────────────────────────┘
                       │ Shared
┌──────────────────────▼──────────────────────┐
│         MetalFrameRenderer (Stateless)      │
│  Input: CVPixelBuffer + RenderParams        │
│  Output: MTLTexture                         │
│  Dùng chung: Preview ↔ Export              │
└─────────────────────────────────────────────┘
```

### Tech Stack cuối cùng

| Layer | Technology | Framework |
|---|---|---|
| Video select | `PHPickerViewController` | PhotosUI |
| Playback | `AVPlayer` + `AVPlayerItemVideoOutput` | AVFoundation |
| Frame display | `MTKView` | Metal |
| Gesture (Crop) | `UIPinchGestureRecognizer`, `UIPanGestureRecognizer` | UIKit |
| GPU rendering | Metal shader + `CVPixelBuffer` | Metal |
| Filter | `CIFilter` (MVP) → LUT texture (v2) | CoreImage / Metal |
| Trim | `CMTimeRange` + `AVPlayer.seek` | AVFoundation |
| Composition | `AVMutableComposition` | AVFoundation |
| Orientation | `AVMutableVideoCompositionLayerInstruction` | AVFoundation |
| Export encode | `AVAssetWriter` + `AVAssetWriterInput` | AVFoundation |
| Export decode | `AVAssetReader` + `AVAssetReaderOutput` | AVFoundation |
| Audio | Passthrough `AVAssetWriterInput` | AVFoundation |
| Background task | `UIBackgroundTaskIdentifier` | UIKit |
| Save to Photos | `PHPhotoLibrary` | Photos |
| UI panels | SwiftUI via `UIHostingController` | SwiftUI |

---

_Tài liệu phân tích implementation — Video Editor Mini App iOS, time bound 1 tuần_
