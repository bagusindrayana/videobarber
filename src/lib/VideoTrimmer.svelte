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
  let timelineRect: DOMRect | null = null;
  let isDragging: "start" | "end" | null = null;
  let timelineRef: HTMLDivElement;

  function getTimelineRect(): DOMRect {
    if (timelineRef) {
      return timelineRef.getBoundingClientRect();
    }
    return new DOMRect();
  }

  function onTimelineClick(event: MouseEvent) {
    const rect = getTimelineRect();
    const clickPosition = (event.clientX - rect.left) / rect.width;
    const clickTime = Math.max(0, Math.min(clickPosition * videoDuration, videoDuration));

    if (Math.abs(clickTime - startTime) < Math.abs(clickTime - endTime)) {
      startTime = Math.min(clickTime, endTime - 0.1);
    } else {
      endTime = Math.max(clickTime, startTime + 0.1);
    }

    dispatch("trimchange", { startTime, endTime });
    dispatch("previewchange", { time: clickTime });
  }

  function onTimelineHover(event: MouseEvent) {
    const rect = getTimelineRect();
    const hoverPosition = (event.clientX - rect.left) / rect.width;
    previewTime = Math.max(0, Math.min(hoverPosition * videoDuration, videoDuration));
    previewPosition = event.clientX - rect.left;
    showPreview = true;
  }

  function onTimelineLeave() {
    showPreview = false;
  }

  function onTimelineMount(node: HTMLDivElement) {
    timelineRef = node;
    timelineRect = node.getBoundingClientRect();
    
    const updateRect = () => {
      timelineRect = node.getBoundingClientRect();
    };
    
    window.addEventListener("resize", updateRect);
    node.addEventListener("mousemove", onTimelineHover);
    node.addEventListener("mouseleave", onTimelineLeave);
    
    return {
      destroy() {
        window.removeEventListener("resize", updateRect);
        node.removeEventListener("mousemove", onTimelineHover);
        node.removeEventListener("mouseleave", onTimelineLeave);
      },
    };
  }

  const handleTrimHandleDrag = (e: Event, handle: "start" | "end") => {
    e.stopPropagation();
    isDragging = handle;
  };

  onMount(() => {
    const handleMouseMove = (e: MouseEvent) => {
      if (!isDragging || !timelineRef) return;
      const rect = timelineRef.getBoundingClientRect();
      const x = Math.max(0, Math.min(e.clientX - rect.left, rect.width));
      const newTime = (x / rect.width) * videoDuration;

      if (isDragging === "start") {
        startTime = Math.min(newTime, endTime - 0.1);
      } else {
        endTime = Math.max(newTime, startTime + 0.1);
      }
      dispatch("trimchange", { startTime, endTime });
      dispatch("previewchange", { time: newTime });
    };

    const handleTouchMove = (e: TouchEvent) => {
      if (!isDragging || !timelineRef) return;
      const rect = timelineRef.getBoundingClientRect();
      const touch = e.targetTouches[0];
      const x = Math.max(0, Math.min(touch.clientX - rect.left, rect.width));
      const newTime = (x / rect.width) * videoDuration;

      if (isDragging === "start") {
        startTime = Math.min(newTime, endTime - 0.1);
      } else {
        endTime = Math.max(newTime, startTime + 0.1);
      }
      dispatch("trimchange", { startTime, endTime });
      dispatch("previewchange", { time: newTime });
    };

    const handleMouseUp = () => {
      isDragging = null;
    };

    document.addEventListener("mousemove", handleMouseMove);
    document.addEventListener("touchmove", handleTouchMove);
    document.addEventListener("mouseup", handleMouseUp);
    document.addEventListener("touchend", handleMouseUp);
    
    return () => {
      document.removeEventListener("mousemove", handleMouseMove);
      document.removeEventListener("touchmove", handleTouchMove);
      document.removeEventListener("mouseup", handleMouseUp);
      document.removeEventListener("touchend", handleMouseUp);
    };
  });

  function formatTime(time: number) {
    const minutes = Math.floor(time / 60);
    const seconds = Math.floor(time % 60);
    return `${minutes}:${seconds.toString().padStart(2, "0")}`;
  }
</script>

<div class="w-full">
  <div class="flex items-center justify-between text-xs text-gray-600 mb-1">
    <span>{formatTime(startTime)}</span>
    <span class="font-medium">{formatTime(endTime - startTime)}</span>
    <span>{formatTime(endTime)}</span>
  </div>
  
  <div
    id="timeline"
    class="relative h-6 bg-gray-200 rounded cursor-pointer"
    on:click={(e) => onTimelineClick(e)}
    use:onTimelineMount
  >
    <!-- Trim range background -->
    <div
      class="absolute top-0 bottom-0 bg-blue-500 opacity-50 rounded-sm"
      style="left: {(startTime / videoDuration) * 100}%; width: {((endTime - startTime) / videoDuration) * 100}%;"
    />
    
    <!-- Red preview line - positioned exactly at cursor -->
    {#if showPreview}
      <div
        class="absolute top-0 bottom-0 w-0.5 bg-red-500 pointer-events-none z-10"
        style="left: {(previewTime / videoDuration) * 100}%; transform: translateX(-50%);"
      />
    {/if}
    
    <!-- Start handle -->
    <div
      class="absolute top-0 bottom-0 w-3 bg-blue-700 cursor-ew-resize rounded-l shadow-md z-20"
      style="left: {(startTime / videoDuration) * 100}%; transform: translateX(-50%);"
      on:mousedown={(e) => handleTrimHandleDrag(e, "start")}
      on:touchstart={(e) => handleTrimHandleDrag(e, "start")}
      role="slider"
      aria-label="Start trim"
      tabindex="0"
      aria-valuenow={Math.round(startTime)}
    />
    
    <!-- End handle -->
    <div
      class="absolute top-0 bottom-0 w-3 bg-blue-700 cursor-ew-resize rounded-r shadow-md z-20"
      style="left: {(endTime / videoDuration) * 100}%; transform: translateX(-50%);"
      on:mousedown={(e) => handleTrimHandleDrag(e, "end")}
      on:touchstart={(e) => handleTrimHandleDrag(e, "end")}
      role="slider"
      aria-label="End trim"
      tabindex="0"
      aria-valuenow={Math.round(endTime)}
    />

    <!-- Frame preview -->
    <div
      class="absolute bottom-full pointer-events-none z-30"
      style="left: {previewPosition}px; transform: translateX(-50%);"
    >
      <FramePreview {videoSrc} {previewTime} visible={showPreview} />
    </div>
  </div>
</div>
