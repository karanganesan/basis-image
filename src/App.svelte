<script>
  import { onMount } from "svelte";

  let canvasElement;
  export let src = "";

  onMount(() => {
    setTimeout(() => {
      BASIS({
        onRuntimeInitialized: () => {
          var gl = canvasElement.getContext("webgl");
          astcSupported = !!gl.getExtension("WEBGL_compressed_texture_astc");
          etcSupported = !!gl.getExtension("WEBGL_compressed_texture_etc1");
          dxtSupported = !!gl.getExtension("WEBGL_compressed_texture_s3tc");
          pvrtcSupported =
            !!gl.getExtension("WEBGL_compressed_texture_pvrtc") ||
            !!gl.getExtension("WEBKIT_WEBGL_compressed_texture_pvrtc");
          bc7Supported = !!gl.getExtension("EXT_texture_compression_bptc");
          window.renderer = new Renderer(gl);
          if (
            !(astcSupported || etcSupported || dxtSupported || pvrtcSupported)
          ) {
            //      elem('nodxt').style.display = 'block';
          }
          loadArrayBuffer(src, dataLoaded);
        }
      }).then(module => (window.Module = module));
    }, 500);
  });
</script>

<svelte:options tag="basis-img" />
<canvas id="basis-canvas" bind:this={canvasElement} />
<svelte:head>
  <script
    src="https://raw.githack.com/BinomialLLC/basis_universal/master/webgl/texture/renderer.js">

  </script>
  <script
    src="https://raw.githack.com/BinomialLLC/basis_universal/master/webgl/texture/dxt-to-rgb565.js">

  </script>
  <script
    src="https://raw.githack.com/BinomialLLC/basis_universal/master/webgl/transcoder/build/basis_transcoder.js">

  </script>
  <script>
    function log(s) {
      console.log(s);
    }

    function logTime(desc, t) {
      log(t + "ms " + desc);
    }

    function loadArrayBuffer(uri, callback) {
      log("Loading " + uri + "...");
      var xhr = new XMLHttpRequest();
      xhr.responseType = "arraybuffer";
      xhr.open("GET", uri, true);
      xhr.onreadystatechange = function(e) {
        if (xhr.readyState == 4 && xhr.status == 200) {
          callback(xhr.response);
        }
      };
      xhr.send(null);
    }

    // ASTC format, from:
    // https://www.khronos.org/registry/webgl/extensions/WEBGL_compressed_texture_astc/
    COMPRESSED_RGBA_ASTC_4x4_KHR = 0x93b0;

    // DXT formats, from:
    // http://www.khronos.org/registry/webgl/extensions/WEBGL_compressed_texture_s3tc/
    COMPRESSED_RGB_S3TC_DXT1_EXT = 0x83f0;
    COMPRESSED_RGBA_S3TC_DXT1_EXT = 0x83f1;
    COMPRESSED_RGBA_S3TC_DXT3_EXT = 0x83f2;
    COMPRESSED_RGBA_S3TC_DXT5_EXT = 0x83f3;

    // BC7 format, from:
    // https://www.khronos.org/registry/webgl/extensions/EXT_texture_compression_bptc/
    COMPRESSED_RGBA_BPTC_UNORM = 0x8e8c;

    // ETC format, from:
    // https://www.khronos.org/registry/webgl/extensions/WEBGL_compressed_texture_etc1/
    COMPRESSED_RGB_ETC1_WEBGL = 0x8d64;

    // PVRTC format, from:
    // https://www.khronos.org/registry/webgl/extensions/WEBGL_compressed_texture_pvrtc/
    COMPRESSED_RGB_PVRTC_4BPPV1_IMG = 0x8c00;
    COMPRESSED_RGBA_PVRTC_4BPPV1_IMG = 0x8c02;

    BASIS_FORMAT = {
      cTFETC1: 0,
      cTFETC2: 1,
      cTFBC1: 2,
      cTFBC3: 3,
      cTFBC4: 4,
      cTFBC5: 5,
      cTFBC7: 6,
      cTFPVRTC1_4_RGB: 8,
      cTFPVRTC1_4_RGBA: 9,
      cTFASTC_4x4: 10,
      cTFATC_RGB: 11,
      cTFATC_RGBA_INTERPOLATED_ALPHA: 12,
      cTFRGBA32: 13,
      cTFRGB565: 14,
      cTFBGR565: 15,
      cTFRGBA4444: 16
    };

    BASIS_FORMAT_NAMES = {};
    for (var name in BASIS_FORMAT) {
      BASIS_FORMAT_NAMES[BASIS_FORMAT[name]] = name;
    }

    DXT_FORMAT_MAP = {};
    DXT_FORMAT_MAP[BASIS_FORMAT.cTFBC1] = COMPRESSED_RGB_S3TC_DXT1_EXT;
    DXT_FORMAT_MAP[BASIS_FORMAT.cTFBC3] = COMPRESSED_RGBA_S3TC_DXT5_EXT;
    DXT_FORMAT_MAP[BASIS_FORMAT.cTFBC7] = COMPRESSED_RGBA_BPTC_UNORM;

    var astcSupported = false;
    var etcSupported = false;
    var dxtSupported = false;
    var bc7Supported = false;
    var pvrtcSupported = false;
    var drawMode = 0;

    var tex,
      width,
      height,
      images,
      levels,
      have_alpha,
      alignedWidth,
      alignedHeight,
      format,
      displayWidth,
      displayHeight;

    function redraw() {
      if (!width) return;

      renderer.drawTexture(tex, displayWidth, displayHeight, drawMode);
    }

    function dataLoaded(data) {
      log("Done loading .basis file, decoded header:");

      const { BasisFile, initializeBasis } = Module;
      initializeBasis();

      const startTime = performance.now();

      const basisFile = new BasisFile(new Uint8Array(data));

      width = basisFile.getImageWidth(0, 0);
      height = basisFile.getImageHeight(0, 0);
      images = basisFile.getNumImages();
      levels = basisFile.getNumLevels(0);
      has_alpha = basisFile.getHasAlpha();

      if (!width || !height || !images || !levels) {
        console.warn("Invalid .basis file");
        basisFile.close();
        basisFile.delete();
        return;
      }

      // Note: If the file is UASTC, the preferred formats are ASTC/BC7.
      // If the file is ETC1S and doesn't have alpha, the preferred formats are ETC1 and BC1. For alpha, the preferred formats are ETC2, BC3 or BC7.

      var formatString = "UNKNOWN";
      if (astcSupported) {
        formatString = "ASTC";
        format = BASIS_FORMAT.cTFASTC_4x4;
      } else if (bc7Supported) {
        formatString = "BC7";
        format = BASIS_FORMAT.cTFBC7;
      } else if (dxtSupported) {
        if (has_alpha) {
          formatString = "BC3";
          format = BASIS_FORMAT.cTFBC3;
        } else {
          formatString = "BC1";
          format = BASIS_FORMAT.cTFBC1;
        }
      } else if (pvrtcSupported) {
        if (has_alpha) {
          formatString = "PVRTC1_RGBA";
          format = BASIS_FORMAT.cTFPVRTC1_4_RGBA;
        } else {
          formatString = "PVRTC1_RGB";
          format = BASIS_FORMAT.cTFPVRTC1_4_RGB;
        }

        if ((width & (width - 1)) != 0 || (height & (height - 1)) != 0) {
          log("ERROR: PVRTC1 requires square power of 2 textures");
        }
        if (width != height) {
          log("ERROR: PVRTC1 requires square power of 2 textures");
        }
      } else if (etcSupported) {
        formatString = "ETC1";
        format = BASIS_FORMAT.cTFETC1;
      } else {
        formatString = "RGB565";
        format = BASIS_FORMAT.cTFRGB565;
        log("Decoding .basis data to 565");
      }

      if (!basisFile.startTranscoding()) {
        log("startTranscoding failed");
        console.warn("startTranscoding failed");
        basisFile.close();
        basisFile.delete();
        return;
      }

      const dstSize = basisFile.getImageTranscodedSizeInBytes(0, 0, format);
      const dst = new Uint8Array(dstSize);

      log(dstSize);

      //  if (!basisFile.transcodeImage(dst, 0, 0, format, 1, 0)) {
      if (!basisFile.transcodeImage(dst, 0, 0, format, 0, 0)) {
        log("basisFile.transcodeImage failed");
        console.warn("transcodeImage failed");
        basisFile.close();
        basisFile.delete();

        return;
      }

      const elapsed = performance.now() - startTime;

      basisFile.close();
      basisFile.delete();

      log("width: " + width);
      log("height: " + height);
      log("images: " + images);
      log("first image mipmap levels: " + levels);
      log("has_alpha: " + has_alpha);
      logTime("transcoding time", elapsed.toFixed(2));

      alignedWidth = (width + 3) & ~3;
      alignedHeight = (height + 3) & ~3;

      displayWidth = alignedWidth;
      displayHeight = alignedHeight;

      var canvas = document
        .querySelector("basis-img")
        .shadowRoot.querySelector("#basis-canvas");

      canvas.width = alignedWidth;
      canvas.height = alignedHeight;

      if (format === BASIS_FORMAT.cTFASTC_4x4) {
        tex = renderer.createCompressedTexture(
          dst,
          alignedWidth,
          alignedHeight,
          COMPRESSED_RGBA_ASTC_4x4_KHR
        );
      } else if (
        format === BASIS_FORMAT.cTFBC3 ||
        format === BASIS_FORMAT.cTFBC1 ||
        format == BASIS_FORMAT.cTFBC7
      ) {
        tex = renderer.createCompressedTexture(
          dst,
          alignedWidth,
          alignedHeight,
          DXT_FORMAT_MAP[format]
        );
      } else if (format === BASIS_FORMAT.cTFETC1) {
        tex = renderer.createCompressedTexture(
          dst,
          alignedWidth,
          alignedHeight,
          COMPRESSED_RGB_ETC1_WEBGL
        );
      } else if (format === BASIS_FORMAT.cTFPVRTC1_4_RGB) {
        tex = renderer.createCompressedTexture(
          dst,
          alignedWidth,
          alignedHeight,
          COMPRESSED_RGB_PVRTC_4BPPV1_IMG
        );
      } else if (format === BASIS_FORMAT.cTFPVRTC1_4_RGBA) {
        tex = renderer.createCompressedTexture(
          dst,
          alignedWidth,
          alignedHeight,
          COMPRESSED_RGBA_PVRTC_4BPPV1_IMG
        );
      } else {
        canvas.width = width;
        canvas.height = height;
        displayWidth = width;
        displayHeight = height;

        // Create 565 texture.
        var dstTex = new Uint16Array(width * height);

        // Convert the array of bytes to an array of uint16's.
        var pix = 0;
        for (var y = 0; y < height; y++)
          for (var x = 0; x < width; x++, pix++)
            dstTex[pix] = dst[2 * pix + 0] | (dst[2 * pix + 1] << 8);

        tex = renderer.createRgb565Texture(dstTex, width, height);
      }

      redraw();
    }
  </script>
</svelte:head>
