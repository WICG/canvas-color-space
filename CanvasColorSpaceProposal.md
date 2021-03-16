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
  * For 2D Canvas and ImageData, this topic will be addressed by UWCG and HDR proposal, because such color spaces require more than the default of 8 bits per pixel.
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

enum CanvasColorSpaceEnum {
  "srgb", // default
  "display-p3",
};

// Feature detection:

interface CanvasColorSpace {
  const CanvasColorSpaceEnum srgb = "srgb";
  const CanvasColorSpaceEnum displayP3 = "display-p3";
};
</pre>

Example:
<pre>
canvas.getContext('2d', { colorSpace: "display-p3"} );,
</pre>

The ``colorSpace`` attribute specifies the color space for the backing storage of the canvas.
* Color spaces match their respective counterparts as defined in the [predefined color spaces](https://www.w3.org/TR/css-color-4/#predefined) section of the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification.
* Implementations should not limit the set of exposed color spaces based on the capabilities of the display. The color space that best represents the capabilities of the canvas' current display may be determined using the [color gamut media queries](https://www.w3.org/TR/mediaqueries-5/#color-gamut) functionality found in the 
[CSS Media Queries Level 5](https://www.w3.org/TR/mediaqueries-5/) specification.

### 2D Canvas

IDL:
<pre>
// Feature activation:

partial dictionary CanvasRenderingContext2DSettings {
  CanvasColorSpaceEnum colorSpace = "srgb";
};
</pre>

* All input colors (e.g, ``fillStyle`` and ``strokeStyle``) follow the same interpretation as CSS color literals, regardless of the canvas color space.
* Images drawn to a canvas are converted from their native color space to the backing storage color space of the canvas.
  * Images with no color profile, when drawn to the canvas, are assumed to be in the sRGB color space.
* All blending operations are performed in the canvas' backing storage color space.
  * The midpoint between the backbuffer's colors ``(1,0,0)`` and ``(0,1,0)`` will be ``(0.5,0.5,0)``, even though this is neither the physical nor perceptual midpoint between the two colors.
  * This means that the same content, rendered with two different canvas backing stores, may be slightly different.
* When an unsupported color space is requested, the color space shall fall back to ``"srgb"``.

#### WebGL

IDL:
<pre>
// Feature activation:

partial dictionary WebGLContextAttributes {
  CanvasColorSpaceEnum colorSpace = "srgb";
};
</pre>

Values stored in WebGL's default back buffer are in the canvas' color space.

Pixel values accessed directly (e.g, through ReadPixels) are in the canvas' color space.

The values written to the default backbuffer through the graphics pipeline are also in the canvas' color space.

Note that if sRGB framebuffer color encoding is enabled for the default backbuffer, then the value written is not the same as the the value assigned to the fragment shader's color output variable.
In that situation, the linear-to-sRGB transformation function is applied to the assigned value before it is written.
Consequently, the value assigned to the fragment shader's color output variable can be interpreted as being in a linear version of the canvas' color space.

### WebGPU

WebGPU's context configuration is specified dynamically using the `GPUCanvasContext` method `configureSwapChain`, which takes a `GPUSwapChainDescriptor` argument.
Add an additional entry to `GPUSwapChainDescriptor` for the color space of the swap chain.

IDL:
<pre>
partial dictionary GPUSwapChainDescriptor {
  CanvasColorSpaceEnum colorSpace = "srgb";
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

ImageBitmap objects (unless created with ``colorSpaceConversion="none"``) should keep track of their internal color space, and should store their contents at highest fidelity possible, subject to implementation limitations.

### ImageData

Add the following types to be used by `ImageData`.

IDL:
<pre>
dictionary ImageDataSettings {
  CanvasColorSpaceEnum colorSpace = "srgb";
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

This example selects a wide color gamut canvas only if the underlying display has a P3 gamut or larger.

```html
  // Note that the gamut is named p3, but the color space is 'display-p3'.
  var matchingColorSpace = window.matchMedia(
      '(color-gamut: p3)').matches ? 'display-p3' : 'srgb';
  var canvas = document.getElementById('Canvas');
  var context = canvas.getContext('2d', {colorSpace:matchingColorSpace});
```

### Drawing wide color gamut content to a 2D Canvas

This example shows drawing a wide color gamut image to an sRGB and a Display-P3 canvas, and discusses the different expected results.

```html
  // Let |myWCGImage| be a loaded wide color gamut Image.
  var myWCGImage;

  // Let |defaultCanvas| be a default (sRGB) canvas. This code will draw the
  // specified image, but will clip it to the sRGB color gamut.
  var defaultCanvas = document.getElementById('DefaultCanvas');
  var defaultCtx = defaultCanvas.getContext('2d');
  defaultCtx.drawImage(myWCGImage, 0, 0, image.width, image.height)

  // Let |wcgCanvas| be a Display-P3 canvas. This code will draw the specified
  // image, which will not be clipped to the sRGB color gamut (it will be
  // restricted only to the P3 color gamut).
  var wcgCanvas = document.getElementById('DefaultCanvas');
  var wcgCtx = wcgCanvas.getContext('2d', {colorSpace:'display-p3'});
  wcgCtx.drawImage(myWCGImage, 0, 0, image.width, image.height)
```

### Drawing and retrieving content using ``ImageData``

This example shows use of the new ``ImageData`` color management APIs.

```html
  var canvas = document.getElementById('Canvas');
  var context = defaultCanvas.getContext('2d', {colorSpace:'display-p3'});

  // Creating a ImageData without specifying ImageDataSettings will create an
  // sRGB ImageData. This will draw the color sRGB-red to the canvas.
  var srgbImageData = new ImageData(1, 1);
  srgbImageData.data[0] = srgbImageData.data[3] = 255;
  srgbImageData.data[1] = srgbImageData.data[2] = 0;
  context.putImageData(srgbImageData, 0, 0);

  // This will draw P3-red to the canvas.
  var p3ImageData = new ImageData(1, 1, {colorSpace:'display-p3'});
  p3ImageData.data[0] = p3ImageData.data[3] = 255;
  p3ImageData.data[1] = p3ImageData.data[2] = 0;
  context.putImageData(p3ImageData, 0, 0);

  // The color space of an ImageData can be retrieved through its
  // ImageDataSettings.
  console.log(srgbImageData.getSettings().colorSpace);
  console.log(p3ImageData.getSettings().colorSpace);

  // Reading back an ImageData without specifying a color space will retrieve
  // pixel values converted to sRGB. This will clamp color values to sRGB, and
  // so |readSrgbImageData.data| will be [255,0,0,255, 255,0,0,255];
  var readSrgbImageData = context.getImageData(0, 0, 2, 1);
  console.log(readSrgbImageData.data);

  // Reading back an ImageData specifying a color space will retrieve the pixel
  // values converted to that color space. |readP3ImageData.data| will be
  // [234,51,35,255, 255,0,0,255].
  var readP3ImageData = context.getImageData(0, 0, 2, 1,
                                             {colorSpace:'display-p3'});
  console.log(readP3ImageData.data);
```

### Drawing WebGL wide color gamut content

This example shows a WebGL application that clears the screen to Display P3 red.

```html
  var canvas = document.getElementById('Canvas');
  var gl = defaultCanvas.getContext('webgl2');
  gl.colorSpace = 'display-p3';
  gl.clearColor(1.0, 0.0, 0.0, 1.0);
  gl.clear(gl.COLOR_BUFFER_BIT);
```

### Uploading an image as Display-P3 pixels

This example shows uploading an image and converting its color profile.
Note that this example assumes the conversion behavior of ``BROWSER_DEFAULT_WEBGL``.
Also note that in this example the color conversion happens synchronously at the moment that ``texImage2D`` is called, which may not be ideal.

```html
  // Let |myImage| be an image that has some (unknown) color profile.
  var myImage;
  var canvas = document.getElementById('Canvas');
  var gl = defaultCanvas.getContext('webgl2');

  // Let texInP3 be the image converted to Display-P3.
  var texInP3 = gl.createTexture();
  gl.bindTexture(gl.TEXTURE_2D, texInP3);
  gl.pixelStorei(gl.UNPACK_COLORSPACE_CONVERSION_WEBGL,
                 gl.BROWSER_DEFAULT_WEBGL);
  gl.colorSpace = 'display-p3';
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.SRGB8_ALPHA8, image.width, image.height, 0,
                gl.RGBA, gl.UNSIGNED_BYTE, image);
```

In this example, the ``ImageBitmap`` API is used to allow the color conversion of the image to be performed asynchronously at decode time.

```html
  // Let |myImageUrl| be the URL for an image that has some (unknown) color
  // profile.
  var myImageUrl;
  var canvas = document.getElementById('Canvas');
  var gl = defaultCanvas.getContext('webgl2');

  fetch(myImageUrl).then(function(response) {
    return response.blob();
  }).then(function(blob) {
    return createImageBitmap(blob, {colorSpace:'display-p3'});
  }).then(function(bitmap) {
    const texInP3 = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texInP3);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.SRGB8_ALPHA8, bitmap.width,
                  bitmap.height, 0, gl.RGBA, gl.UNSIGNED_BYTE, bitmap);
    
  });
```

## Adoption
Lack of color management and color interoperability is a longstanding complaint about the canvas API.
Authors of games and imaging apps are expected to be enthusiastic adopters.

## Unresolved Issues

* Should we support custom color spaces based on ICC profiles? Would offer ultimate flexibility. Would be hard to make implementations as efficient as built-in color spaces, in particular for implement linearPixelMath for profiles that have arbitrary transfer curves.

* Through what mechanism should HDR metadata be specified? To what extent should tonemapping be specified (e.g, should it be specified that tonemapping not alter SDR values).

* Should there be API-level support for mixing chromaticities and transfer functions, including use of no-op transfer functions?

* Should context creation throw on an unrecognized, non-undefined creation attribute?

* The [Media Query APIs](https://www.w3.org/TR/mediaqueries-4/) use the name "p3", while the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4/#predefined) uses the name "display-p3". This divergence could be confusing.

## Proposal History

This propsal was originally incubated in the WHATWG github issue tracker and incorporates feedback that was provided in the following thread: https://github.com/whatwg/html/issues/299

This proposal was further discussed in the Khronos WebGL working group, with the participation of engineers from Apple, Google, Microsoft, Mozilla, Nvidia, and others.

From 2016 to 2017, it was discussed on the [W3C WICG Discourse thread](https://discourse.wicg.io/t/canvas-color-spaces-wide-gamut-and-high-bit-depth-rendering/1505).

The current venue for discussing this proposal is in issues and pull requests to the [WICG canvas-color-space GitHub repo](https://github.com/WICG/canvas-color-space).
