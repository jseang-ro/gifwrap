# gifwrap

A Jimp-compatible library for working with GIF images 

## Overview

`gifwrap` is a minimalist library for working with GIFs in Javascript, supporting both single- and multi-frame GIFs. It reads GIFs into an internal representation that's easy to work with and allows for making GIFs from scratch. The frame class is structured to make it easy to move images between [`Jimp`](https://github.com/oliver-moran/jimp) and `gifwrap` for more sophisticated image manipulation in `Jimp`, but the module has no dependency on `Jimp`.

The library uses Dean McNamee's [`omggif`](https://github.com/deanm/omggif) GIF encoder/decoder by default (locally tweaked), but it employs an abstraction that allows using other encoders and decoders as well, once suitably wrapped.

At present, the module only works in Node.js. Includes Typescript typings.

## Installation

```
npm install gifwrap --save
```

or

```
yarn add gifwrap
```

## Usage

You can work with either GIF files or GIF encodings, and you can create GIFs from scratch.

The GifFrame class represents a single image frame, and the library largely represents a GIF as an array of GifFrame instances. For example, here is how you create a GIF from scratch:

```js
const { GifFrame, GifUtil, GifCodec } = require('gifwrap');
const width = 200, height = 100;
const frames = [];

let frame = new GifFrame(width, height, { delayCentisecs: 10 });
// modify the pixels at frame.bitmap.data
frames.push(frame);
frame = new GifFrame(width, height, { delayCentisecs: 15 });
// modify the pixels at frame.bitmap.data
frames.push(frame);
// add more frames as desired...

// to write to a file...
GifUtil.write("my-creation.gif", frames, { loops: 3 }).then(gif => {
    console.log("written");
});

// to get the byte encoding without writing to a file...
const codec = new GifCodec();
codec.encodeGif(frames).then(gif => {
    // byte encoding is now in gif.buffer
});

```

Images are represented within a GifFrame exactly as they are in a `Jimp` image. In particular, each GifFrame instance has a `bitmap` property having the following structure:

* `frame.bitmap.width` - Width of image in pixels
* `frame.bitmap.height` - Height of image in pixels
* `frame.bitmap.data` - A Node.js Buffer that can be accessed like an array of bytes. Every 4 adjacent bytes represents the RGBA values of a single pixel. These 4 bytes correspond to red, green, blue, and alpha, in that order. Each pixel begins at an index that is a multiple of 4.

GIFs do not support partial transparency, so within `frame.bitmap.data`, the pixels having alpha value 0xFF are treated as opaque and pixels of all other alpha values are treated as transparent. The encoder ignores the RGB values of transparent pixels.

`gifwrap` also provides utilities for reading GIF files and for parsing raw encodings:

```js
const { GifUtil } = require('gifwrap');
GifUtil.read("fancy.gif").then(inputGif =>

    inputGif.frames.foreach(frame => {

        const buf = frame.bitmap.data;
        frame.scanAllCoords((x, y, bi) => {

            // Halve all grays on right half of image.

            if (x > inputGif.width / 2) {
                const r = buf[bi];
                const g = buf[bi + 1];
                const b = buf[bi + 2];
                const a = buf[bi + 3];
                if (r === g && r === b && a === 0xFF) {
                    buf[bi] /= 2;
                    buf[bi + 1] /= 2;
                    buf[bi + 2] /= 2;
                }
            }
        });
    });

    // Pass inputGif to write() to preserve the original GIF's specs.
    return GifUtil.write("modified.gif", inputGif.frames, inputGif).then(outputGif => {
        console.log("modified");
    });
});
```

```js
const { GifUtil, GifCodec } = require('gifwrap');
const codec = new GifCodec();
const byteEncodingBuffer = getByteEncodingForSomeGif();

codec.decodeGif(byteEncodingBuffer).then(sourceGif => {

    const edgeLength = Math.min(sourceGif.width, sourceGif.height);
    sourceGif.frames.forEach(frame => {

        // Make each frame a centered square of size edgeLength x edgeLength.
        // Note that frames may vary in size and that reframe() works even if
        // the frame's image is smaller than the square. Should this happen,
        // the space surrounding the original image will be transparent.

        const xOffset = (frame.bitmap.width - edgeLength)/2;
        const yOffset = (frame.bitmap.height - edgeLength)/2;
        frame.reframe(xOffset, yOffset, edgeLength, edgeLength);
    });

    // The encoder determines GIF size from the frames, not the provided spec.
    return GifUtil.write("modified.gif", sourceGif.frames, sourceGif).then(outputGif => {
        console.log("modified");
    });
});
```

Notice that both encoding and decoding yields a GIF object. This is an instance of class Gif, and it provides information about the GIF, such as its size and how many times it loops. But notice also that the code only ever encodes an array of frames and a spec for a GIF. The spec is the subset of Gif properties that can't be inferred from the frames -- namely, how many times the GIF loops and how to attempt to package the color tables within the encoding.

## Leveraging Jimp

This module was originally written as a wrapper around Jimp images -- hence its name -- and then with frames as subclasses of Jimp images. Neither approach worked out well. The final approach requires just a tad of legwork to use `gifwrap` images within Jimp.

Both Jimp images and GifFrame instances share the `bitmap` property. By transferring this property back and forth between Jimp images and GifFrame instances, an image can be moved back and forth between the two libraries.

You can construct a GifFrame from a Jimp image as follows:

```js
const { GifFrame } = require('gifwrap');
const Jimp = require('jimp');
const j = new Jimp(200, 100, 0xFFFFFFFF);

const fCopied = new GifFrame(j); // frame containing clone of the Jimp bitmap
const fShared1 = new GifFrame(1, 1, 0);
fShared1.bitmap = j.bitmap; // frame sharing a bitmap with Jimp
const fShared2 = new GifFrame(j.bitmap.width, j.bitmap.height, j.bitmap.data); // also shared
```

And you can construct a Jimp instance from a GifFrame image as follows:

```js
const { BitmapImage, GifFrame } = require('gifwrap');
const Jimp = require('jimp');
const f = new Frame(200, 100, 0xFFFFFFFF);

const jCopied = new Jimp(1, 1, 0);
jCopied.bitmap = new BitmapImage(f); // jimp containing a clone of the frame bitmap
jShared = new Jimp(1, 1, 0);
jShared.bitmap = f.bitmap; // jimp sharing a bitmap with the frame
```

You may want to create helper functions to encapsulate this behavior. In order to avoid creating a dependency on `Jimp`, such helper functions do not appear in `gifwrap`.

## Encoders and Decoders

`gifwrap` provides a default GIF encoder/decoder, but it is architected to be able to work with other encoders and decoders. The encoder and decoder may even be separate implementations. Encoders and decoders have varying capabilities, performance measures, and levels of reliability.

GifCodec is the default implementation, and it's both an encoder and a decoder. It's an adapter that wraps a locally-modified version of the [`omggif`](https://github.com/deanm/omggif) module. It appears to support a broad variety of GIFs, although it cannot produce an interlaced encoding (which there is little need for anyway). The `gifwrap` test suite also happens to serve as a test suite for `omggif`, by virtue of using `omggif` underneath. It revealed very few and only minor bugs, which the local copy of `omggif` corrects.

An encoder need only implement GifCodec's [`encodeGif()`](#GifCodec+encodeGif) method, and a decoder need only implement its [`decodeGif()`](#GifCodec+decodeGif) method. See the descriptions of those methods for the requirement details. Although GifCodec is stateless, so that instances an be reused across multiple encodings and decodings, third party encoders and decoders need not be. However, applications that use the library with stateful encoders will need to be aware of the need to create new instances.

To use a third-party encoder or decoder with the GifUtil `write()` and `read()` functions, just pass an instance of the encoder or decoder as the last parameter to `write()` or `read()`, respectively. For example:

```js
const { GifUtil } = require('gifwrap');
const SnazzyDecoder = require('gifwrap-snazzy-decoder');
const AwesomeEncoder = require('gifwrap-awesome-encoder');

GifUtil.read("fancy.gif", new SnazzyDecoder()).then(gif =>

    /*...*/

    return GifUtil.write("modified.gif", gif.frames, gif, new AwesomeEncoder()).then(newGif => {
        console.log("modified");
    });
});
```

## API Reference

The `gifwrap` module provides the following classes and namespaces:  

* **gifwrap**

    * [.**Gif**](#new_Gif_new)
    * [.**BitmapImage**](#new_BitmapImage_new)
    * [.**GifFrame**](#new_GifFrame_new)
    * [.**GifUtil**](#TBD)
    * [.**GifCodec**](#new_GifCodec_new)
    * [.**GifError**](#new_GifError_new)


* [BitmapImage](#BitmapImage)

    * [new BitmapImage()](#new_BitmapImage_new)

    * [.blit(toImage, toX, toY, fromX, fromY)](#BitmapImage+blit)

    * [.fillRGBA(rgba)](#BitmapImage+fillRGBA)

    * [.getRGBA(x, y)](#BitmapImage+getRGBA)

    * [.greyscale()](#BitmapImage+greyscale)

    * [.reframe(xOffset, yOffset, width, height, fillRGBA)](#BitmapImage+reframe)

    * [.scale(factor)](#BitmapImage+scale)

    * [.scanAllCoords(scanHandler)](#BitmapImage+scanAllCoords)

    * [.scanAllIndexes(scanHandler)](#BitmapImage+scanAllIndexes)



* [GifFrame](#GifFrame)

    * [new GifFrame()](#new_GifFrame_new)

    * [.getPalette()](#GifFrame+getPalette)



* [GifUtil](#GifUtil)

    * [.cloneFrames(frames)](#GifUtil.cloneFrames)

    * [.getColorInfo(frames, maxGlobalIndex)](#GifUtil.getColorInfo)

    * [.getMaxDimensions(frames)](#GifUtil.getMaxDimensions)

    * [.read(source, An)](#GifUtil.read)

    * [.write(path, frames, spec, encoder)](#GifUtil.write)



* [GifCodec](#GifCodec)

    * [new GifCodec(options)](#new_GifCodec_new)

    * [.decodeGif(buffer)](#GifCodec+decodeGif)

    * [.encodeGif(frames, spec)](#GifCodec+encodeGif)



<a name="new_Gif_new"></a>

### new Gif(buffer, frames, spec)

| Param | Type | Description |
| --- | --- | --- |
| buffer | <code>Buffer</code> | A Buffer containing the encoded bytes |
| frames | [<code>Array.&lt;GifFrame&gt;</code>](#GifFrame) | Array of frames found in the encoding |
| spec | <code>object</code> | Properties of the encoding as listed above |

Gif is a class representing an encoded GIF. It is intended to be a read-only representation of a byte-encoded GIF. Only encoders and decoders should be creating instances of this class.

Property | Description
--- | ---
width | width of the GIF at its widest
height | height of the GIF at its highest
loops | the number of times the GIF should loop before stopping; 0 => loop indefinately
usesTransparency | boolean indicating whether at least one frame contains at least one transparent pixel
colorScope | the scope of the color tables as encoded within the GIF; either Gif.GlobalColorsOnly (== 1) or Gif.LocalColorsOnly (== 2).
frames | a array of GifFrame instances, one for each frame of the GIF
buffer | a Buffer holding the encoding's byte data

Its constructor should only ever be called by the GIF encoder or decoder.

<a name="new_BitmapImage_new"></a>

### new BitmapImage()
BitmapImage is a class that hold an RGBA (red, green, blue, alpha) representation of an image. It's shape is borrowed from the Jimp package to make it easy to transfer GIF image frames into Jimp and Jimp images into GIF image frames. Each instance has the following properties:

Property | Description
--- | ---
width | width of image in pixels
height | height of image in pixels
data | a Buffer whose every four bytes represents a pixel, each sequential byte of a pixel corresponding to the red, green, blue, and alpha values of the pixel

Its constructor supports the following signatures:

new BitmapImage(bitmap: { width: number, height: number, data: Buffer })
new BitmapImage(bitmapImage: BitmapImage)
new BitmapImage(width: number, height: number, buffer: Buffer)
new BitmapImage(width: number, height: number, backgroundRGBA?: number)

When a `BitmapImage` is provided, the constructed `BitmapImage` is a deep clone of the provided one, so that each image's pixel data can subsequently be modified without affecting each other.

`backgroundRGBA` is an optional parameter representing a pixel as a single number. In hex, the number is as follows: 0xRRGGBBAA, where RR is the red byte, GG the green byte, BB, the blue byte, and AA the alpha value. An AA of 0xFF is considered opaque, and all other AA values are treated as transparent.

<a name="BitmapImage+blit"></a>

### *bitmapImage*.blit(toImage, toX, toY, fromX, fromY)

| Param | Type | Description |
| --- | --- | --- |
| toImage | [<code>BitmapImage</code>](#BitmapImage) | Image into which to copy the square |
| toX | <code>number</code> | x-coord in toImage of upper-left corner of receiving square |
| toY | <code>number</code> | y-coord in toImage of upper-left corner of receiving square |
| fromX | <code>number</code> | x-coord in this image of upper-left corner of source square |
| fromY | <code>number</code> | y-coord in this image of upper-left corner of source square |

Copy a square portion of this image into another image.

**Returns**: [<code>BitmapImage</code>](#BitmapImage) - The present image to allow for chaining.  
<a name="BitmapImage+fillRGBA"></a>

### *bitmapImage*.fillRGBA(rgba)

| Param | Type | Description |
| --- | --- | --- |
| rgba | <code>number</code> | Color with which to fill image, expressed as a singlenumber in the form 0xRRGGBBAA, where AA is 0xFF for opaque and any other value for transparent. |

Fills the image with a single color.

**Returns**: [<code>BitmapImage</code>](#BitmapImage) - The present image to allow for chaining.  
<a name="BitmapImage+getRGBA"></a>

### *bitmapImage*.getRGBA(x, y)

| Param | Type | Description |
| --- | --- | --- |
| x | <code>number</code> | x-coord of pixel |
| y | <code>number</code> | y-coord of pixel |

Gets the RGBA number of the pixel at the given coordinate in the form 0xRRGGBBAA, where AA is the alpha value, with 0xFF being opaque.

**Returns**: <code>number</code> - RGBA of pixel in 0xRRGGBBAA form  
<a name="BitmapImage+greyscale"></a>

### *bitmapImage*.greyscale()
Converts the image to greyscale using inferred Adobe metrics.

**Returns**: [<code>BitmapImage</code>](#BitmapImage) - The present image to allow for chaining.  
<a name="BitmapImage+reframe"></a>

### *bitmapImage*.reframe(xOffset, yOffset, width, height, fillRGBA)

| Param | Type | Description |
| --- | --- | --- |
| xOffset | <code>number</code> | The x-coord offset of the upper-left pixel of the desired image relative to the present image. |
| yOffset | <code>number</code> | The y-coord offset of the upper-left pixel of the desired image relative to the present image. |
| width | <code>number</code> | The width of the new image after reframing |
| height | <code>number</code> | The height of the new image after reframing |
| fillRGBA | <code>number</code> | The color with which to fill space added to the image as a result of the reframing, in 0xRRGGBBAA format, where AA is 0xFF to indicate opaque and any other value to indicate transparent. This parameter is only required when the reframing exceeds the original boundaries (i.e. does not simply perform a crop). |

Reframes the image as if placing a frame around the original image and replacing the original image with the newly framed image. When the new frame is strictly within the boundaries of the original image, this method crops the image. When any of the new boundaries exceed those of the original image, the `fillRGBA` must be provided to indicate the color with which to fill the extra space added to the image.

**Returns**: [<code>BitmapImage</code>](#BitmapImage) - The present image to allow for chaining.  
<a name="BitmapImage+scale"></a>

### *bitmapImage*.scale(factor)

| Param | Type | Description |
| --- | --- | --- |
| factor | <code>number</code> | The factor by which to scale up the image. Must be an integer >= 1. |

Scales the image size up by an integer factor. Each pixel of the original image becomes a square of the same color in the new image having a size of `factor` x `factor` pixels.

**Returns**: [<code>BitmapImage</code>](#BitmapImage) - The present image to allow for chaining.  
<a name="BitmapImage+scanAllCoords"></a>

### *bitmapImage*.scanAllCoords(scanHandler)
**See**: scanAllIndexes  

| Param | Type | Description |
| --- | --- | --- |
| scanHandler | <code>function</code> | A function(x: number, y: number, bi: number) to be called for each pixel of the image with that pixel's x-coord, y-coord, and index into the `data` buffer. The function accesses the pixel at this coordinate by accessing the `this.data` at index `bi`. |

Scans all coordinates of the image, handing each in turn to the provided handler function.

<a name="BitmapImage+scanAllIndexes"></a>

### *bitmapImage*.scanAllIndexes(scanHandler)
**See**: scanAllCoords  

| Param | Type | Description |
| --- | --- | --- |
| scanHandler | <code>function</code> | A function(bi: number) to be called for each pixel of the image with that pixel's index into the `data` buffer. The pixels is found at index 'bi' within `this.data`. |

Scans all pixels of the image, handing the index of each in turn to the provided handler function. Runs a bit faster than `scanAllCoords()`, should the handler not need pixel coordinates.

<a name="new_GifFrame_new"></a>

### new GifFrame()
GifFrame is a class representing an image frame of a GIF. GIFs contain one or more instances of GifFrame.

Property | Description
--- | ---
xOffset | x-coord of position within GIF at which to render the image (defaults to 0)
yOffset | y-coord of position within GIF at which to render the image (defaults to 0)
disposalMethod | GIF disposal method; only relevant when the frames aren't all the same size (defaults to 2, disposing to background color)
delayCentisecs | duration of the frame in hundreths of a second
interlaced | boolean indicating whether the frame renders interlaced

Its constructor supports the following signatures:

new GifFrame(bitmap: {width: number, height: number, data: Buffer}, options?)
new GifFrame(bitmapImage: BitmapImage, options?)
new GifFrame(width: number, height: number, buffer: Buffer, options?)
new GifFrame(width: number, height: number, backgroundRGBA?: number, options?)
new GifFrame(frame: GifFrame)

See the base class BitmapImage for a discussion of all parameters but `options` and `frame`. `options` is an optional argument providing initial values for the above-listed GifFrame properties. Each property within option is itself optional.

Provide a `frame` to the constructor to create a clone of the provided frame. The new frame includes a copy of the provided frame's pixel data so that each can subsequently be modified without affecting each other.

<a name="GifFrame+getPalette"></a>

### *gifFrame*.getPalette()
Get a summary of the colors found within the frame. The return value is an object of the following form:

Property | Description
--- | ---
colors | An array of all the opaque colors found within the frame. Each color is given as an RGB number of the form 0xRRGGBB. The array is sorted by increasing number. Will be an empty array when the frame is completely transparent.
usesTransparency | boolean indicating whether there are any transparent pixels within the frame. A pixel is considered transparent if its alpha value is not 0xFF.
indexCount | The number of color indexes required to represent this palette of colors. It is equal to the number of opaque colors plus one if the frame includes transparency.

**Returns**: <code>object</code> - An object representing a color palette as described above.  
<a name="GifUtil.cloneFrames"></a>

### *GifUtil*.cloneFrames(frames)

| Param | Type | Description |
| --- | --- | --- |
| frames | [<code>Array.&lt;GifFrame&gt;</code>](#GifFrame) | An array of GifFrame instances to clone |

cloneFrames() clones provided frames. It's a utility method for cloning an entire array of frames at once.

**Returns**: [<code>Array.&lt;GifFrame&gt;</code>](#GifFrame) - An array of GifFrame clones of the provided frames.  
<a name="GifUtil.getColorInfo"></a>

### *GifUtil*.getColorInfo(frames, maxGlobalIndex)
**Throws**:

- [<code>GifError</code>](#GifError) When any frame requires more than 256 color indexes.


| Param | Type | Description |
| --- | --- | --- |
| frames | [<code>Array.&lt;GifFrame&gt;</code>](#GifFrame) | Frames to examine for color and transparency. |
| maxGlobalIndex | <code>number</code> | Maximum number of color indexes (including one for transparency) allowed among the returned compilation of colors. `colors` and `indexCount` are not returned if the number of color indexes required to accommodate  all frames exceeds this number. Returns `colors` and `indexCount` by default. |

getColorInfo() gets information about the colors used in the provided frames. The method is able to return an array of all colors found across all frames.

`maxGlobalIndex` controls whether the computation short-circuits to avoid doing work that the caller doesn't need. The method only returns `colors` and `indexCount` for the colors across all frames when the number of indexes required to store the colors and transparency in a GIF (which is the value of `indexCount`) is less than or equal to `maxGlobalIndex`. Such short-circuiting is useful when the caller just needs to determine whether any frame includes transparency.

**Returns**: <code>object</code> - Object containing at least `palettes` and `usesTransparency`. `palettes` is an array of all the palettes returned by GifFrame#getPalette(). `usesTransparency` indicates whether at least one frame uses transparency. If `maxGlobalIndex` is not exceeded, the object also contains `colors`, an array of all colors (RGB) found across all palettes, sorted by increasing value, and `indexCount` indicating the number of indexes required to store the colors and the transparency in a GIF.  
<a name="GifUtil.getMaxDimensions"></a>

### *GifUtil*.getMaxDimensions(frames)

| Param | Type | Description |
| --- | --- | --- |
| frames | [<code>Array.&lt;GifFrame&gt;</code>](#GifFrame) | Frames to measure for their aggregate maximum dimensions. |

getMaxDimensions() returns the pixel width and height required to accommodate all of the provided frames, according to the offsets and dimensions of each frame.

**Returns**: <code>object</code> - An object of the form {maxWidth, maxHeight} indicating the maximum width and height required to accommodate all frames.  
<a name="GifUtil.read"></a>

### *GifUtil*.read(source, An)

| Param | Type | Description |
| --- | --- | --- |
| source | <code>string</code> \| <code>Buffer</code> | Source to decode. When a string, it's the GIF filename to load and parse. When a Buffer, it's an encoded GIF to parse. |
| An | <code>object</code> | optional GIF decoder object implementing the `decode` method of class GifCodec. When provided, the method decodes the GIF using this decoder. When not provided, the method uses GifCodec. |

read() decodes an encoded GIF, whether provided as a filename or as a byte buffer.

**Returns**: <code>Promise</code> - A Promise that resolves to an instance of the Gif class, representing the decoded GIF.  
<a name="GifUtil.write"></a>

### *GifUtil*.write(path, frames, spec, encoder)

| Param | Type | Description |
| --- | --- | --- |
| path | <code>string</code> | Filename to write GIF out as. Will overwrite an existing file. |
| frames | [<code>Array.&lt;GifFrame&gt;</code>](#GifFrame) | Array of frames to be written into GIF. |
| spec | <code>object</code> | An optional object that may provide values for `loops` and `colorScope`, as defined for the Gif class. However, `colorSpace` may also take the value Gif.GlobalColorsPreferred (== 0) to indicate that the encoder should attempt to create only a global color table. `loop` defaults to 0, looping indefinitely, and `colorScope` defaults to Gif.GlobalColorsPreferred. |
| encoder | <code>object</code> | An optional GIF encoder object implementing the `encode` method of class GifCodec. When provided, the method encodes the GIF using this encoder. When not provided, the method uses GifCodec. |

write() encodes a GIF and saves it as a file.

**Returns**: <code>Promise</code> - A Promise that resolves to an instance of the Gif class, representing the encoded GIF.  
<a name="new_GifCodec_new"></a>

### new GifCodec(options)

| Param | Type | Description |
| --- | --- | --- |
| options | <code>object</code> | Optionally takes an objection whose only possible property is `transparentRGB`. Images are internally represented in RGBA format, where A is the alpha value of a pixel. When `transparentRGB` is provided, this RGB value is assigned to transparent pixels, which have alpha value 0x00. All other pixels have alpha value 0xFF. The RGB color of transparent pixels shouldn't matter for most applications. Defaults to 0x000000. |

GifCodec is a class that both encodes and decodes GIFs. It implements both the `encode()` method expected of an encoder and the `decode()` method expected of a decoder, and it wraps a modified version of the `omggif` GIF encoder/decoder package. GifCodec serves as this library's default encoder and decoder, but it's possible to wrap other GIF encoders and decoders for use by `gifwrap` as well.

Instances of this class are stateless and can be shared across multiple encodings and decodings.

Its constructor takes one option argument:

<a name="GifCodec+decodeGif"></a>

### *gifCodec*.decodeGif(buffer)
**Throws**:

- [<code>GifError</code>](#GifError) Error upon encountered an encoding-related problem with a GIF, so that the caller can distinguish between software errors and problems with GIFs.


| Param | Type | Description |
| --- | --- | --- |
| buffer | <code>Buffer</code> | Bytes of an encoded GIF to decode. |

Decodes a GIF from a Buffer to yield an instance of Gif. Transparent pixels of the GIF are given alpha values of 0x00, and opaque pixels are given alpha values of 0xFF.

**Returns**: <code>Promise</code> - A Promise that resolves to an instance of the Gif class, representing the encoded GIF.  
<a name="GifCodec+encodeGif"></a>

### *gifCodec*.encodeGif(frames, spec)
**Throws**:

- [<code>GifError</code>](#GifError) Error upon encountered an encoding-related problem with a GIF, so that the caller can distinguish between software errors and problems with GIFs.


| Param | Type | Description |
| --- | --- | --- |
| frames | [<code>Array.&lt;GifFrame&gt;</code>](#GifFrame) | Array of frames to encode |
| spec | <code>object</code> | An optional object that may provide values for `loops` and `colorScope`, as defined for the Gif class. However, `colorSpace` may also take the value Gif.GlobalColorsPreferred (== 0) to indicate that the encoder should attempt to create only a global color table. `loop` defaults to 0, looping indefinitely, and `colorScope` defaults to Gif.GlobalColorsPreferred. |

Encodes a GIF from provided frames. Any pixel not having an alpha value of 0xFF renders as transparent within the encoding, while all pixels of alpha value 0xFF are opaque.

**Returns**: <code>Promise</code> - A Promise that resolves to an instance of the Gif class, representing the encoded GIF.  
<a name="new_GifError_new"></a>

### new GifError(messageOrError)

| Param | Type |
| --- | --- |
| messageOrError | <code>string</code> \| <code>Error</code> | 

GifError is a class representing a GIF-related error


## LICENSE

MIT License

Copyright © 2017 Joseph T. Lapp

Copyright © 2013 Dean McNamee (default encoder/decoder omggif)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
