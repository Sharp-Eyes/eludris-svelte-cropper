<script lang="ts">
  import { onMount } from 'svelte';

  const BOX_HEIGHT = 150;
  const BOX_WIDTH = 900;

  let image: HTMLImageElement;
  let canvas: HTMLCanvasElement;

  let dragging = false;
  let imageX = 0;
  let imageY = 0;
  let lastX = 0;
  let lastY = 0;
  let xBoundary = 0;
  let yBoundary = 0;
  let scale = 1;

  onMount(() => {
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
    image.style.transform = `translate(${imageX*scale}px, ${imageY*scale}px) scale(${scale})`;
  };

  const scaleImage = () => {
    xBoundary = (image.width - BOX_WIDTH/scale) / 2;
    yBoundary = (image.height - BOX_HEIGHT/scale) / 2;
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
      console.log(e.x, lastX)
      // Divide move distance by scale to slow down movement when zoomed.
      imageX += (e.x - lastX) / scale;
      imageY += (e.y - lastY) / scale;
      updateImagePosition();
      lastX = e.x;
      lastY = e.y;
    }
  };

  const cropImage = () => {
    const ctx = canvas.getContext('2d')!;

    ctx.drawImage(
      image,
      (image.width - BOX_WIDTH/scale)/2 - imageX,
      (image.height - BOX_HEIGHT/scale)/2 - imageY,
      BOX_WIDTH / scale,
      BOX_HEIGHT / scale,
      0,
      0,
      BOX_WIDTH,
      BOX_HEIGHT,
    );
  }

</script>

<svelte:body on:mouseup={stopDrag} on:mousemove={mouseMove} />

<div id="cropper">
  <img id="image" src="das_ding.png" alt="editable" bind:this={image} />
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div id="overlay" on:mousedown={startDrag} on:wheel={onWheel}>
    <div id="overlay-cutout" />
  </div>
</div>
<input type="range" min="0.5" max="5" step="0.0001" bind:value={scale} on:input={scaleImage} />
<button on:click={cropImage}>Crop</button>
<canvas width={BOX_WIDTH} height={BOX_HEIGHT} bind:this={canvas}></canvas>

<style>
  #cropper {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;

    width: 1000px;
    height: 400px;
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
    height: 150px;
    width: 900px;
    border: 3px solid white;
    border-radius: 10px;
    box-shadow: 0 0 200px 200px #000c;
  }
</style>
