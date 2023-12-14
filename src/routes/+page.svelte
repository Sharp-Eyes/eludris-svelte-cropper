<script lang="ts">
  import { onMount } from 'svelte';
  import { parseGIF, decompressFrames } from 'gifuct-js';
  import GIF from 'gif.js';

  const CONTAINER_HEIGHT = 400;
  const CONTAINER_WIDTH = 1000;
  const BOX_HEIGHT = 150;
  const BOX_WIDTH = 900;

  let image: HTMLImageElement;
  let canvas: HTMLCanvasElement;
  let tempCanvas: HTMLCanvasElement;

  let dragging = false;
  let imageX = 0;
  let imageY = 0;
  let lastX = 0;
  let lastY = 0;
  let xBoundary = 0;
  let yBoundary = 0;
  let scale = 1;
  let preview = '';

  onMount(() => {
    tempCanvas = document.createElement('canvas');

    xBoundary = (image.width - BOX_WIDTH) / 2;
    yBoundary = (image.height - BOX_HEIGHT) / 2;
    updateImagePosition();
  });

  $: {
    if (image) {
    }
  }

  const updateImagePosition = () => {
    if (image.width * scale > BOX_WIDTH) {
      imageX = Math.max(Math.min(imageX, xBoundary), -xBoundary);
    } else {
      imageX = 0;
    }
    if (image.height * scale > BOX_HEIGHT) {
      imageY = Math.max(Math.min(imageY, yBoundary), -yBoundary);
    } else {
      imageY = 0;
    }
    image.style.transform = `translate(${imageX * scale}px, ${imageY * scale}px) scale(${scale})`;
  };

  const scaleImage = () => {
    xBoundary = (image.width - BOX_WIDTH / scale) / 2;
    yBoundary = (image.height - BOX_HEIGHT / scale) / 2;
    updateImagePosition();
  };

  const startDrag = (e: MouseEvent) => {
    lastX = e.x;
    lastY = e.y;
    dragging = true;
  };

  const stopDrag = () => {
    dragging = false;
  };

  const onWheel = (e: WheelEvent) => {
    if (e.deltaY < 0) {
      scale = Math.min(scale + 0.1, 5);
      scaleImage();
    } else {
      scale = Math.max(scale - 0.1, 0.5);
      scaleImage();
    }
  };

  const mouseMove = (e: MouseEvent) => {
    if (dragging) {
      // Divide move distance by scale to slow down movement when zoomed.
      imageX += (e.x - lastX) / scale;
      imageY += (e.y - lastY) / scale;
      updateImagePosition();
      lastX = e.x;
      lastY = e.y;
    }
  };

  async function doCrop() {
    let imageResponse = await fetch(image.src);
    let contentType = imageResponse.headers.get('content-type');

    if (contentType == 'image/gif') {
      await cropGif(imageResponse);
    } else {
      cropImage();
    }
  }

  async function cropGif(resp: Response) {
    // Actual destination canvas ctx, same size as the crop box.
    const ctx = canvas.getContext('2d')!;
    // Intermediary canvas ctx, same size as the gif.
    const tempCtx = tempCanvas.getContext('2d')!;

    const GifBuilder = new GIF({
      workers: 2,
      quality: 10,
      workerScript: '/src/gif.worker.js',
      transparent: '#0000'
    });

    // Parse gif and prepare temp canvas
    let inputGif = parseGIF(await resp.arrayBuffer());
    let frames = decompressFrames(inputGif, true);

    tempCanvas.width = image.width;
    tempCanvas.height = image.height;
    let imageData = tempCtx.createImageData(image.width, image.height);

    // Loop over each frame, paste it on the temp canvas, and crop it onto the main canvas.
    frames.forEach((frame) => {
      imageData.data.set(frame.patch);
      tempCtx.putImageData(imageData, 0, 0);

      ctx.reset();
      // Fill with transparency in case the crop is smaller than the crop box size.
      ctx.fillStyle = 'transparent';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      ctx.drawImage(
        tempCanvas,
        (image.width - BOX_WIDTH / scale) / 2 - imageX,
        (image.height - BOX_HEIGHT / scale) / 2 - imageY,
        BOX_WIDTH / scale,
        BOX_HEIGHT / scale,
        0,
        0,
        BOX_WIDTH,
        BOX_HEIGHT
      );

      GifBuilder.addFrame(canvas, { copy: true, delay: frame.delay });
    });

    GifBuilder.on('finished', function (blob) {
      // For testing, just open a popup window with the new cropped gif.
      preview = URL.createObjectURL(blob);
    });

    GifBuilder.render();
  }

  const cropImage = () => {
    const ctx = canvas.getContext('2d')!;

    ctx.drawImage(
      image,
      (image.width - BOX_WIDTH / scale) / 2 - imageX,
      (image.height - BOX_HEIGHT / scale) / 2 - imageY,
      BOX_WIDTH / scale,
      BOX_HEIGHT / scale,
      0,
      0,
      BOX_WIDTH,
      BOX_HEIGHT
    );

    preview = canvas.toDataURL();
  };
</script>

<svelte:body on:mouseup={stopDrag} on:mousemove={mouseMove} />

<div style:height="{CONTAINER_HEIGHT}px" style:width="{CONTAINER_WIDTH}px" id="cropper">
  <img id="image" src="horse-eating.gif" alt="editable" bind:this={image} />
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div id="overlay" on:mousedown={startDrag} on:wheel={onWheel}>
    <div style:height="{BOX_HEIGHT}px" style:width="{BOX_WIDTH}px" id="overlay-cutout" />
  </div>
</div>
<input type="range" min="0.5" max="5" step="0.0001" bind:value={scale} on:input={scaleImage} />
<button on:click={doCrop}>Crop</button>
<canvas width={BOX_WIDTH} height={BOX_HEIGHT} bind:this={canvas} />
<img src={preview} alt="result" id="preview" />

<style>
  #cropper {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
  }

  #image {
    position: absolute;
    z-index: -1;
  }

  #overlay {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100%;
    width: 100%;
    cursor: move;
    touch-action: none;
  }

  #overlay-cutout {
    border: 3px solid white;
    border-radius: 10px;
    box-shadow: 0 0 200px 200px #000c;
  }

  canvas {
    display: none;
  }
</style>
