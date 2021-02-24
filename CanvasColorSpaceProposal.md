# Color managing canvas contents

## Use Case Description
* Contents displayed through a canvas element should be in a well-defined color space to minimize differences in appearance across browsers and display devices.
* Canvases should be able to take advantage of the full color gamut and dynamic range of the display device.
* Contents displayed through a canvas element should be color managed in order to minimize differences in appearance across browsers and display devices. Improving color fidelity matters for artistic (e.g, photo and paint apps) and for e-commerce (e.g, product presentation) use cases.

### Current Limitations
* The color space of canvases is undefined in the current specification, although is de facto sRGB.
* The bit-depth of canvases is likewise undefined, although is de facto 8 bits per component.
* Images retrieved from a canvas are in an undefined color space and no color space information is encoded by ``toDataURL`` or ``toBlob``.

### Requests for this Feature

* <cite>[https://github.com/whatwg/html/issues/299]</cite> <blockquote><p>Allow 2dcontexts to use deeper color buffers</p></blockquote>
* <cite>[https://bugs.chromium.org/p/chromium/issues/detail?id=425935]</cite> <blockquote><p>Wrong color profile with 2D canvas</p></blockquote>
* Engineers from the Google Photos, Maps and Sheets teams have expressed a desire for canvases to be color managed. Particularly for the use case of resizing an image, using a canvas, prior to uploading it to the server, to save bandwidth. The problem is that the images retrieved from a canvas are in an undefined color space and no color space information is encoded by toDataURL or toBlob.

### Related Specifications
* The [CSS Media Queries Level 5](https://www.w3.org/TR/mediaqueries-5/#color-gamut) specification defines the ``color-gamut`` media query to determine the capabilities of the current display device.
* The [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification defines several color spaces that will be used in this specification. At time of writing this specification is partially implemented on some browsers.

## Proposed Solution

* Clearly define the default color space of canvases to be ``srgb``, and the default encoding to be 8 bits per pixel.
* Add canvas context creation attributes to specify the color space, storage format, and color encoding function of a canvas.
* Clarify behavior for drawing into canvases, compositing canvases, and exporting canvas contents.
* Add mechanisms to specify the color space and encoding of ``ImageData`` objects.

### Processing Model

IDL:
<pre>
// Feature enums:

enum CanvasColorSpaceEnum {
  "srgb", // default
  "display-p3",
  "rec-2020",
};

enum CanvasStorageFormatEnum {
  "unorm8",      // default
  "unorm8-srgb", // Same as unorm8, but with an sRGB color encoding function
  "unorm16",
  "float16",     // IEEE 754
};

// Feature detection:

interface CanvasColorSpace {
  const CanvasColorSpaceEnum srgb = "srgb";
  const CanvasColorSpaceEnum displayP3 = "display-p3";
  const CanvasColorSpaceEnum rec2020 = "rec-2020";
};

interface CanvasStorageFormat {
  const CanvasStorageFormatEnum unorm8 = "unorm8";
  const CanvasStorageFormatEnum unorm8Srgb = "unorm8-srgb";
  const CanvasStorageFormatEnum unorm16 = "unorm16";
  const CanvasStorageFormatEnum float16 = "float16";
};

// Feature activation:

partial dictionary CanvasRenderingContext2DSettings {
  CanvasColorSpaceEnum colorSpace = "srgb";
  CanvasStorageFormatEnum storageFormat = "unorm8";
  boolean highDynamicRange = false;
};

partial dictionary WebGLContextAttributes {
  CanvasColorSpaceEnum colorSpace = "srgb";
  CanvasStorageFormatEnum storageFormat = "unorm8";
  boolean highDynamicRange = false;
};
</pre>

Example:
<pre>
canvas.getContext('2d', { colorSpace: "rec2020",
                          storageFormat: "float16"} );
</pre>

#### The ``colorSpace`` canvas creation attribute

The ``colorSpace`` attribute specifies the color space for the backing storage of the canvas.
* Color spaces match their respective counterparts as defined in the [predefined color spaces](https://www.w3.org/TR/css-color-4/#predefined) section of the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification.
* When an unsupported color space is requested, the color space shall fall back to ``"srgb"``.
* Implementations should not limit the set of exposed color spaces based on the capabilities of the display. The color space that best represents the capabilities of the canvas' current display may be determined using the [color gamut media queries](https://www.w3.org/TR/mediaqueries-5/#color-gamut) functionality found in the 
[CSS Media Queries Level 5](https://www.w3.org/TR/mediaqueries-5/) specification.

#### The ``storageFormat`` canvas creation attribute

The ``storageFormat`` attribute specifies the format for storing individual pixel color channel values, as well as the color encoding function to be used for non-alpha channels, if any.
* Support for ``"unorm8"`` is mandatory. All other formats are optional.
* When an unsupported format is requested, the format shall fall back to ``"unorm8"``.

#### The ``highDynamicRange`` canvas creation attribute

The ``highDynamicRange`` attribute specifies whether or not this canvas can display luminance greater than that of the color ``(1,1,1)``.
* It is possible to represent colors of arbitrarily high luminance using floating-point storage formats.
* Unless ``highDynamicRange`` is set to ``true``, colors must have their luminance clamped to the standard dynamic range.
* The motivation for this behavior is that enabling the display of high dynamic range content can come at a significant performance and power cost which should not be imposed unless explicitly requested.

#### Interpreting color values outside of the domain of [0, 1]

Some storage formats allow specifying color values that are outside of the domain of [0, 1].
For all color spaces, the transfer function is extended to the domain of all real numbers as follows.
* For color values greater than 1, the transfer functions are extended by not clamping the function at 1.
* For color values less than 0, all color space transfer functions are extended by point symmetry around 0.
In symbols, this means that for a transfer function of ``trFn`` the extended transfer function will be ``exTrFn(val) = sign(val) * trFn(abs(val))``.

For example, the extension of the sRGB transfer function defined in the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification to the full domain of real values would be as follows:
<pre>
function sRGBToLinear(val) {
if (val < -0.04045)
  return -Math.pow((-val + 0.055) / 1.055, 2.4);
else if (val <= 0.04045)
  return val / 12.92;
else
  return Math.pow((val + 0.055) / 1.055, 2.4);
</pre>

#### 2D canvas behavior

* All input colors (e.g, ``fillStyle`` and ``strokeStyle``) follow the same interpretation as CSS color literals, regardless of the canvas color space.
* Images drawn to a canvas are converted from their native color space to the backing storage color space of the canvas.
  * Images with no color profile, when drawn to the canvas, are assumed to be in the sRGB color space.
* All blending operations are performed in the canvas' backing storage color space.
  * The midpoint between the backbuffer's colors ``(1,0,0)`` and ``(0,1,0)`` will be ``(0.5,0.5,0)``, even though this is neither the physical nor perceptual midpoint between the two colors.
  * This means that the same content, rendered with two different canvas backing stores, may be slightly different.
* The functions ``toDataURL`` and ``toBlob`` should produce resources that best match the fidelity of the underlying canvas.
  * Subject to the limitations of the implementation and the requested format.

#### WebGL behavior

* Values stored in WebGL backbuffers are in the canvas' color space.
* Values written by ``gl_FragColor`` use the primaries of the canvas' color space.
* The color encoding function is defined as follows:
* For formats of ``"unorm8"``, ``"unorm16"``, and ``"float16"`` the color encoding function is:
<pre>
function encode(val) { return val; }
</pre>
* For the format ``"unorm8-srgb"``, the color encoding function is:
<pre>
function encode(val) {
  if (val < 0) return 0.0;
  else if (val < 0.0031308) return 12.92 * val;
  else if (val < 1) return (1.055 * Math.pow(val, 0.41666) - 0.055);
  else return 1.0;
}
</pre>
* For all formats, the values stored in the framebuffer are in the canvas' color space.

#### Compositing the canvas element

Canvas contents are composited in accordance with the canvas element's style (e.g. CSS compositing and blending rules). The necessary compositing operations must be performed in an intermediate colorspace, the compositing space, that is implementation specific. The compositing space must have sufficient precision and a sufficiently wide gamut to guarantee no undue loss of precision or gamut clipping in bringing the canvas's contents to the display.

The chromiumance of color values outside of [0, 1] is not to be clamped, and extended values may be used to display colors outside of the gamut defined by the canvas' color space's primaries.
This is in contrast with luminance, which may be clamped based on the value of the ``highDynamicRange`` attribute.

#### ImageBitmap

ImageBitmap objects (unless created with ``colorSpaceConversion="none"``) should keep track of their internal color space, and should store their contents at highest fidelity possible, subject to implementation limitations.

#### ImageData

Add the following types to be used by `ImageData`.
<pre>
enum ImageDataStorageFormat {
  "unorm8", // default
  "unorm16",
  "float32",
};
dictionary ImageDataSettings {
  CanvasColorSpaceEnum colorSpace = "srgb";
  ImageDataStorageFormat storageFormat = "unorm8";
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

The type of the ``data`` attribute is determined by the ``storageFormat`` parameter according to the following table.

| ``storageFormat`` Value | ``data`` Type |
|-|-|
| ``"unorm8"`` | ``Uint8ClampedArray`` |
| ``"unorm16"`` | ``Uint16Array`` |
| ``"float32"`` | ``Float32Array`` |


The constructor that takes both an `ImageDataArray` and an `ImageDataSettings` will throw an exception if the type of the `ImageDataArray` is incompatible with the type specified in `ImageDataSettings` (e.g, `ImageDataArray` is a `Float32Array`, but `ImageDataSettings` specifies ``storageFormat="unorm8"``).

When an ``ImageData`` is used in a canvas (e.g, in ``putImageData``), the data is converted from the ``ImageData``'s color space to the color space of the canvas.

To the ``CanvasRenderingContext2D`` interface, add
<pre>
partial interface CanvasRenderingContext2D {
    ImageData createImageData(long sw, long sh, optional ImageDataSettings settings);
    ImageData getImageData(long sx, long sy, long sw, long sh, optional ImageDataSettings settings);
}
</pre>

The changes to this interface are the addion of the optional ``ImageDataSettings`` argument. If this argument is unspecified, then the default values of ``storageFormat="unorm8"`` and ``colorSpace="srgb"`` will be used (these defaults match previous behavior).

The ``getImageData`` method is responsible for converting the data from the canvas' internal format to the format requested in the ``ImageDataSettings``.

### Examples

#### Selecting the best color space match for the user agent's display device
<pre>
var colorSpace = window.matchMedia("(color-gamut: rec2020)").matches ? "rec-2020" :
                (window.matchMedia("(color-gamut: p3)").matches ? "display-p3" : "srgb");
</pre>

### Limitations
* toDataURL and toBlob may be lossy, depending on the file format, when used on a canvas that has a storage type other than ``"unorm8"``. Possible future improvements could solve or mitigate this issue by adding more file formats or adding options to specify the resource color space.

### Adoption
Lack of color management and color interoperability is a longstanding complaint about the canvas API.
Authors of games and imaging apps are expected to be enthusiastic adopters.

## Unresolved Issues

* Should we support custom color spaces based on ICC profiles? Would offer ultimate flexibility. Would be hard to make implementations as efficient as built-in color spaces, in particular for implement linearPixelMath for profiles that have arbitrary transfer curves.

* Through what mechanism should HDR metadata be specified? To what extent should tonemapping be specified (e.g, should it be specified that tonemapping not alter SDR values).

* Should there be API-level support for mixing chromaticities and transfer functions, including use of no-op transfer functions?

* Should it be "rgba8" and "rgba16f" instead of "unorm8" and "float16"?

* Should context creation throw on an unrecognized, non-undefined creation attribute?

* The [Media Query APIs](https://www.w3.org/TR/mediaqueries-4/) use the names "p3" and "rec2020", while the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4/#predefined) uses the names "display-p3" and "rec-2020". This divergence could be confusing.

## Proposal History

This propsal was originally incubated in the WHATWG github issue tracker and incorporates feedback that was provided in the following thread: https://github.com/whatwg/html/issues/299

This proposal was further discussed in the Khronos WebGL working group, with the participation of engineers from Apple, Google, Microsoft, Mozilla, Nvidia, and others.

From 2016 to 2017, it was discussed on the [W3C WICG Discourse thread](https://discourse.wicg.io/t/canvas-color-spaces-wide-gamut-and-high-bit-depth-rendering/1505).

The current venue for discussing this proposal is in issues and pull requests to the [WICG canvas-color-space GitHub repo](https://github.com/WICG/canvas-color-space).
