<script lang="ts">
    import { onMount, afterUpdate } from 'svelte';
  
    export let videoSrc: string;
    export let previewTime: number = 0;
    export let visible: boolean = false;
  
    let canvas: HTMLCanvasElement;
    let video: HTMLVideoElement;

    $: videoSrc && updatePreview();
  
    onMount(() => {
      video = document.createElement('video');
      video.src = videoSrc;
      video.preload = 'metadata';
    });
  
    afterUpdate(() => {
      if (visible && video && canvas) {
        updatePreview();
      }
    });
  
    async function updatePreview() {
      if(video == null){
        return;
      }
      video.src = videoSrc;
      video.currentTime = previewTime;
      await new Promise((resolve) => {
        video.onseeked = resolve;
      });
      const ctx = canvas.getContext('2d');
      if(ctx){
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      }
    }
  </script>
  
  <canvas
    bind:this={canvas}
    width="160"
    height="90"
    class="border border-gray-300 rounded shadow-lg pointer-events-none"
    class:hidden={!visible}
  ></canvas>
  
  <style>
    .hidden {
      display: none;
    }
  </style>