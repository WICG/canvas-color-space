# Color managing canvas contents

## Use Case Description
* Contents displayed through a canvas element should be color managed in order to minimize differences in appearance across browsers and display devices. Improving color fidelity matters a lot for artistic uses (e.g. photo and paint apps) and for e-commerce (product presentation).
* Canvases should be able to take advantage of the full color gamut and dynamic range of the display device.

### Current Limitations
* The color space of canvases is undefined in the current specification, though de facto sRGB.
* The bit-depth of canvases is currently fixed to 8 bits per component, which is below the capabilities of some monitors. Monitors with higher contrast ratios require more bits per component to avoid banding.
* Color encoding and blending is de facto perceptually-linear unorm8, which allocates too little precision on low values (and too much precision on high values), leading to poor handling of dark scenes and content.

### Current Usage and Workarounds
The lack of color space interoperability is hard to work around. For some browser implementations which color-correct images drawn to canvases by applying the display profile, apps that want to use canvases for color corrected image processing are stuck doing convoluted workarounds, such as:
* reverse-engineer the display profile by drawing test pattern images to the canvas and inspecting the color corrected result via getImageData
* bypass CanvasRenderingContext2D.drawImage() and use image decoders implemented in JavaScript to extract raw image data that was not tainted by the browser's color correction behavior.''

An aspect of current implementations that is interoperable is that colors match between CSS/HTML and canvases:
* A color value used as a canvas drawing style will have the same appearance as if the same color value were used as a CSS style
* An image resource drawn to a canvas element will have the same appearance as if it were displayed as the replaced content of an HTML element or used as a CSS style value.

This color matching behavior needs to be preserved to avoid breaking pre-existing content.

Some implementations convert images drawn to canvases to the sRGB color space. This has the advantage of making the color correction behavior device independent, but it clamps the gamuts of the rendered content to the sRGB gamut, which is significantly narrower than the gamuts of some current consumer devices.

### Requests for this Feature

* <cite>[https://github.com/whatwg/html/issues/299]</cite> <blockquote><p>Allow 2dcontexts to use deeper color buffers</p></blockquote>
* <cite>[https://bugs.chromium.org/p/chromium/issues/detail?id=425935]</cite> <blockquote><p>Wrong color profile with 2D canvas</p></blockquote>
* Engineers from the Google Photos, Maps and Sheets teams have expressed a desire for canvases to become color managed.  Particularly for the use case of resizing an image, using a canvas, prior to uploading it to the server, to save bandwidth. The problem is that the images retrieved from a canvas are in an undefined color space and no color space information is encoded by toDataURL or toBlob.

## Background

Color spaces are generally tuples of:
* Chromaticity coordinates (X,Y in CIE 1931 space)
  * Primaries (coords for 1.0 for each of R/G/B)
  * White-point (coord for white)
* Transfer function (often "gamma")

Within the same chromaticity, 0.0 will always be min-brightness and 1.0 is max-brightness.
However, because of the transfer function, 0.5 will not be the average physical brightness of 0.0 and 1.0.
You can see this by making stripes of 1.0 and 0.0, and matching that pattern's brightness to a solid grey background.
(On my machine, striped 0/1/0/1 appears equally bright as #ACACAC, or 0.67)

## Proposed Solution

* Clearly define the color space of canvases.  Most current browser implementations are either color-managed or are currently actively working on becoming color-managed.  Therefore, it will be possible to specify that the default color space of canvases as 'srgb' instead of leaving it undefined in the spec (as is currently the case due to lack of interoperability).
* Add a canvas context creation attribute to specify a color space.
* Add a canvas context creation attribute to specify an encoding format for storing pixel values.
* Color space and encoding format parameters are also extended to image storage interfaces that interact with canvases, such as CanvasPattern, ImageData and ImageBitmap.

### Processing Model

IDL:
<pre>
// Feature enums:

enum CanvasColorSpaceEnum {
  "srgb", // default
  "rec-2020",
};

enum CanvasColorEncodingEnum {
  "unorm8",      // default, 0.5 encoded as 0x80
  "unorm8-srgb", // 0.5 encoded as 0xbc
  "float16",     // IEEE 754
};

// Feature detection:

interface CanvasColorSpace {
  const CanvasColorSpaceEnum srgb = "srgb";
  const CanvasColorSpaceEnum rec2020 = "rec-2020";
};

interface CanvasColorEncoding {
  const CanvasColorEncodingEnum unorm8 = "unorm8";
  const CanvasColorEncodingEnum unorm8Srgb = "unorm8-srgb";
  const CanvasColorEncodingEnum float16 = "float16";
};

// Feature activation:

partial dictionary CanvasRenderingContext2DSettings {
  CanvasColorSpaceEnum colorSpace = "srgb";
  CanvasColorEncodingEnum colorEncoding = "unorm8";
};

partial dictionary WebGLContextAttributes {
  CanvasColorSpaceEnum colorSpace = "srgb";
  CanvasColorEncodingEnum colorEncoding = "unorm8";
};

partial interface CanvasRenderingContext2D {
  CanvasRenderingContext2DSettings getContextAttributes();
};
</pre>

Example:
<pre>
canvas.getContext('2d', { colorSpace: "rec2020",
                          colorEncoding: "float16"} );
</pre>

#### The colorSpace canvas creation parameter

* Color spaces match their respective counterparts as defined in the [CSS colorspaces](https://www.w3.org/TR/css-color-4/#predefined).
* All input colors (e.g, fillStyle or strokeStyle, and gradient stops) follow the same interpretation as CSS color literals, regardless of canvas color space.
* Values written to WebGL backbuffers (e.g, values written to gl_FragColor or the clear color) are in the canvas's color space.
* Images with no color profile, when drawn to the canvas, are assumed to be in the sRGB color space.
* Unless otherwise explicitly specified by the user, toDataURL/toBlob will produce resources in sRGB color space, with unorm8 encoding (matching existing behavior). If the destination image format supports colorspace tagging or embedded color profiles, the resource will be tagged as being in sRGB color space.

##### The "srgb" color space
* This color space matches the existing canvas behavior.
* Guarantees that color values used as fillStyle or strokeStyle exactly match the appearance of the same color value when it is used in CSS.
* On implementations that do not color-manage CSS colors, the canvas "srgb" color space must not be color-managed either, in order to preserve color-matching between CSS and canvas-rendered content. This situation shall be referred to as the "legacy behavior".
* All content drawn into such a 2d canvas RC must be color corrected to sRGB.
    * Exception: User agents that implement the legacy behavior must apply color correction steps that match the color correction that is applied to image resources that are displayed via CSS.
* Displayed canvases must be color corrected for the display if a display color profile is available. This color correction happens downstream at the compositing stage, and has no script-visible side-effects.
* toDataURL/toBlob produce resources tagged as being in the sRGB color space, if the encoding format supports colorspace tagging or embedded color profiles.
    * Exception: User agents that implement the legacy behavior must not encode any color space metadata.

##### The "rec-2020" color space
* As per the [CSS rec-2020 color space](https://www.w3.org/TR/css-color-4/#valdef-color-rec-2020).
* This color space has different primaries and a different transfer function than "srgb".
* Support is optional, and should not be present on srgb-only UAs.


#### The colorEncoding context creation attribute
The colorEncoding attributes specifies the encoding to be used for storing pixel channel color values.
* Support for "unorm8" is mandatory. All other encodings are optional.
* When an unsupported encoding is requested, the encoding shall fall back to "unorm8".
* The alpha channel is always interpreted as if clamped to [0,1].
* Float RGB channel values outside of [0,1] range can be used to represent colors outside of the chosen color gamut. This allows float pixel formats to represent all possible colors and brightness levels. How values outside of [0,1] are displayed depends on the capabilites of the device and output display. Some implementations may simply clamp these values to [0,1]. If the device and display are capable, a (perceptually-linear) pixel value of (2,2,2) should be twice as bright as (1,1,1).
* Operations on encoded values always operate on decoded values, not the encoded bits.
    * I.e. in "unorm8-srgb" encoding, 0xff (1.0) minus 0xbc (0.5) equals 0xbc (0.5).


#### Selecting the best color space match for the user agent's display device
<pre>
var colorSpace = window.matchMedia("(color-gamut: rec2020)").matches ? "rec2020" :
    (window.matchMedia("(color-gamut: p3)").matches ? "p3" : "srgb");
</pre>

#### Selecting the best encoding for the user agent's display device
Selection should be based on the best color space match (see above). For srgb, at least 8 bits per component is recommended; for p3, 10 bits; and for rec2020, 12 bits.  The float16 format is suitable for any colorspace.  There may soon be a proposal to add a way of detecting HDR displays, possibly something like "window.screen.isHDR()" (TBD), which would be a good hint to use the float16 format.

#### Non-standard color spaces
For future consideration: support could be added for color space defined using the [CSS @color-profile rule](https://www.w3.org/TR/css-color-4/#at-profile).

#### Compositing the canvas element
Canvas contents are composited in accordance with the canvas element's style (e.g. CSS compositing and blending rules). The necessary compositing operations must be performed in an intermediate colorspace, the compositing space, that is implementation specific. The compositing space must have sufficient precision and a sufficiently wide gamut to guarantee no undue loss of precision or gamut clipping in bringing the canvas's contents to the display. Implementations should not expose color spaces that are unreasonble for the display.

#### Feature detection
2D rendering contexts are to expose a new getContextAttributes() method, that works much like the method of the same name on WebGLRenderingContext. The method returns the "actual context attributes" which represents the settings that were successfully applied at context creation time. The settings attribute reflects the result of running the algorithm for coercing the settings argument for 2D contexts, as well as the result of any fallbacks that may have happened as a result of options not being supported by the UA.

Web apps may infer that a user agent that does not implement getContextAttributes() does not support the colorSpace and pixelFormat attributes.

Note: An alternative approach that was considered was to augment the probablySupportsContext() API by making it check the second argument.  That approach is difficult to consolidate with how dictionary arguments are meant to work, where unsupported entries are just ignored.

#### ImageBitmap

ImageBitmap objects (unless created with ``colorSpaceConversion="none"``) should keep track of their internal color space, and should store their contents at highest fidelity possible, subject to implementation limitations.

#### ImageData

Add the following types to be used by `ImageData`.
<pre>
enum ImageDataEncoding {
  "uint8", // default
  "uint16",
  "float32",
};
dictionary ImageDataSettings {
  CanvasColorSpaceEnum colorSpace = "srgb";
  ImageDataStorageType encoding = "uint8";
};
typedef (Uint8ClampedArray or Uint16Array or Float32Array) ImageDataArray;
</pre>

Update the `ImageData` interface to the include the following.
<pre>
partial interface ImageData {
  constructor(unsigned long sw, unsigned long sh, optional ImageDataSettings);
  constructor(ImageDataArray data, unsigned long sw, unsigned long sh, optional ImageDataSettings);
  readonly ImageDataSettings getImageDataSettings();
  readonly attribute ImageDataArray data;
};
</pre>

The changes to this interface are:
* The constructors now take an optional `ImageDataSettings` dictionary.
* The ImageDataSettings attribute may be queried using `getImageDataSettings`.
* The constructor and attribute that used to be a `Uint8ClampedArray` are now a `ImageDataArray` union, which can specify data in multiple formats.

The type of the ``data`` attribute is determined by the ``encoding`` parameter according to the following table.

| ``encoding`` Value | ``data`` Type |
|-|-|
| ``"uint8"`` | ``Uint8ClampedArray`` |
| ``"uint16"`` | ``Uint16Array`` |
| ``"float32"`` | ``Float32Array`` |


The constructor that takes both an `ImageDataArray` and an `ImageDataSettings` will throw an exception if the type of the `ImageDataArray` is incompatible with the type specified in `ImageDataSettings` (e.g, `ImageDataArray` is a `Float32Array`, but `ImageDataSettings` specifies an encoding that is set to `"uint8"`).

When an ``ImageData`` is used in a canvas (e.g, in ``putImageData``), the data is converted from the ``ImageData``'s color space to the color space of the canvas.

To the ``CanvasRenderingContext2D`` interface, add
<pre>
partial interface CanvasRenderingContext2D {
    ImageData createImageData(long sw, long sh, optional ImageDataSettings settings);
    ImageData getImageData(long sx, long sy, long sw, long sh, optional ImageDataSettings settings);
}
</pre>

The changes to this interface are the addion of the optional ``ImageDataSettings`` argument. If this argument is unspecified, then the default values of ``encoding="uint8"`` and ``colorSpace="srgb"`` will be used (these defaults match previous behavior).

The ``getImageData`` method is responsible for converting the data from the canvas' internal format to the format requested in the ``ImageDataSettings``.


### Limitations
* toDataURL and toBlob may be lossy, depending on the file format, when used on a canvas that has an encoding other than ``"uint8"``. Possible future improvements could solve or mitigate this issue by adding more file formats or adding options to specify the resource color space.

### Adoption
Lack of color management and color interoperability is a longstanding complaint about the canvas API.
Authors of games and imaging apps are expected to be enthusiastic adopters.

## Unresolved Issues

* Should we support custom color spaces based on ICC profiles? Would offer ultimate flexibility. Would be hard to make implementations as efficient as built-in color spaces, in particular for implement linearPixelMath for profiles that have arbitrary transfer curves.

* Should float16 allow alpha values outside [0,1] range at any stage of the pipeline? What would they mean?

* Should there be API-level support for mixing chromaticities and transfer functions, including use of no-op transfer functions?

* Should it be "rgba8" and "rgba16f" instead of "unorm8" and "float16"?

* Should context creation throw on an unrecognized, non-undefined creation attribute?

## Proposal History

This propsal was originally incubated in the WHATWG github issue tracker and incorporates feedback that was provided in the following thread: https://github.com/whatwg/html/issues/299

This proposal was further discussed in the Khronos WebGL working group, with the participation of engineers from Apple, Google, Microsoft, Mozilla, Nvidia, and others.

From 2016 to 2017, it was discussed on the [W3C WICG Discourse thread](https://discourse.wicg.io/t/canvas-color-spaces-wide-gamut-and-high-bit-depth-rendering/1505).

The current venue for discussing this proposal is in issues and pull requests to the [WICG canvas-color-space GitHub repo](https://github.com/WICG/canvas-color-space).
