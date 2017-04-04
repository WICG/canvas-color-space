# Color managing canvas contents

## Use Case Description
* Contents displayed through a canvas element should be color managed in order to minimize differences in appearance across browsers and display devices. Improving color fidelity matters a lot for artistic uses (e.g. photo and paint apps) and for e-commerce (product presentation).
* Canvases should be able to take advantage of the full color gamut and dynamic range of the display device.
* Creative apps that do image manipulation generally prefer compositing, filtering and interpolation calculations to be performed in a linear color space.

### Current Limitations
* The color space of canvases is undefined in the current specification.
* The bit-depth of canvases is currently fixed to 8 bits per component, which is below the capabilities of some monitors. Monitors with higher contrast ratios require more bits per component to avoid banding.

### Current Usage and Workarounds
The lack of color space interoperability is hard to work around. With some browser implementations that color correct images drawn to canvases by applying the display profile, apps that want to use canvases for color corrected image processing are stuck doing convoluted workarounds, such as:
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
* Engineers from the Google Photos, Maps and Sheets teams have expressed a desire for canvases to become color managed.  Particularly for the use case of resizing an imaging, using a canvas, prior to uploading it to the server, to save bandwidth. The problem is that the images retrieved from a canvas are in an undefined color space and no color space information is encoded by toDataURL or toBlob. 

## Proposed Solution

* Clearly define the color space of canvases.  Most current browser implementations are either color-managed or are currently actively working on becoming color-managed.  Therefore, it will be possible to specify that the default color space of canvases as 'srgb' instead of leaving it undefined in the spec (as is currently the case due to lack of interoperability).
* Add a canvas context creation attribute to specify a color space.
* Add a canvas context creation attribute to specify a numeric format for storing pixel values.
* Color space and numeric format parameters are also extended to image storage interfaces that interact with canvases, such as CanvasPattern, ImageData and ImageBitmap.

### Processing Model

#### The colorSpace canvas creation parameter

IDL:
<pre>
enum CanvasColorSpace {
  "srgb", // default
  "rec2020",
  "p3",
};

enum CanvasPixelFormat {
  "8-8-8-8", // default
  "10-10-10-2",
  "12-12-12-12",
  "float16",
};

partial dictionary CanvasRenderingContext2DSettings {
  CanvasColorSpace colorSpace = "srgb";
  CanvasPixelFormat pixelFormat = "8-8-8-8";
  boolean linearPixelMath = false;
};

partial interface CanvasRenderingContext2D {
  CanvasRenderingContext2DSettings getContextAttributes();
};
</pre>

Example:
<pre>
canvas.getContext('2d', { colorSpace: "p3", pixelFormat: "10-10-10-2", linearPixelMath: true});
</pre>


#### The "srgb" color space

* Guarantees that color values used as fillStyle or strokeStyle exactly match the appearance of the same color value when it is used in CSS.
* On implementations that do not color-manage CSS colors, the canvas "srgb" color space must not be color-managed either, in order to preserve color-matching between CSS and canvas-rendered content. This situation shall be referred to as the "legacy behavior".
* All content drawn into the canvas must be color corrected to sRGB. Exception: User agents that implement the legacy behavior must apply color correction steps that match the color correction that is applied to image image resource that are displayed via CSS.
* Displayed canvases must be color corrected for the display if a display color profile is available. This color correction happens downstream at the compositing stage, and has no script-visible side-effects.
* toDataURL/toBlob produce resources tagged as being in the sRGB color space, if the encoding format supports colorspace tagging or embedded color profiles. Exception: User agents that implement the legacy behavior must not encode any color space metadata.
* Images with no color profile, when drawn to the canvas, are assumed to already be in the canvas's color space, and require no color transformation.

#### The "rec2020" color space
* Support is optional.
* User agents that select the rec2020 color gamut in CSS media queries must support this color space.
* The color space corresponds to ITU-R Recommendation BT.2020.
* toDataURL/toBlob convert image data to the rec-2020 color space.
* Images with no color profile, when drawn to the canvas, are assumed to be in the sRGB color space, and are converted to rec2020 for the purpose of the draw.

#### The "p3" color space
* Support is optional.
* User agents that select either of the the p3 or the rec2020 color gamuts in CSS media queries must support this color space.
* The color space corresponds to the DCI P3 specification (TODO: add link).
* toDataURL/toBlob convert image data to the p3 space.
* Images with no color profile, when drawn to the canvas, are assumed to be in the sRGB color space, and are converted to p3 for the purpose of the draw.

#### The pixelFormat context creation attribute
The pixelFormat attributes specifies the numeric types to be used for storing pixel values.
* Support for "8-8-8-8" is mandatory. All other formats ar optional.
* When an unsupported format is requested, the format shall fall back to "8-8-8-8".
* With formats that use integer numeric types for the color channels, the canvas pixel buffer shall store gamma-corrected color values, using the transfer curves prescribed by the specification of the current color space.
* With formats that use floating-point numeric types for the color channels, the canvas pixel buffer shall store non-gamma-corrected (a.k.a linear) color values.
* Float values outside of [0,1] range can be used to represent colors outside of the chosen color gamut. This allows float pixel formats to represent all possible colors and brightness levels. How values outside of [0,1] are displayed depends on the capabilites of the device and output display. Some implementations may simply clamp these values to [0,1]. If the device and display are capable, a pixel value of (2,2,2) should be twice as bright as (1,1,1).

#### Selecting the best color space match for the user agent's display device
<pre>
var colorSpace = window.matchMedia("(color-gamut: rec2020)").matches ? "rec2020" : 
    (window.matchMedia("(color-gamut: p3)").matches ? "p3" : "srgb");
</pre>

#### Selecting the best pixelFormat for the user agent's display device
Selection should be based on the best color space match (see above). For srgb, at least 8 bits per component is recommended; for p3, 10 bits; and for rec2020, 12 bits.  The float16 format is suitable for any colorspace.  There may soon be a proposal to add a way of detecting HDR displays, possibly something like "window.screen.isHDR()" (TBD), which would be a good hint to use the float16 format.

#### The linearPixelMath context creation attribute
The linerPixelMath context creation attribute indicates whether gamma-corrected pixel values should be transiently converted to linear space for performing arithmetic on color values.
* Defaults to false.
* Has no effect if the pixelFormat uses floating-point numeric types.
* Affects the behavior of: globalCompositeOperation, image resampling performed by drawImage, gradient interpolation.

#### Non-standard color spaces
For future consideration: support could be added for color space defined using the [CSS @color-profile rule](https://www.w3.org/TR/css-color-4/#at-profile).

#### Compositing the canvas element
Canvas contents are composited in accordance with the canvas element's style (e.g. CSS compositing and blending rules). The necessary compositing operations must be performed in an intermediate colorspace, the compositing space, that is implementation specific. The compositing space must have sufficient precision and a sufficiently wide gamut to guarantee no undue loss of precision or gamut clipping in bringing the canvas's contents to the display.

#### Feature detection

2D rendering contexts are to expose a new getContextAttributes() method, that works much like the method of the same name on WebGLRenderingContext. The method returns the "actual context attributes" which represents the settings that were successfully applied at context creation time. The settings attribute reflects the result of running the algorithm for coercing the settings argument for 2D contexts, as well as the result of any fallbacks that may have happened as a result of options not being supported by the UA.

Web apps may infer that a user agent that does not implement getContextAttributes() does not support the colorSpace, pixelFormat and linearPixelMath attributes.

Note: An alternative approach that was considered was to augment the probablySupportsContext() API by making it check the second argument.  That approach is difficult to consolidate with how dictionary arguments are meant to work, where unsupported entries are just ignored.

#### ImageBitmap

ImageBitmap objects are augmented to have an internal color space attribute of type CanvasColorSpace and an internal pixelFormat attribute of type CanvasPixelFormat. The colorSpaceConversion creation attribute also accepts enum values that correspond to CanvasColorSpace values. Specifying a CanvasColorSpace value results in a conversion of the image to the specified color space.

#### ImageData

IDL
<pre>

enum ImageDataStorageType {
  "uint8", // default
  "uint16",
  "float32",
};

typedef (Uint8ClampedArray or Uint16ClampedArray or Float32Array) ImageDataArray;

dictionary ImageDataColorSettings {
  CanvasColorSpace colorSpace = "srgb";
  ImageDataStorageType storageType = "uint8";
};

[Constructor(unsigned long sw, unsigned long sh, optional ImageDataColorSettings imageDataColorSettings),
 Constructor(ImageDataArray data, unsigned long sw, optional unsigned long sh, optional ImageDataColorSettings imageDataColorSettings),
 Exposed=(Window,Worker)]
interface ImageData {
  readonly attribute unsigned long width;
  readonly attribute unsigned long height;
  readonly attribute ImageDataArray data;
  readonly attribute ImageDataColorSettings colorAttributes;
};
</pre>

* When using the constructor that takes an ImageDataArray parameter, the "storageType" setting is ignored.
* getImageData() produces an ImageData object with the same color space as the source canvas, using an ImageDataArray of a type that is appropriate for the pixelFormat of the source canvas (smallest possible numeric size that guarantees no loss of precision).
* putImageData() performs a color space conversion to the color space of the destination canvas.

### Limitations 
* toDataURL and toBlob are lossy, depending on the file format, when used on a canvas that has a pixelFormet other than 8-8-8-8. Possible future improvements could solve or mitigate this issue by adding more file formats or adding options to specify the resource color space.

### Implementation notes 
* When possible, the linerPixelMath option should use GPU API extensions for sRGB transfer curve support. This will streamline the gamma compression/decrompression overhead for performing filtering and compositing in linear space.

### Adoption
Lack of color management and color interoperability is a longstanding complaint about the canvas API.
Authors of games and imaging apps are expected to be enthusiastic adopters.

## Unresolved Issues

* Should black level compensation be implied in "linear spaces", such that (0,0,0) always represents absolute black in linear color values?

* Should it be possible to specify the color space parameter as an array representing a fallback list? Example: <code>canvas.getContext('2d', {colorSpace: ["p3", "rec2020"]});</code> would mean use p3 if possible; if not supported, try rec2020; and if rec2020 is not supported, fallback to srgb, which is the default.  Since color spaces are already feature detectable, this would be a convenience feature.  Same question applies to pixelFormats.

* Should we support custom color spaces based on ICC profiles? Would offer ultimate flexibility. Would be hard to make implementations as efficient as built-in color spaces, in particular for implement linearPixelMath for profiles that have arbitrary transfer curves. 

## Proposal History

This propsal was originally incubated in the WHATWG github issue tracker and incorporates feedback that was provided in the following thread: https://github.com/whatwg/html/issues/299

This proposal was further discussed in the Khronos WebGL working group, with the participation of engineers from Apple, Google, Microsoft, Mozilla, Nvidia, and others.

The current venue for discussing this proposal is a [W3C WICG Discourse thread](https://discourse.wicg.io/t/canvas-color-spaces-wide-gamut-and-high-bit-depth-rendering/1505)

