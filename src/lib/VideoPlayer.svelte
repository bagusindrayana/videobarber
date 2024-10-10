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

<div class="relative w-full md:w-1/2 mx-auto bg-gray-500">
  <video
    src={videoSrc}
    bind:this={video}
    on:timeupdate={onTimeUpdate}
    on:play={onPlay}
    on:seeked={onSeeked}
    class=" w-full rounded-lg shadow-lg aspect-video " style="max-height: 50vh;"
  >
    Your browser does not support the video tag.
  </video>
  <div
    class="absolute bottom-0 left-0 right-0 bg-gradient-to-t from-black to-transparent p-1 md:p-4"
  >
    <div class="flex items-center justify-between text-white">
      <div class="flex items-center flex-1">
        <button
          class="text-white hover:text-primary p-0 md:p-2 rounded-full focus:outline-none focus:ring-2 focus:ring-primary"
          on:click={togglePlay}
        >
          {#if isPlaying}
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              ><g
                fill="none"
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                ><rect width="4" height="16" x="14" y="4" rx="1" /><rect
                  width="4"
                  height="16"
                  x="6"
                  y="4"
                  rx="1"
                /></g
              ></svg
            >
          {:else}
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              ><path
                fill="none"
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="m6 3l14 9l-14 9z"
              /></svg
            >
          {/if}
        </button>
        <div class="flex items-center space-x-2 flex-1 mx-4">
          <span class="text-sm">{formatTime(currentTime)}</span>
          <input
            type="range"
            min={startTime}
            max={endTime}
            step="1"
            value={currentTime}
            on:input={handleTimelineChange}
            class="flex-1 appearance-none bg-gray-600 h-1 rounded-full focus:outline-none focus:ring-2 focus:ring-primary"
          />
          <span class="text-sm">{formatTime(endTime)}</span>
        </div>
      </div>
      <div class="hidden md:flex items-center space-x-2 ">
        <button
          class="text-white hover:text-primary p-0 md:p-2 rounded-full focus:outline-none focus:ring-2 focus:ring-primary"
          on:click={toggleMute}
        >
          {#if isMuted}
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              ><path
                fill="none"
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298zM22 9l-6 6m0-6l6 6"
              /></svg
            >
          {:else}
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              ><path
                fill="none"
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M11 4.702a.705.705 0 0 0-1.203-.498L6.413 7.587A1.4 1.4 0 0 1 5.416 8H3a1 1 0 0 0-1 1v6a1 1 0 0 0 1 1h2.416a1.4 1.4 0 0 1 .997.413l3.383 3.384A.705.705 0 0 0 11 19.298zM16 9a5 5 0 0 1 0 6m3.364 3.364a9 9 0 0 0 0-12.728"
              /></svg
            >
          {/if}
        </button>
        <input
          type="range"
          min="0"
          max="1"
          step="0.1"
          value={volume}
          on:input={handleVolumeChange}
          class="w-24 appearance-none bg-gray-600 h-1 rounded-full focus:outline-none focus:ring-2 focus:ring-primary"
        />
      </div>
    </div>
  </div>
</div>
