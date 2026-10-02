# Universal Cross-Android PDF Search with Visual Text Highlighting

Fix PDF search on Android 13 (Moto G52) and all Android versions below API 35 by introducing a universal dual-engine search architecture, and implement visual on-page highlighting of matched text across both Android 15+ and legacy Android versions.

## User Review & Critical Decisions

> [!IMPORTANT]
> The following product decisions were confirmed during review:
> - **Universal Cross-Android Engine**:
>   - **Android 15+ (API 35+)**: Leverages platform-native `PdfRenderer.Page.searchText(query)` returning `List<PageMatchBounds>`.
>   - **Android 13 and below (API 24–34)**: Leverages `com.tom-roush:pdfbox-android:2.0.27.0` using a position-aware text stripper to decode full font encodings, CMaps, and compute exact character coordinates.
> - **Visual On-Page Highlighting**:
>   - Renders a semi-transparent highlight overlay (translucent golden amber `#FFD54F` at 40% opacity) over all matched words on the currently viewed page.
>   - The **currently active match** is highlighted with a brighter gold fill and a crisp accent focus border so the user instantly spots the target word.
>   - Navigating via Previous/Next (▲/▼) smoothly shifts the focus indicator to each match occurrence in real time.

---

## 1. Problem Analysis & Root Cause

1. **Android 13 / Moto G52 Zero Results**:
   - `PdfTextSearchManager.kt` checked `Build.VERSION.SDK_INT >= Build.VERSION_CODES.VANILLA_ICE_CREAM` (API 35).
   - On Android 13 (API 33), the check failed and returned `emptyList()`.
   - Android's stock `PdfRenderer` on API < 35 lacked public search or text extraction methods.
2. **Missing Text Highlighting**:
   - Matches previously jumped to the target page, but did not visually highlight the exact coordinates of the word on the PDF page canvas.

---

## 2. Proposed Architecture & Solution

```
                            ┌───────────────────────────────────┐
                            │        PdfTextSearchManager       │
                            │   searchStreaming(query, callback)│
                            └─────────────────┬─────────────────┘
                                              │
                      ┌───────────────────────┴───────────────────────┐
                      │                                               │
             Build.VERSION >= 35                             Build.VERSION < 35
                      │                                               │
                      ▼                                               ▼
         ┌─────────────────────────┐                     ┌─────────────────────────┐
         │ Native Platform Engine  │                     │  Universal PDFBox Engine│
         │ - PdfRenderer.Page      │                     │  - com.tom-roush:       │
         │ - page.searchText(query)│                     │    pdfbox-android       │
         │ - Extracts bounds       │                     │  - Coordinate Stripper  │
         │   (RectF per match)     │                     │  - Full CMap & Font Map │
         └────────────┬────────────┘                     └────────────┬────────────┘
                      │                                               │
                      └───────────────────────┬───────────────────────┘
                                              │
                                              ▼
                             ┌─────────────────────────────────┐
                             │ SearchMatch with RectF Bounds   │
                             │ (pageIndex, bounds, snippet)    │
                             └────────────────┬────────────────┘
                                              │
                                              ▼
                             ┌─────────────────────────────────┐
                             │      SinglePagePdfViewer        │
                             │ Canvas Highlight Overlay:       │
                             │ - All matches: Amber 40%        │
                             │ - Active match: Amber 80% + ring│
                             └─────────────────────────────────┘
```

### 2.1 Dependencies
- `app/build.gradle.kts`:
  ```kotlin
  implementation("com.tom-roush:pdfbox-android:2.0.27.0")
  ```

### 2.2 Coordinate Calculation & Data Model
- Extend `SearchMatch`:
  ```kotlin
  data class MatchRect(val left: Float, val top: Float, val right: Float, val bottom: Float)
  data class SearchMatch(
      val pageIndex: Int,
      val matchIndexOnPage: Int = 0,
      val bounds: List<MatchRect> = emptyList(),
      val snippet: String = ""
  )
  ```
- **Android 15+**: `PageMatchBounds.bounds` directly provides the bounding rectangles.
- **Android 13**: Custom `PDFTextStripper` records `TextPosition` coordinates for matching substrings and converts them to normalized page fractions `(0f..1f)`.

### 2.3 On-Page Highlight Layer (`SinglePagePdfViewer.kt`)
- On top of the rendered PDF page image inside `SinglePagePdfViewer`, an overlay `Canvas` reads the matches for the current page:
  - Draws `drawRoundRect` for each match box scaled to the rendered image aspect ratio and zoom scale.
  - Distinguishes between secondary matches (subtle yellow/amber) and the active selected match (prominent amber highlight with accent outline).
  - Tapping ▲ or ▼ immediately updates `currentActiveMatchIndex`, updating the visual highlight in real time.

---

## 3. Implementation Steps

1. **Add Dependency**:
   - Add `com.tom-roush:pdfbox-android:2.0.27.0` to `app/build.gradle.kts`.
2. **Implement Dual-Engine Search (`PdfTextSearchManager.kt`)**:
   - Android 15+: uses `page.searchText(query)` + extracts `PageMatchBounds`.
   - Android 13 & below: uses `PDDocument` + custom coordinate-aware `PDFTextStripper`.
3. **Connect Highlights to Reader (`SinglePagePdfViewer.kt` & `ReaderActivity.kt`)**:
   - Expose current active search matches to `SinglePagePdfViewer`.
   - Render highlight rectangles over the matched text coordinates with zoom/pan support.
4. **Verification**:
   - Compile and execute tests.
   - Validate on both Android 15 and Android 13 devices.
