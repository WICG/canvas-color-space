# Canvas 2D Color Management Explainer

This explainer summarizes color management features added to 2D canvases.

## Background and related specifications

The features described are those added in [this WhatWG PR](https://github.com/whatwg/javascript/pull/6562).

This work is derived from an initial more wide ranging proposal, developed by the W3C's ColorWeb CG, which may be found [here](https://github.com/WICG/canvas-color-space/blob/main/CanvasColorSpaceProposal.md).
This proposal was revised during WhatWG review.

The related [CSS Media Queries Level 5](https://www.w3.org/TR/mediaqueries-5/#color-gamut) specification defines the `color-gamut` media query to determine the capabilities of the current display device.

The related [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification defines several color spaces that may be used to define CSS colors.

## Summary of changes

Formalization of existing functionality:

* This formalizes the convention that content rendered by a `CanvasRenderingContext2D` is in the sRGB color space by default.
* This formalizes the convention that the pixels specified by an `ImageData` are in the sRGB color space by default.
* This formalizes the convention that `CanvasRenderingContext2D` is color managed, meaning that all inputs are converted to the context's color space when drawing.
* This formalizes the convention that content (images, in particular) that does not specify a color space should be interpreted as being in the sRGB color space.
* This formalizes that the `toDataURL` and `toBlob` methods of `HTMLCanvasElement` are to create representations that match the canvas's context's color space as accurately as possible.
* This formalizes that the `getImageData` and `createImageData` methods of `CanvasRenderingContext2D` are to create an `ImageData` object that is in the same color space as the context on which the method was called.

New functionality:

* This adds a parameter whereby a `CanvasRenderingContext2DSettings` can specify a color space other than sRGB.
* This adds a parameter whereby an `ImageData` can specify a color space.
* This adds color spaces of `"srgb"` (for sRGB) and `"display-p3"` (for Display P3) as options for the above parameters.

## New APIs

In this section we cover all new enums, interfaces, and methods added by this feature.

### `PredefinedColorSpace`

The color spaces in this proposal are available in the new `PredefinedColorSpace` enum.
The names in this enum are chosen to the names in the [predefined color spaces](https://www.w3.org/TR/css-color-4/#predefined) section of the [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4) specification.

```idl
enum PredefinedColorSpace {
  "srgb", // default
  "display-p3",
};
```

### `CanvasRenderingContext2DSettings`

To create a `CanvasRenderingContext2D` that does not use the default color space of sRGB, the color space may be specified in the `CanvasRenderingContext2DSettings` used to create the context.
This dictionary is already used to specify if the context's pixel format should include an alpha channel, among other features.
Similarly to the alpha channel, the color space of an existing `CanvasRenderingContext2D` may be queried using the [`getContextAttributes`](https://javascript.spec.whatwg.org/multipage/canvas.javascript#dom-context-2d-canvas-getcontextattributes) method, which will return a `CanvasRenderingContext2DSettings`.

```idl
partial dictionary CanvasRenderingContext2DSettings {
  PredefinedColorSpaceEnum colorSpace = "srgb";
};
```

### `ImageDataSettings`

To create an `ImageData` that does not use the default color space of sRGB, the color space may be specified in a new `ImageDataSettings` dictionary.
Note that this dictionary does not specify a default color space.

```idl
dictionary ImageDataSettings {
  PredefinedColorSpaceEnum colorSpace;
};
```

### `ImageData`

The `ImageData` interface's constructors are updated to optionally take this `ImageDataSettings` argument.
If this argument is not specified, or if it is specified but its `colorSpace` entry is not specified, then the `ImageData` will default to sRGB.

```idl
partial interface ImageData {
  constructor(unsigned long sw,
              unsigned long sh,
              optional ImageDataSettings settings = {});
  constructor(Uint8ClampedArray data,
              unsigned long sw,
              optional unsigned long sh,
              optional ImageDataSettings settings = {});
};
```

The `ImageData` interface is updated to include a `PredefinedColorSpace` attribute.

```idl
partial interface ImageData {
  readonly attribute PredefinedColorSpace colorSpace;
};
```

### `CanvasRenderingContext2D`

The `CanvasRenderingContext2D` interface is updated to include an optional `ImageDataSettings` argument to the `createImageData` and `getImageData` functions.
If this argument is not specified, or if it is specified but its `colorSpace` entry is not specified, then the `ImageData`'s color space will be set to that context's color space.
The `getImageData` method will convert pixel values from the context's color space to the result's color space, if needed.

```idl
partial interface CanvasRenderingContext2D {
  ImageData createImageData(long sw,
                            long sh,
                            optional ImageDataSettings settings = {});
  ImageData getImageData(long sx,
                         long sy,
                         long sw,
                         long sh,
                         optional ImageDataSettings settings = {});
}
```

## Formalized and new behaviors

In this section we discuss new behaviors, and formalize behaviors that were previously undefined.

### Default color space for `CanvasRenderingContext2D`

The default color space for content rendered using `CanvasRenderingContext2D` has, by convention but not by specification, been sRGB.
This is now formalized by the fact that `CanvasRenderingContext2DSettings` specifies a default value of `"srgb"` for `colorSpace`.

### Default color space for `ImageData`

The default color space for `ImageData` has historically been sRGB (perhaps more accurately, its behaviors have been consistent with it being sRGB).
As mentioned earlier, this is now formalized by the [ImageData initialization algorithm](https://html.spec.whatwg.org/multipage/canvas.html#initialize-an-imagedata-object).

As mentioned earlier, if an `ImageDataSettings` does not specify `colorSpace` in a call to `getImageData` or `createImageData`, the resulting `ImageData` will [default to](https://html.spec.whatwg.org/multipage/canvas.html#dom-context-2d-createimagedata) the color space of the context.
This means that doing a "round trip" using `getImageData` and then `putImageData` will lose no fidelity.

### Color conversions

When drawing content into a `CanvasRenderingContext2D`, that content is converted from its source color space to the context's color space.
This is now [formalized](https://html.spec.whatwg.org/multipage/canvas.html#colour-spaces-and-colour-correction).
Historically, by convention but not by specification, (in WebKit and Chromium), this has meant converting to sRGB.

Content that does not specify a color space [is to be interpreted](https://drafts.csswg.org/css-color/#untagged) as being in the sRGB color space.
This language is now [re-emphasized](https://whatpr.org/html/6466/canvas.html#colour-spaces-and-colour-correction) in non-normative text.

The color space conversion algorithm for all conversions has historically been relative colorimetric intent.
This is now formalized.
Relative colorimetric intent is described in detail [here](https://drafts.csswg.org/css-color/#valdef-color-profile-rendering-intent-relative-colorimetric).
It is the simplest (or next-to-simplest) color conversion algorithm.
When converting from a wider gamut space to a more narrow gamut space, any colors that do not fit in the narrow gamut space are clipped.

### Creating images with `toDataURL` and `toBlob`

Historically, `toDataURL` and `toDataBlob` did not specify a color profile in the resulting image.

This behavior is now changed to [specify](https://html.spec.whatwg.org/multipage/canvas.html#serialising-bitmaps-to-a-file) that the resulting image, when allowed by the image format, should include a color profile indicating the source's color space.

This means that doing a "round trip" using `toDataURL` or `toBlob` and then drawing the result using `drawImage` will lose no fidelity (up to compression artifacts).

## Related rendering contexts

This section discusses other rendering contexts besides `CanvasRenderingContext2D`.

### `ImageBitmapRenderingContext` (no changes)

By default, displaying content as an `ImageBitmap` that is provided to an `ImageBitmapRenderingContext` should be identical to displaying that content in an element (e.g, as a `<img>` or `<video>`).

To this end, `ImageBitmap` should, by default, not specify a color space.
The color space information is internal to the `ImageBitmap`.
Like all other sources, when it is drawn in a `CanvasRenderingContext2D`, it is to be converted to the context's color space first.

`ImageBitmapRenderingContext` should not specify a color space.
It should display its contents in full fidelity.
Specifying a color space would only add an artificial limitation to the ability to display content.

An optional color space argument may be added to `ImageBitmapOptions` in the future, for use with WebGL and WebGPU texture uploading (which are not part of this feature).

### `PaintRenderingContext2D` (no changes)

For a `PaintRenderingContext2D`, the actual pixel values and output color space are not observable to the application (`getImageData` is removed from this interface).
The user agent should be able select the best color space for the display device, just as it does for deciding the color space in which regular elements are to be drawn and composited.
This best color space can potentially change behind the scenes (e.g, when a window is moved between display devices).
Similarly to `ImageBitmapRenderingContext`, having the application specify a color space for `PaintRenderingContext2D` would add an artificial limitation.

There have been discussions about, at some point, specifying a working color space for an element.
In that situation the `PaintRenderingContext2D` would inherit that element-specified working color space.

### `WebGLRenderingContextBase` (not part of this feature)

Note that this is being included for completeness, and is not part of the feature being described in this document.

The `WebGLRenderingContextBase` will have a `PredefinedColorSpace` attribute that defines how to interpret the pixels written by WebGL, and the color space to convert input content to when uploading them as textures.
The existing behavior is for this space to implicitly be sRGB.

See KhronosGroup PR [here](https://github.com/KhronosGroup/WebGL/pull/3292).

### `GPUCanvasContext` (not part of this feature)

Note that this is being included for completeness, and is not part of the feature being described in this document.

Similarly to WebGL, WebGPU will allow specifying the color space of the `GPUSwapChain` for output, and for conversion when importing content as textures.

## Examples

In these examples, we will use the variable `myWCGImage` to refer to a wide color gamut image.
[This page](https://webkit.org/blog-files/color-gamut/comparison.javascript) gives several good examples of wide color gamut images.
For the purposes of these examples, one could image `myWCGImage` to be the example image where the red WebKit logo is drawn using sRGB's red, on a background of Display P3's red.

### Selecting the best color space match for the user agent's display device

In this example, the application queries if the target display device has a color gamut that mostly covers the P3 gamut.
If so, the application selects Display P3 as the canvas's color space.

```javascript
  // Note that the gamut is named p3, but the color space is 'display-p3'.
  let matchingColorSpace = window.matchMedia(
      '(color-gamut: p3)').matches ? 'display-p3' : 'srgb';
  let canvas = document.getElementById('MyCanvas');
  let context = canvas.getContext('2d', {colorSpace:matchingColorSpace});
```

### Example of querying the presence of color space support

In this example, the application specifies Display P3 for its color space.
To determine if the user agent supports this feature, the application calls `getContextAttributes` and queries if there exists a `colorSpace` entry.
If the user agent does not support this feature, then there will be no such entry.
The application also prints the value (which [must be](https://javascript.spec.whatwg.org/multipage/canvas.javascript#2d-context-creation-algorithm) the same as the value specified in the input `CanvasRenderingContext2DSettings`).

```javascript
  let canvas = document.getElementById('MyCanvas');
  let context = canvas.getContext('2d', {colorSpace:'display-p3'});
  let attributes = context.getContextAttributes();
  if ('colorSpace' in attributes) {
    console.log('Color space support is present!');
    console.log('Color space is: ' + attributes.colorSpace);
  }
```

### Example of an invalid color space

Because the color space in `CanvasRenderingContext2DSettings` is an enum, invalid color spaces will be caught by the IDL.
The following code will throw an exception.

```javascript
  let canvas = document.querySelector('canvas');
  let context = canvas.getContext('2d', {colorSpace:'the-best-color-space!'});
```

### Example of drawing a wide color gamut image

In this example, a wide color gamut image is drawn to a Display P3 canvas.
When drawn, the image is converted from whatever color space it is in, to Display P3.

```javascript
  let wcgCanvas = document.getElementById('MyWcgCanvas');
  let wcgContext = wcgCanvas.getContext('2d', {colorSpace:'display-p3'});
  wcgContext.drawImage(myWCGImage, 0, 0, myWCGImage.width, myWCGImage.height);
```

Note that if we did not specify the `colorSpace` entry, then the resulting context would be limited to sRGB.
In that case, the resulting canvas would clip colors to the sRGB gamut.
The result would be what is seen on the left side of the example page linked at the top of this document.

```javascript
  let canvas = document.getElementById('MyCanvas');
  let context = wcgCanvas.getContext('2d');
  context.drawImage(myWCGImage, 0, 0, myWCGImage.width, myWCGImage.height)
```

### Example of reading pixels from a Display P3 canvas

In this example (taken from the ["pixel manipulation" section](https://javascript.spec.whatwg.org/multipage/canvas.javascript#pixel-manipulation) of the HTML spec) demonstrates conversion of CSS colors, and reading back colors.

```javascript
  let canvas = document.querySelector('canvas');
  let context = canvas.getContext('2d', {colorSpace:'display-p3'});

  // Draw a red rectangle. Note that the hex color notation
  // specifies sRGB colors.
  context.fillStyle = "#FF0000";
  context.fillRect(0, 0, 64, 64);

  // Get the image data.
  let pixels = context.getImageData(0, 0, 1, 1);

  // This will print 'display-p3', reflecting the default behavior
  // of returning image data in the canvas's color space.
  console.log(pixels.colorSpace);

  // This will print the values 234, 51, and 35, reflecting the
  // red fill color, converted to 'display-p3'.
  console.log(pixels.data[0]);
  console.log(pixels.data[1]);
  console.log(pixels.data[2]);
```

If one wanted the returned pixel to be converted to sRGB, then the following synatax for `getImageData` could be used.
The resulting values in `pixels.data` would be `255, 0, 0, 255`.

```javascript
  let pixels = context.getImageData(0, 0, 1, 1, colorSpace:'srgb');
```

### Example of using an offscreen canvas

In this example, an `OffscreenCanvasRenderingContext2D` is created using the same `colorSpace` entry in `CanvasRenderingContext2DSettings` as is used in a `CanvasRenderingContext2D`.
The contents of the offscreen context are transferred to a `ImageBitmap`.
The `ImageBitmap` is then displayed in a `ImageBitmapRenderingContext`.
Note no color space is specified in the creation or rendering of the `ImageBitmap` (its internal representation is chosen to represent its source at maximum fidelity).

```javascript
  // Create an offscreen WCG context, and draw a WCG image to it.
  let offscreen = new OffscreenCanvas(100, 100);
  let context = offscreen.getContext('2d', {colorSpace:'display-p3'});
  context.drawImage(myWCGImage, 0, 0, myWCGImage.width, myWCGImage.height)

  // Transfer to an ImageBitmap, and display in an ImageBitmapRenderingContext.
  let offscreen_bitmap = offscreen.transferToImageBitmap();
  let canvas = document.getElementById('MyCanvas');
  let context_onscreen = canvas.getContext('bitmaprenderer');
  context_onscreen.transferFromImageBitmap(offscreen_bitmap);
```

## Future work and other references.

The end of the aforementioned [proposal document](https://github.com/WICG/canvas-color-space/blob/main/CanvasColorSpaceProposal.md) discusses several of the designs considered but discarded.

There are ongoing preliminary discussions about adding [wider gamut color spaces and high dynamic range support](https://github.com/w3c/ColorWeb-CG/blob/master/hdr_html_canvas_element.md).

