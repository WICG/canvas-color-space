# Color managing canvas contents

## Background

### Current Limitations and Goals

Wide color gamut web content, in the form of images and videos, is ubiquitous today.
Wide color gamut display devices are also ubiquitous.

The color space of content displayed in an ``HTMLCanvasElement`` via 2D Canvas, WebGL, or WebGPU, is not explicitly defined.
The de facto behavior of most browers is to limit this content to the sRGB color space, with 8 bits per color.
This presents a significant limitation compared to the capabilities of display devices and the demands of content creators.

The goals of this proposal are to:
* Ensure that canvas content's color is well-defined, to minimize differences in appearance across browsers and display devices
* Allow 2D Canvas, WebGL, and WebGPU to take advantage of a wider color gamut.

Non-goals of this proposal are:
* Ultra-wide color gamut color space support (e.g, Rec2020) and high dynamic range support.
  * This topic is deferred to a subsequent proposal.
* High bit depth 2D Canvas and ImageData support.
  * For 2D Canvas and ImageData, this topic will be addressed by [UWCG and HDR proposal](#additional-color-spaces-high-bit-depth-and-high-dynamic-range),
  because such color spaces require more than the default of 8 bits per pixel.
  * For WebGL, this topic is addressed in a separate pixel format proposal.
  * For WebGPU, high bit depth pixel formats may be specified in ``GPUSwapChainDescriptor``.

### Use Cases

Examples of web applications that will benefit from a well-defined color space and wide color gamut support include:
* Photo viewing and manipulation applications.
* Visual design applications.
* Commerce applications, for product presentation.
* Gaming applications.

### Requests for this Feature

* <cite>[https://github.com/whatwg/html/issues/299]</cite> <blockquote><p>Allow 2dcontexts to use deeper color buffers</p></blockquote>
* <cite>[https://bugs.chromium.org/p/chromium/issues/detail?id=425935]</cite> <blockquote><p>Wrong color profile with 2D canvas</p></blockquote>
* Engineers from the Google Photos, Maps and Sheets teams have expressed a desire for canvases to be color managed. Particularly for the use case of resizing an image, using a canvas, prior to uploading it to the server, to save bandwidth. The problem is that the images retrieved from a canvas are in an undefined color space and no color space information is encoded by toDataURL or toBlob.

### Related Specifications

* The [CSS Media Queries Level 5](https://www.w3.org/TR/mediaqueries-5/#color-gamut) specification defines the ``color-gamut`` media query to determine the capabilities of the current display device.
* The [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification defines several color spaces that will be used in this specification. At time of writing this specification is partially implemented on some browsers.

## Proposed Solution

This proposal aims to accomplish the above stated goals by:
* Introducing sRGB and Display-P3 as the set of color spaces that are available for use by canvas APIs.
* Adding an API whereby 2D Canvas, WebGL, WebGPU, and ImageData may specify one of those color spaces.

This proposal will also clarify:
* The default color space for 2D Canvas, WebGL, WebGPU, and ImageData.
* The compositing behavior for 2D Canvas.
* The behavior when exporting elements using ``toDataURL`` and ``toBlob``.

The remainder of this section discusses this functionality and behavior in detail.

### Color Spaces

IDL:
<pre>
// Feature enums:

enum PredefinedColorSpaceEnum {
  "srgb", // default
  "display-p3",
};

// Feature detection:

interface PredefinedColorSpace {
  const PredefinedColorSpaceEnum srgb = "srgb";
  const PredefinedColorSpaceEnum displayP3 = "display-p3";
};
</pre>

Example:
<pre>
canvas.getContext('2d', { colorSpace: "display-p3"} );,
</pre>

The ``colorSpace`` attribute specifies the color space for the backing storage of the canvas.
* Color spaces match their respective counterparts as defined in the [predefined color spaces](https://www.w3.org/TR/css-color-4/#predefined) section of the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification.
* Implementations should not limit the set of exposed color spaces based on the capabilities of the display.
* The color space that best represents the capabilities of the canvas' current display may be determined using the [color gamut media queries](https://www.w3.org/TR/mediaqueries-5/#color-gamut) functionality found in the 
[CSS Media Queries Level 5](https://www.w3.org/TR/mediaqueries-5/) specification.

### 2D Canvas

IDL:
<pre>
// Feature activation:

partial dictionary CanvasRenderingContext2DSettings {
  PredefinedColorSpaceEnum colorSpace = "srgb";
};
</pre>

2D Canvases are color managed.
* All inputs that are drawn to a canvas have specific color space (there are no inputs to a canvas that do not have a well-defined color space).
  * Input colors (e.g, ``fillStyle`` and ``strokeStyle``) follow the same interpretation as CSS color literals, regardless of the canvas color space.
  * Images with no color profile, when drawn to the canvas, are assumed to be in the sRGB color space.
* All inputs that are drawn to a canvas are converted from their color space to the canvas' color space before being drawn to the canvas.
* All blending and gradient interpolation operations are performed in the canvas' color space.
  * The midpoint between the backbuffer's colors ``(1,0,0)`` and ``(0,1,0)`` will be ``(0.5,0.5,0)``, even though this is neither the physical nor perceptual midpoint between the two colors.
  * This means that the same content, rendered with two different canvas backing stores, may be slightly different.

When an unsupported color space is requested, the color space will fall back to ``"srgb"``.

### WebGL

IDL:
<pre>
partial interface WebGLRenderingContextBase {
  attribute PredefinedColorSpaceEnum colorSpace = "srgb";
};
</pre>

Values stored in WebGL's default back buffer are to be interpreted as being in the color space specified by the ``colorSpace`` attribute.
Changing the ``colorSpace`` attribute is not destructive to the contents of the default back buffer.

The ``UNPACK_COLORSPACE_CONVERSION_WEBGL`` pixel storage parameter indicates the color space conversion that will be applied to ``HTMLImageElement`` sources passed to ``texImage2D`` and ``texSubImage2D``.
In implementations in which a ``UNPACK_COLORSPACE_CONVERSION_WEBGL`` of ``BROWSER_DEFAULT_WEBGL`` causes a conversion to sRGB color space, it is recommended that this behavior be changed to be a conversion to the color space indicated by ``colorSpace``.

Note that, while convenient, this behavior is likely less efficient than specifying color conversion in ``ImageBitmapOptions``, where the color conversion may be done asynchronously and simultaneously with image decode.

### WebGPU

WebGPU's context configuration is specified dynamically using the `GPUCanvasContext` method `configureSwapChain`, which takes a `GPUSwapChainDescriptor` argument.
Add an additional entry to `GPUSwapChainDescriptor` for the color space of the swap chain.

IDL:
<pre>
partial dictionary GPUSwapChainDescriptor {
  PredefinedColorSpaceEnum colorSpace = "srgb";
};
</pre>

All values read from and written to the swap chain are in the canvas' color space.

As described in the WebGL section above, if the ``GPUSwapChainDescriptor``'s ``format`` is ``"rgba8unorm-srgb"`` or ``"bgra8unorm-srgb"``, then the value assigned to the color output of the fragment shader can be interpreted as being in a linear version of the canvas' color space, because it will have the linear-to-sRGB transformation applied to it before it is written to the swap chain.

### HTMLCanvasElement

The functions ``toDataURL`` and ``toBlob`` should produce resources that best match the fidelity of the underlying canvas, subject to the limitations of the implementation and the requested format.

#### Compositing of the HTMLCanvasElement
Canvas contents are composited in accordance with the canvas element's style (e.g. CSS compositing and blending rules). The necessary compositing operations must be performed in an intermediate colorspace, the compositing space, that is implementation specific. The compositing space must have sufficient precision and a sufficiently wide gamut to guarantee no undue loss of precision or gamut clipping in bringing the canvas's contents to the display.

The chromiumance of color values outside of [0, 1] is not to be clamped, and extended values may be used to display colors outside of the gamut defined by the canvas' color space's primaries.
This is in contrast with luminance, which is to be clamped to the maximum standard dynamic range luminance, unless high dynamic range is explicitly enabled for the canvas element.

### ImageBitmap

IDL:
<pre>
partial dictionary ImageBitmapOptions {
  CanvasColorSpaceEnum colorSpace = "srgb";
}
</pre>

When creating an ``ImageBitmap``, if the ``colorSpaceConversion`` entry of the specified ``ImageBitmapOptions`` is not ``"none"``, then the internal color space for the resulting ``ImageBitmap`` will be the color space that is specified by the ``colorSpace`` entry of the ``ImageBitmapOptions``.
If that ``ImageBitmap`` is then used as input to populate a WebGL or WebGPU texture, then the pixel values written to the texture will represent the ``ImageBitmap``'s source contents in the specified color space.

### ImageData

Add the following types to be used by `ImageData`.

IDL:
<pre>
dictionary ImageDataSettings {
  PredefinedColorSpaceEnum colorSpace = "srgb";
};
</pre>

Update the `ImageData` interface to the include the following.

IDL:
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

When an ``ImageData`` is used in a canvas (e.g, in ``putImageData``), the data is converted from the ``ImageData``'s color space to the color space of the canvas.

To the ``CanvasRenderingContext2D`` interface, add
<pre>
partial interface CanvasRenderingContext2D {
    ImageData createImageData(long sw, long sh, optional ImageDataSettings settings);
    ImageData getImageData(long sx, long sy, long sw, long sh, optional ImageDataSettings settings);
}
</pre>

The changes to this interface are the addion of the optional ``ImageDataSettings`` argument. If this argument is unspecified, then the default value of ``colorSpace="srgb"`` will be used (this default match previous behavior).

The ``getImageData`` method is responsible for converting the data from the canvas' internal format to the format requested in the ``ImageDataSettings``.

## Examples

### Selecting the best color space match for the user agent's display device
<pre>
var colorSpace = window.matchMedia("(color-gamut: p3)").matches ? "display-p3" : "srgb";
</pre>

## Resolved Issues

### Additional color spaces, high bit depth, and high dynamic range

Should we support color spaces besides `'srgb'` and `'display-p3'`?

In another proposal.
This proposal limits its scope to the capabilites of 8-bit buffers.
Additional color spaces such as ``'rec2020'`` or any HDR color spaces will need higher bit depth support.
We defer supporting color spaces that require more than 8 bit of precision to a separate proposal that also covers high bit depth and high dyanmic range.

_NOTE: The [Color-on-the-web Community Group](https://www.w3.org/community/colorweb/) is working on [adding support for HDR imagery to Canvas](https://github.com/w3c/ColorWeb-CG)._

### Custom color profile support

Should we support custom color spaces based on ICC profiles?

Not now.
Rendering to an arbitrary ICC profile adds a significant level of complexity to the implementation.
This proposal does not add this capability, but does not close the door to its addition in a future proposal.
The ``PredefinedColorSpaceEnum`` could be changed to a ``DOMString`` that could refer any CSS color profile, including CSS Color Module Level 4's [custom profiles](https://www.w3.org/TR/css-color-4/#at-profile).

### Separating chromaticities and transfer functions

Should there be API-level support for mixing chromaticities and transfer functions, including use of no-op transfer functions?

No.
The color space names are selected to coincide with the CSS Color Module Level 4 [predefined color space names](https://www.w3.org/TR/css-color-4/#predefined).
Any new syntax for specifying color spaces should be made in that context.

### Handling unrecognized color spaces

Should context creation throw on an unrecognized, non-undefined creation attribute?

No.
Consider the following example.
An application calls ``canvas.getContext('2d', {colorSpace:'myNonExistentColorSpace'})``.

* If run on a browser that does not support canvas color spaces the ``colorSpace`` key is ignored, and an sRGB canvas is created.
* If context creation were to throw on an unrecognized attribute, then this code would fail only on browsers that support canvas color spaces.

Succeeding on a browser that does not support the feature and failing on a browser that does support the feature is an undesirable behavior.

### Naming of color gamuts

The [Media Query APIs](https://www.w3.org/TR/mediaqueries-4/) use the name "p3", while the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4/#predefined) uses the name "display-p3". Could this divergence be confusing?

Yes and no.
Indeed it can be confusing that a gamut name and color space name are similar but not the same.
That said, there is a concept of the P3 primaries outside of the Display P3 color space (e.g, the DCI P3 color space), so it's not as though this is in error.

### Alternative schemes for specifying ``CanvasRenderingContext2D`` color space

This proposal adds a ``colorSpace`` argument to ``CanvasRenderingContext2DSettings``.
The color space is then an immutable property of the ``CanvasRenderingContext2D``.
Should it be put elsewhere, say, as a mutable attribute on ``CanvasRenderingContext2D``?

No, but reviewers of this proposal should consider the options.
The alternative would look as follows.

```html
  // Proposal mode
  context = canvas.getContext('2d', {colorSpace:'display-p3'});,

  // Alternative mode
  context = canvas.getContext('2d');,
  context.colorSpace = 'display-p3';
```

The benefit of the alternative mode is that it side-steps the aforementioned issue related to specifying invalid color spaces.
The disadvantage is that it breaks the color management mode of 2D canvases, and adds significant API and implementation complexity.

This behavior of specifying a color space at creation time also applies to ``ImageData``.
In the case of ``ImageData``, a color space must be specified at creation time (e.g, in ``getImageData``).

Note that WebGL puts the ``colorSpace`` on the ``WebGLRenderingContextBase`` interface.
Changing the WebGL color space just causes the pixels to be reinterpreted, but that is suitable because WebGL is not color managed, and raw pixel access is the norm.

In contrast, a 2D canvas is color managed and does not allow raw pixel access.
What should happen to the canvas if its color space changes?
There are three options:

* Destroy the context's contents. This is what happens when a ``HTMLCanvasElement`` is resized.
* Convert the existing pixels to the new color space.
* Reinterpret the existing pixels as though they are in the new color space. This is very much against the spirit of the 2D canvas being color managed, and likely leaks implementation-specific details to the web application.

The most reasonable of these options is to destroy the context's contents.

## Proposal History

This propsal was originally incubated in the WHATWG github issue tracker and incorporates feedback that was provided in the following thread: https://github.com/whatwg/html/issues/299

This proposal was further discussed in the Khronos WebGL working group, with the participation of engineers from Apple, Google, Microsoft, Mozilla, Nvidia, and others.

From 2016 to 2017, it was discussed on the [W3C WICG Discourse thread](https://discourse.wicg.io/t/canvas-color-spaces-wide-gamut-and-high-bit-depth-rendering/1505).

The current venue for discussing this proposal is in issues and pull requests to the [WICG canvas-color-space GitHub repo](https://github.com/WICG/canvas-color-space).
