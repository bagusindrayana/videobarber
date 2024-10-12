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
    const clickPosition = (e.clientX - timelineRect.left) / timelineRect.width;
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

    const handleTouchMove = (e: TouchEvent) => {
      if (!isDragging) return;
      const rect = timelineRef.getBoundingClientRect();
      const x = Math.max(
        0,
        Math.min(e.targetTouches[0].clientX - rect.left, rect.width),
      );
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
    document.addEventListener("touchmove", handleTouchMove);
    document.addEventListener("mouseup", handleMouseUp);
    document.addEventListener("touchend", handleMouseUp);
  });

  function formatTime(time: number) {
    const minutes = Math.floor(time / 60);
    const seconds = Math.floor(time % 60);
    return `${minutes}:${seconds.toString().padStart(2, "0")}`;
  }
</script>

<div class="w-full max-w-3xl mx-auto mt-4">
  
  
  <div class="flex justify-between mb-2">
    <span>From : {formatTime(startTime)}</span>
    <span>To : {formatTime(endTime)}</span>
  </div>
  <div class="space-y-2 px-4 md:px-0 ">
    <label >Timeline</label>
    <div
      id="timeline"
      class="relative h-8 bg-gray-200 rounded cursor-pointer"
      on:click={(e) => onTimelineClick(e)}
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
        on:touchstart={(e) => handleTrimHandleDrag(e, "start")}
      />
      <div
        class="absolute top-0 bottom-0 w-1 bg-blue-700 cursor-ew-resize"
        style="left: {(endTime / videoDuration) * 100}%"
        on:mousedown={(e) => handleTrimHandleDrag(e, "end")}
        on:touchstart={(e) => handleTrimHandleDrag(e, "end")}
      />

      <div
        class="absolute bottom-full  pointer-events-none"
        style="left: {previewPosition}px; transform: translateX(-50%);"
      >
        <FramePreview {videoSrc} {previewTime} visible={showPreview} />
      </div>
    </div>
    <!-- <div class="flex justify-between text-sm text-muted-foreground">
      <span>From : {formatTime(startTime)}</span>
      <span>To : {formatTime(endTime)}</span>
    </div> -->
   
    <div class="flex justify-between text-sm text-muted-foreground">
      <span>Length : {formatTime(endTime-startTime)}</span>
    </div>
    <div class="text-center">
      <p>
        Trim Range: {formatTime(startTime)} - {formatTime(endTime)}
    </p>
    </div>
  </div>
</div>
