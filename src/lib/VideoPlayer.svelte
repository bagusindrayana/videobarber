<script lang="ts">
  import { createEventDispatcher, onMount } from "svelte";

  export let videoSrc: string;
  export let currentTime: number = 0;
  export let startTime: number = 0;
  export let endTime: number = 0;

  let video: HTMLVideoElement;
  const dispatch = createEventDispatcher();

  onMount(() => {
    if (video) {
      video.addEventListener("loadedmetadata", () => {
        dispatch("loadedmetadata", { duration: video.duration });
      });
    }
  });

  function onTimeUpdate() {
    currentTime = video.currentTime;
    dispatch("timeupdate", { currentTime: video.currentTime });

    if (video.currentTime >= endTime) {
      video.pause();
      video.currentTime = startTime;
    }
  }

  function onPlay() {
    if (video.currentTime < startTime || video.currentTime >= endTime) {
      video.currentTime = startTime;
    }
  }

  function onSeeked() {
    if (video.currentTime < startTime) {
      video.currentTime = startTime;
    } else if (video.currentTime > endTime) {
      video.currentTime = startTime;
    }
  }

  let isPlaying = false;
  let isMuted = false;
  let volume = 1;
  let duration = 0;

  onMount(() => {
    video.addEventListener("timeupdate", () => {
      currentTime = video.currentTime;
    });
    video.addEventListener("loadedmetadata", () => {
      duration = video.duration;
    });
  });

  function togglePlay() {
    if (isPlaying) {
      video.pause();
    } else {
      video.play();
    }
    isPlaying = !isPlaying;
  }

  function toggleMute() {
    video.muted = !isMuted;
    isMuted = !isMuted;
  }

  function handleVolumeChange(event: Event) {
    const volumeValue = parseFloat((event.target as HTMLInputElement).value);
    video.volume = volumeValue;
    volume = volumeValue;
    isMuted = volumeValue === 0;
  }

  function handleTimelineChange(event: Event) {
    const timeValue = parseFloat((event.target as HTMLInputElement).value);
    video.currentTime = timeValue;
    currentTime = timeValue;
  }

  function formatTime(time: number) {
    const minutes = Math.floor(time / 60);
    const seconds = Math.floor(time % 60);
    return `${minutes}:${seconds.toString().padStart(2, "0")}`;
  }

  export function updateVideoTime(time: number) {
    if (video) {
      video.currentTime = time;
    }
  }
</script>

<div class="relative w-full bg-gray-900 rounded-lg overflow-hidden shadow-lg" style="max-height: 50vh;">
  <video
    src={videoSrc}
    bind:this={video}
    on:timeupdate={onTimeUpdate}
    on:play={onPlay}
    class="w-full h-full object-contain"
    style="max-height: 50vh;"
  >
    Your browser does not support the video tag.
  </video>

  <div class="absolute bottom-0 left-0 right-0 bg-gradient-to-t from-black/80 to-transparent p-2">
    <div class="flex items-center justify-between text-white gap-2">
      <button
        class="text-white hover:text-blue-400 p-1 rounded-full focus:outline-none flex-shrink-0"
        on:click={togglePlay}
      >
        {#if isPlaying}
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24">
            <g fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2">
              <rect width="4" height="16" x="14" y="4" rx="1" />
              <rect width="4" height="16" x="6" y="4" rx="1" />
            </g>
          </svg>
        {:else}
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24">
            <path fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="m6 3l14 9l-14 9z" />
          </svg>
        {/if}
      </button>

      <span class="text-xs flex-shrink-0">{formatTime(currentTime)}</span>

      <input
        type="range"
        min={startTime}
        max={endTime}
        step="0.1"
        value={currentTime}
        on:input={handleTimelineChange}
        class="flex-1 appearance-none bg-gray-600 h-1 rounded-full focus:outline-none focus:ring-2 focus:ring-blue-400 min-w-0"
      />

      <span class="text-xs flex-shrink-0">{formatTime(endTime)}</span>

      <button
        class="text-white hover:text-blue-400 p-1 rounded-full focus:outline-none flex-shrink-0"
        on:click={toggleMute}
      >
        {#if isMuted}
          <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24">
            <path fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298zM22 9l-6 6m0-6l6 6" />
          </svg>
        {:else}
          <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24">
            <path fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298zM16 9a5 5 0 0 1 0 6" />
          </svg>
        {/if}
      </button>
    </div>
  </div>
</div>
