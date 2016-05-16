# Color managing canvas contents

## Use Case Description
* Contents displayed through a canvas element should be color managed in order to minimize differences in appearance across browsers and display devices. Improving color fidelity matters a lot for artistic uses (e.g. photo and paint apps) and for e-commerce (product presentation).
* Canvases should be able to take advantage of the full color gamut of the display device.
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

Add a canvas color space creation parameter that allows user code to chose between backwards compatible behavior and color managed behaviors.  Color space support is also extended to image storage intefaces that interact with canvases, such as CanvasPattern, ImageData and ImageBitmap.

### Processing Model

#### The colorSpace canvas creation parameter

IDL:
<pre>
enum CanvasColorSpace {
  "legacy-srgb",
  "srgb",
  "linear-rec-2020",
  "optimal",
  "optimal-strict"
};

dictionary CanvasRenderingContext2DSettings {
  boolean alpha = true;
  CanvasColorSpace colorSpace = "legacy-srgb";
};
</pre>

Example:
<pre>
canvas.getContext('2d', { colorSpace: "srgb"})
</pre>

#### The "legacy-srgb" color space

This mode assures backwards compatible behavior and is designed to accomodate use cases that require colors to match CSS on implementation that do not color-manage CSS.
* Guarantees that color values used as fillStyle or strokeStyle match the appearance of the same color value when it is used in CSS.
* Color management behavior is implementation specific, may not use strict sRGB space, but is expected to be near sRGB. For example, could be display referred color space.
* toDataURL/toBlob produce resources with no color profile (backwards compat)
* Image resources with no color profile are never color corrected (backwards compat). This rule and the previous one allow for lossless toDataURL/drawImage round trips, which is a significant use case.

#### The "srgb" color space

* May break color matching with CSS on implementations that do not color-manage CSS.
* 8 bit unsigned integers per color component.
* All content drawn into the canvas must be color corrected to sRGB
* displayed canvases must be color corrected for the display if a display color profile is available. This color correction happens downstream at the compositing stage, and has no script-visible side-effects.
* Compositing, filtering and interpolation operations must perform all arithmetic in '''linear''' sRGB space.
* toDataURL/toBlob produce resources tagged as being in the sRGB color space
* Images with no color profile, when drawn to the canvas, are assumed to already be in the sRGB color space.

#### The "linear-rec-2020" color space
* Color space provided for wide gamut and high dynamic range rendering.
* User agents may decide not to support the mode, based on host machine capabilities
* Uses 16-bit floating point representation.
* The color space corresponds to ITU-R Recommendation BT.2020, '''without gamma compression'''.
* toDataURL/toBlob convert image data to the rec-2020 color space (with gamma), and produce image resources with at least 12 bits per color component, if the format supports it. Thus, in the case of the png format, which supports 8 or 16 bits per component, 16bpc would be used.
* Image with no color profile, when drawn to the canvas, are assumed to be in the sRGB color space, and are converted to linear-rec-2020 for the purpose of the draw.

#### The "optimal" color space
The "optimal" option lets the user agent decide which space is optimal for the current display device based on the device's capabilities and color profile characteristics.
* May select a color space that is not defined in this specification.
* The user agent must select a color space with a sufficiently wide gamut to avoid undue gamut clipping when displaying to the current display device.
* The user agent must select a color space with sufficient bit-depth to avoid undue banding given the current display device
* The user agent must select the color space that is the most efficient in terms of memory usage, while respecting all the other rules. If more than one color space matches the criteria

#### The "optimal-strict" color space
Similar to "optimal", but with additional constraints:
* Canvas operations must perform compositing, filtering and interpolation arithmetic in linear space.
* This mode may not select "legacy-srgb".
* The alpha component must have at least 8 bits.

#### Non-standard color spaces
User agents may support color spaces not defined in this specification. An important use case for non-standard spaces is to provided implementers with some latitude to create color spaces that are high-performing "optimal" matches for certain combinations of display, CPU and GPU technologies.

#### Feature detection

Rendering context objects (2d, WebGL) are to expose a new "settings" attribute, which represents the settings that were successfully applied at context creation time.

Note: An alternative approach that was considered was to augment the probablySupportsContext() API by making it check the second argument.  That approach is difficult to consolidate with how dictionary argument are meant to work, where unsupported entries are just ignored.

#### ImageBitmap

ImageBitmap objects are augmented to have an internal color space attribute of type CanvasColorSpace. The colorSpaceConversion creation attribute also accepts enum values that correspond to CanvasColorSpace values. Specifying a CanvasColorSpace value results in a conversion of the image to the specified color space.

#### ImageData

IDL
<pre>
typedef (Uint8ClampedArray or Float32Array) ImageDataArray;

[Constructor(unsigned long sw, unsigned long sh, optional CanvasColorSpace colorSpace = "legacy-srgb"),
 Constructor(ImageDataArray data, unsigned long sw, optional unsigned long sh, optional CanvasColorSpace colorSpace),
 Exposed=(Window,Worker)]
interface ImageData {
  readonly attribute unsigned long width;
  readonly attribute unsigned long height;
  readonly attribute ImageDataArray data;
  readonly attribute CanvasColorSpace colorSpace;
};
</pre>

* <code>data</code> is a Uint8ClampedArray if colorSpace is "srgb" or "legacy-srgb"
* <code>data</code> is a Float32Array if colorSpace is "linear-rec-2020"
* Non-standard color spaces that have more than 8bpc and use an integer format may use Uint16ClampedArray for <code>data</code>. Regardless of the number of bits used by the color space, the representation of color components as Uint16 values must use the range [0,65535].
* getImageData() produces an ImageData object in the same color space as the source canvas
* putImageData() performs a color space conversion to the color space of the destination canvas.

### Limitations 
* No support for arbitrary color spaces and bit depth.  This capability could be added in the future.  The current proposal attempts to solve the problem with a minimal API surface, and keeps the implementation scope reasonable.  The extensible design will allow us to extend the capabilities in the future if necessary.  The rec-2020 space was chosen for its very wide gamut and its non-virtual primary colors, which strikes a balance that is deemed practical.
* toDataURL is lossy when us on a canvas that is in the linear-rec-2020 space. Possible future improvements could solve or mitigate this issue by adding more file formats or adding options to specify the resource color space.
* ImageData uses float32, which is inefficient due to memory consumption and necessary conversion operations. Float32 was chosen because it is convenient for manipulation (e.g. image processing) due to its native support in JavaScript (and current CPUs). A possible extension would be to add and option for rec-2020 content to be encoded as float16s packed into Uint16 values.


### Security and privacy issues
Some current implementations of CanvasRenderingContext2D color correct image resources for the display as they are drawn to the canvas. In other words, the canvas is in output referred color space. This is a known fingerprinting vulnerability since it exposes the user's display's color profile to scripts via getImageData.  The current proposal does not solve the fingerprinting issue because it will still exist in legacy-srgb.  To solve the problem, implementations must color-correct CSS colors, then by extension, legacy-srgb mode will be in the true sRGB color space by virtue of the color matching rules outlined above.  When that becomes the case, images drawn to canvases will be color corrected to sRGB, which solves the problem.  There is resistance to adopting this model because going through an sRGB intermediate is lossy compared to directly color correcting images for the display in a single pass (may cause banding and gamut clipping).  This feature proposal mitigates the lossiness argument thanks to the linear-rec-2020 option.

Implementors should be mindful of fingerprinting in their designs of non-standard color spaces. E.g. they should avoid providing color spaces that expose the display's specific profile, though doing so is tempting for an "optimal" match.

### Implementation notes 
* Because float16 arithmetic is supported by most GPUs, but not by CPUs, implementations should probably opt to not support linear-rec-2020 on hardware that does not provide any native support.
* When available, the srgb color space should use GPU API extensions for sRGB support. This will streamline the conversion overhead for performing filtering and compositing in linear space.

### Adoption
Lack of color management and color interoperability is a longstanding complaint about the canvas API.
Authors of games and imaging apps are expected to be enthusiastic adopters.

## Unresolved Issues

* Should more standard color spaces be added? We must strike a balance between providing useful interoperable options and avoiding an over-spec'ing. Risk associated with over-spec'ing include: high implementation burden, compatibility risks, shipping is forever. Suggestions for additional color spaces include: 
    * DCI-P3 or Apple's P3 (8bpc possibly 10bpc?). One of the appeals of P3 is to match the media query API (c.f. https://drafts.csswg.org/mediaqueries-4/#color-gamut). It also offers a reasonable halfway compromise between sRGB and rec-2020, and it is a good match for some current display profiles.
    * 8bpc rec-2020, with gamma compression. This color space uses the same gamma curve as sRGB, so it is convenient to support on current GPUs.  Would offer wide gamut with low memory cost.  There are concerns that this space may be of limited usefulness due to potential banding issues, since the gamut is so wide.
    * A space based on AdobeRGB with 8bpc (can't "Adobe" in the name). This space would be close to the display profile spaces of many current displays. Could use the same primaries as AdobeRGB with the gamma curve of sRGB for ease of implementation.

* Should non-standard color spaces use vendor prefixes to avoid compatibility issues or name clashes?

* Should black level compensation be implied in "linear spaces", such that (0,0,0) always represent absolute black in linear color values?

* Should linear-rec-2020 or non-standard color spaces be allowed to operate outside of the [0,1] range in 2D canvases? The current definitions of compositing and blending modes were not designed with this in mind.

* Should it be possible to specify the color space parameter as an array representing a fallback list? Example: <code>canvas.getContext('2d', {colospace: ["p3", "linear-rec-2020"]});</code> would mean use p3 if possible; if not fallback to linear-rec-2020.  Since color spaces are already feature detectable, this would be a convenience feature.

* Should we support custom color spaces based on ICC profiles? Would offer ultimate flexibility. Would be hard to make implementations as efficient as built-in color spaces, in particular for compositing in linear space. Referencing a remote ICC profile may be problematic because getContext() is synchronous. Could solve that by using a Blob rather than a URI for specifying the ICC profile.

## Proposal History

This proposal was originally incubated in the Khronos 3D Web group, with the participation of engineers from Google, Microsoft, Apple, Nvidia, and others.

This propsal was further discussed in the WHATWG github issue tracker and contains modifications based on feedback that was provided in the following thread: https://github.com/whatwg/html/issues/299

The current venue for discussing this proposal is a [W3C WICG Discourse thread](https://discourse.wicg.io/t/canvas-color-spaces-wide-gamut-and-high-bit-depth-rendering/1505)

