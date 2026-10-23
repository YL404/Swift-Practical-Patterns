# Scaling PDF image assets

Use this reference when an image imported as a PDF becomes soft as its SwiftUI size grows. First distinguish the source artwork, asset-catalog configuration, and the representation being scaled.

Apple documentation and WWDC guidance verified on 2026-10-23. Runtime vector preservation was introduced in iOS 11 with Xcode 9; the configuration below targets iOS/iPadOS apps. The SwiftUI example requires iOS 13. The JSON format also describes named `NSImage` assets, but macOS rendering behavior is outside this reference's verification scope.

## Keep one source and preserve its geometry when needed

- **Single Scale** supplies one PDF source across display scales. It does not by itself preserve vector data at runtime.
- **Preserve Vector Data** sets `properties.preserves-vector-representation` to `true`. It retains the PDF's vector information. For an image that can grow beyond its natural size, enable it; a fixed-size asset does not need it solely because its source is a PDF.

Apple describes generating optimized raster representations during asset compilation, retaining those for normal sizes, and using preserved vector data to rasterize a larger image when needed. This is not a promise that every presentation or transform redraws the PDF. [Asset-catalog format](https://developer.apple.com/library/archive/documentation/Xcode/Reference/xcode_ref-Asset_Catalog_Format/ImageSetType.html), [runtime behavior and tradeoff](https://developer.apple.com/videos/play/wwdc2018/227/)

For a new universal image set containing `Illustration.pdf`, this is a complete minimal `Contents.json`. For an existing asset, retain its other properties and appearance/device variants; do not replace its metadata wholesale. Omitting `scale` represents a PDF for any display scale.

```json
{
  "images": [
    { "filename": "Illustration.pdf", "idiom": "universal" }
  ],
  "info": { "author": "xcode", "version": 1 },
  "properties": { "preserves-vector-representation": true }
}
```

This SwiftUI fragment assumes that image set is compiled into the app's asset catalog. The frame is an example display size, not a PDF export requirement:

```swift
Image("Illustration")
    .resizable()
    .scaledToFit()
    .frame(width: 320, height: 320)
```

## Diagnose the remaining blur

Check whether the source contains vector paths or embedded raster images. A PDF container does not convert a bitmap into vector geometry; preserving it cannot add source detail. Then inspect the actual image pipeline: converting the asset to a fixed-size bitmap or flattening it before a later enlargement changes what is being scaled. Do not assume `.resizable()`, an interpolation setting, or a larger export can restore discarded detail. Inspect the asset's use at its actual rendered size.

Even vector geometry can produce soft edges when thin strokes land between physical pixel boundaries. Apple discusses adjusting artwork to the target display grid and, where necessary, providing tuned scale variants. This is a separate diagnosis from missing vector preservation. [Artwork alignment](https://developer.apple.com/videos/play/wwdc2018/227/)

## Verification boundary

The exact JSON parsed successfully and a temporary image set containing a simple 160-by-160-point vector PDF compiled with Xcode 27.1 `actool` for the `iphoneos` platform, iPhone device family, and iOS 13 minimum deployment target. The SwiftUI fragment, wrapped in a view, passed Swift 6 typechecking with complete strict concurrency, nonisolated default isolation, and warnings as errors using iPhoneOS SDK 27.1 targeting arm64 iOS 13; a macOS SDK 27.0 / arm64 macOS 11 check also passed for the fragment.

A local `assetutil --info` comparison reported raster renditions and a Vector entry for both enabled and disabled preservation inputs. The entries differed in scale metadata, so merely finding a Vector entry is not evidence that the runtime policy is enabled. No installed iOS app, resized image output, bitmap-only PDF, Dynamic Type/large content viewer, older runtime, or performance was tested. Runtime recommendations rely on the linked Apple contract and WWDC explanation, not a reproduced blur comparison.

## Sources

- https://livsycode.com/best-practices/why-your-vector-pdf-looks-blurry/
