<script lang="ts">
  import { createEventDispatcher, onMount } from "svelte";
  import FramePreview from "./FramePreview.svelte";

  export let videoDuration: number;
  export let startTime: number = 0;
  export let endTime: number = videoDuration;
  export let videoSrc: string;

  const dispatch = createEventDispatcher();

  let previewTime: number = 0;
  let showPreview: boolean = false;
  let previewPosition: number = 0;
  let timelineRect: DOMRect;
  let isDragging: any;
  let timelineRef: HTMLDivElement;

  function onStartChange(event: Event) {
    startTime = parseFloat((event.target as HTMLInputElement).value);
    if (startTime > endTime) {
      startTime = endTime;
    }
    if (startTime < 0) {
      startTime = 0;
    }
    dispatch("trimchange", { startTime, endTime });
    dispatch("previewchange", { time: startTime });
  }

  function onEndChange(event: Event) {
    endTime = parseFloat((event.target as HTMLInputElement).value);
    if (endTime < startTime) {
      endTime = startTime;
    }
    if (endTime > videoDuration) {
      endTime = videoDuration;
    }
    dispatch("trimchange", { startTime, endTime });
    dispatch("previewchange", { time: endTime });
  }

  function onTimelineClick(event: MouseEvent) {
    const clickPosition =
      (event.clientX - timelineRect.left) / timelineRect.width;
    const clickTime = clickPosition * videoDuration;

    if (Math.abs(clickTime - startTime) < Math.abs(clickTime - endTime)) {
      startTime = clickTime;
      if (startTime > endTime) {
        startTime = endTime;
      }
    } else {
      endTime = clickTime;
      if (endTime < startTime) {
        endTime = startTime;
      }
    }
    if (startTime < 0) {
      startTime = 0;
    }
    if (endTime > videoDuration) {
      endTime = videoDuration;
    }

    dispatch("trimchange", { startTime, endTime });
    dispatch("previewchange", { time: clickTime });
  }

  function onTimelineHover(event: MouseEvent) {
    const hoverPosition =
      (event.clientX - timelineRect.left) / timelineRect.width;
    previewTime = hoverPosition * videoDuration;
    previewPosition = event.clientX - timelineRect.left;
    showPreview = true;
  }

  function onTimelineLeave() {
    showPreview = false;
  }

  function onTimelineMount(node: HTMLDivElement) {
    timelineRect = node.getBoundingClientRect();
    node.addEventListener("mousemove", onTimelineHover);
    node.addEventListener("mouseleave", onTimelineLeave);
    return {
      destroy() {
        node.removeEventListener("mousemove", onTimelineHover);
        node.removeEventListener("mouseleave", onTimelineLeave);
      },
    };
  }

  $: progressPercentage = ((endTime - startTime) / videoDuration) * 100;
  $: startPercentage = (startTime / videoDuration) * 100;

  const handleTimelineClick = (e: any) => {
    const clickPosition =
      (e.clientX - timelineRect.left) / timelineRect.width;
    const clickTime = clickPosition * videoDuration;
    dispatch("previewchange", { time: clickTime });
  };

  const handleTrimHandleDrag = (e: any, handle: "start" | "end") => {
    isDragging = handle;
  };

  const handleTrimHandleRelease = () => {
    isDragging = null;
  };

  onMount(() => {
    timelineRef = document.getElementById("timeline") as HTMLDivElement;
    const handleMouseMove = (e: MouseEvent) => {
      if (!isDragging) return;
      const rect = timelineRef.getBoundingClientRect();
      const x = Math.max(0, Math.min(e.clientX - rect.left, rect.width));
      const newTime = (x / rect.width) * videoDuration;

      if (isDragging === "start") {
        startTime = Math.min(newTime, endTime - 0.1);
      } else {
        endTime = Math.max(newTime, startTime + 0.1);
      }
      dispatch("trimchange", { startTime, endTime });
    dispatch("previewchange", { time: endTime });
    };

    const handleMouseUp = () => {
      isDragging = null;
    };

    document.addEventListener("mousemove", handleMouseMove);
    document.addEventListener("mouseup", handleMouseUp);
  });
</script>

<div class="w-full max-w-3xl mx-auto mt-4">
  <div class="flex justify-between mb-2">
    <span>Start: {startTime.toFixed(2)}s</span>
    <span>End: {endTime.toFixed(2)}s</span>
  </div>
  <!-- <div class="relative h-8" on:click={onTimelineClick} use:onTimelineMount>
    <div class="absolute w-full h-2 bg-gray-300 rounded-full mt-3"></div>
    <div
      class="absolute h-2 bg-blue-500 rounded-full mt-3"
      style="width: {progressPercentage}%; left: {startPercentage}%"
    ></div>
    <input
      type="range"
      min="0"
      max={videoDuration}
      step="0.01"
      bind:value={startTime}
      on:input={onStartChange}
      class="absolute w-full"
    />
    <input
      type="range"
      min="0"
      max={videoDuration}
      step="0.01"
      bind:value={endTime}
      on:input={onEndChange}
      class="absolute w-full"
    />
    <div
      class="absolute bottom-full"
      style="left: {previewPosition}px; transform: translateX(-50%);"
    >
      <FramePreview {videoSrc} {previewTime} visible={showPreview} />
    </div>
  </div> -->

  <div class="space-y-2">
    <label>Timeline</label>
    <div
      id="timeline"
      class="relative h-8 bg-gray-200 rounded cursor-pointer"
      on:click={(e) => handleTimelineClick(e)}
      use:onTimelineMount
    >
    
      <div
        class="absolute top-0 bottom-0 bg-blue-500 opacity-50"
        style="left: {(startTime / videoDuration) * 100}%; right: {100 -
          (endTime / videoDuration) * 100}%;"
      />
      <div
        class="absolute top-0 bottom-0 w-0.5 bg-red-500"
        style="left: {(previewTime / videoDuration) * 100}%;"
      />
      <div
        class="absolute top-0 bottom-0 w-1 bg-blue-700 cursor-ew-resize"
        style="left: {(startTime / videoDuration) * 100}%;"
        on:mousedown={(e) => handleTrimHandleDrag(e, "start")}
      />
      <div
        class="absolute top-0 bottom-0 w-1 bg-blue-700 cursor-ew-resize"
        style="left: {(endTime / videoDuration) * 100}%"
        on:mousedown={(e) => handleTrimHandleDrag(e, "end")}
      />
      
      <div
      class="absolute bottom-full"
      style="left: {previewPosition}px; transform: translateX(-50%);"
    >
      <FramePreview {videoSrc} {previewTime} visible={showPreview} />
    </div>
    </div>
    <div class="flex justify-between text-sm text-muted-foreground">
      <!-- <span>{formatTime(startTime)}</span>
        <span>{formatTime(endTime)}</span> -->
    </div>
  </div>
</div>

<style>
  input[type="range"] {
    -webkit-appearance: none;
    width: 100%;
    background: transparent;
    pointer-events: none;
  }

  input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    height: 20px;
    width: 20px;
    border-radius: 50%;
    background: #4299e1;
    cursor: pointer;
    margin-top: -8px;
    pointer-events: auto;
  }

  input[type="range"]::-moz-range-thumb {
    height: 20px;
    width: 20px;
    border-radius: 50%;
    background: #4299e1;
    cursor: pointer;
    pointer-events: auto;
  }

  input[type="range"]::-webkit-slider-runnable-track {
    width: 100%;
    height: 4px;
    background: transparent;
    border-radius: 2px;
  }

  input[type="range"]::-moz-range-track {
    width: 100%;
    height: 4px;
    background: transparent;
    border-radius: 2px;
  }
</style>
