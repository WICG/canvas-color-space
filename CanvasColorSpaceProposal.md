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

### CanvasColorSpace

IDL Additions:
<pre>
// Feature enums:
enum CanvasColorSpaceEnum {
  "srgb", // default
  "display-p3",
  "rec-2020",
};

// Feature detection:
interface CanvasColorSpace {
  const CanvasColorSpaceEnum srgb = "srgb";
  const CanvasColorSpaceEnum displayP3 = "display-p3";
  const CanvasColorSpaceEnum rec2020 = "rec-2020";
};
</pre>

Color spaces match their respective counterparts as defined in the [CSS colorspaces](https://www.w3.org/TR/css-color-4/#predefined). Support for color spaces should not be limited based on the capabilities of the user's display. There exist [Media Query APIs](https://www.w3.org/TR/mediaqueries-4/) for that purpose.

When displaying a canvas element, the browser must ensure that the colors displayed on the user's screen match the colors specified in the canvas element as closely as is possible on the platform. The color space conversions responsible for this happen in compositing, and should have no script-visible side-effects.

Several APIs will allow the user to specify floating-point colors outside of the usual [0, 1] interval. These values will be clamped to the range [0, 1] when being displayed.

TODO: Add 'extended-srgb' and 'extended-srgb-linear' color spaces which will not be subject to this constraint.

TODO: Add a mechanism for indicating that extended color spaces are to be used for HDR rendering (this should probably not be the default, because HDR rendering has substantial power costs).

### HTMLCanvasElement API Changes

The behavior of the HTMLCanvasElement functions toDataURL and toBlob methods are to produce encodings that match the canvas element's color space and pixel depth as closely as possible, subject to the limitations of the encoding format. It may be appropriate to add additional parameters to the relevant APIs to allow for more user control in this area.

### CanvasRenderingContext2D

IDL Additions:
<pre>
enum CanvasColorEncoding {
  "uint8", // default
  "float16",
};

partial dictionary CanvasRenderingContext2DSettings {
  CanvasColorSpace colorSpace = "srgb";
  CanvasColorEncoding colorEncoding = "uint8";
};

partial interface CanvasRenderingContext2D {
  CanvasRenderingContext2DSettings getContextAttributes();
  ImageData getImageData(long sx, long sy, long sw, long sh, optional ImageDataSettings imageDataSettings);
  ImageData createImageData(long sw, long sh, optional ImageDataSettings imageDataSettings);
};
</pre>

All input colors (e.g, fillStyle or strokeStyle, and gradient stops) follow the same interpretation as CSS color literals, regardless of canvas color space. Images that do not specify a color space, when drawn to the canvas, are treated as though they specified the sRGB color space.

Note that because there exists no direct access to the canvas backbuffer, it is not required that the colorEncoding be truly respected -- from the user's point of view, there will be no way to know if the true backing has higher precision than what was requested.

### WebGL 2 API Changes

IDL Additions:
<pre>
enum WebGLPixelFormat {
  "unorm8",      // default
  "unorm8-srgb", // gl_FragColor value of 0.5 encoded as 0xBC
  "float16",
};

partial dictionary WebGLContextAttributes {
  CanvasColorSpace colorSpace = "srgb";
  WebGLPixelFormat pixelFormat = "unorm8";
};
</pre>

Values written to WebGL backbuffers (e.g, values written to gl_FragColor or the clear color) are in the canvas's color space. In the case of "unorm8-srgb", gl_FragColors are in a color space with the primaries indicated by the specified color space, but with a linear transfer function. This means than a gl_FragColor value of 0.5 represents half of the intensity of a gl_FragColor of 1.0, and so it is encoded as 0xBC, not 0x80. All blending operations are performed in the linear space. The pixel format "unorm8-srgb" may only be used with color spaces that have the same transfer function as sRGB (TODO: determine if rec-2020 should).

If a WebGLPixelFormat other than "unorm8" is specified, then the WebGLContextAttribute alpha must be "true". This reflects the decreasing presence of three-component pixel formats in modern graphics APIs.

TODO: Should we add values for UNPACK_COLORSPACE_CONVERSION_WEBGL to convert to all available CanvasColorSpace values?

### ImageData API Changes

IDL Additions:
<pre>
typedef (Uint8ClampedArray or Uint16Array or Float32Array) ImageDataArray;

enum ImageDataStorageFormat {
  "uint8", // default
  "uint16",
  "float32",
};

dictionary ImageDataAttributes {
  CanvasColorSpace colorSpace = "srgb";
  ImageDataStorageFormat storageFormat = "uint8";
};

partial interface ImageData {
  ImageDataAttributes getAttributes();
  ImageDataArray data;
};
</pre>

An ImageData's data attribute will be of the type indicated by the its storageFormat.

Note that the storage formats are different from the pixel formats for 2D Canvas and WebGL pixel formats. This reflects the fact that there does not exist a Float16Array, although Float16 is a useful format for graphics buffers.

### ImageBitmap

TODO: Should we update createImageBitmap's colorSpaceConversion argument to take any CanvasColorSpace value. Should the behavior of "default" be solidified in its meaning?

### Adoption
Lack of color management and color interoperability is a longstanding complaint about the canvas API.
Authors of games and imaging apps are expected to be enthusiastic adopters.

## Issues

Issues:

* How should feature detection work (it should be separate for WebGL and Canvas, and potentially for CSS as well)?

* For ImageData, is it appropriate to be able to query the ImageDataStorageFormat, when that can be inferred from the type of the data member? This would make sense if we supported both uint8 and unorm8.

* Should we support custom color spaces based on ICC profiles?

* [CSS rec-2020 color space](https://www.w3.org/TR/css-color-4/#valdef-color-rec-2020) gives a different behavior than Wikipedia. Determine if this is intended (gamma 2.4 transfer function versus sRGB transfer function).

* The [Media Query APIs](https://www.w3.org/TR/mediaqueries-4/) use the names "p3" and "rec2020" instead of "display-p3" and "rec-2020". This divergence could be confusing.

* Is the naming for CanvasColorEncoding as "uint8" appropriate, or would "unorm8" be better? It may be better to avoid the term "unorm8" outside of lower level graphics APIs.

* The naming for WebGLPixelFormat may want to more closely mirror existing graphics APIs:
  * Example APIs:
    * [WebGL 2](https://developer.mozilla.org/en-US/docs/Web/API/WebGL2RenderingContext/texStorage2D): RGBA8, SRGB8_APLHA8(?), RGBA16F
    * [MTLPixelFormat](https://developer.apple.com/documentation/metal/mtlpixelformat?language=objc): MTLPixelFormatRGBA8Unorm,
 MTLPixelFormatRGBA8Unorm_sRGB, MTLPixelFormatRGBA16Float
    * [VKFormat](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkFormat.html): VK_FORMAT_R8G8B8A8_UNORM, VK_FORMAT_R8G8B8A8_SRGB, VK_FORMAT_R16G16B16A16_SFLOAT
    * [DXGIFormat](https://docs.microsoft.com/en-us/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format): DXGI_FORMAT_R8G8B8A8_UNORM, DXGI_FORMAT_R8G8B8A8_UNORM_SRGB, DXGI_FORMAT_R16G16B16A16_FLOAT
    * [OpenGL ES 3](https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glTexStorage2D.xhtml): GL_RGBA8, GL_RGBA8_SNORM, GL_RGBA16F
  * If we follow this scheme, then a "default" value will need to specified (because it will need to be harmonized with the alpha value).

* Should float16 allow alpha values outside [0,1] range at any stage of the pipeline? What would they mean?

* Should there be API-level support for mixing chromaticities and transfer functions, including use of no-op transfer functions?

## Proposal History

This propsal was originally incubated in the WHATWG github issue tracker and incorporates feedback that was provided in the following thread: https://github.com/whatwg/html/issues/299

This proposal was further discussed in the Khronos WebGL working group, with the participation of engineers from Apple, Google, Microsoft, Mozilla, Nvidia, and others.

From 2016 to 2017, it was discussed on the [W3C WICG Discourse thread](https://discourse.wicg.io/t/canvas-color-spaces-wide-gamut-and-high-bit-depth-rendering/1505).

The current venue for discussing this proposal is in issues and pull requests to the [WICG canvas-color-space GitHub repo](https://github.com/WICG/canvas-color-space).
