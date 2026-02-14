<script lang="ts">
  export let outputFormat: string = "mp4";
  export let videoQuality: string = "high";
  export let audioEnabled: boolean = true;
  export let isGif: boolean = false;

  const formats = [
    { value: "mp4", label: "MP4" },
    { value: "webm", label: "WebM" },
    { value: "mov", label: "MOV" },
    { value: "avi", label: "AVI" },
    { value: "mkv", label: "MKV" },
  ];

  const qualities = [
    { value: "original", label: "Original" },
    { value: "high", label: "High" },
    { value: "medium", label: "Med" },
    { value: "low", label: "Low" },
  ];

  $: if (isGif) {
    audioEnabled = false;
  }
</script>

<div class="w-full">
  <div class="grid grid-cols-3 gap-2 text-xs">
    <!-- Output Format -->
    <div>
      <label class="block text-xs font-medium mb-1 text-gray-600">Format</label>
      <select
        bind:value={outputFormat}
        disabled={isGif}
        class="w-full rounded border border-gray-300 px-2 py-1 text-xs focus:outline-none focus:ring-1 focus:ring-blue-500 disabled:bg-gray-200"
      >
        {#each formats as format}
          <option value={format.value}>{format.label}</option>
        {/each}
      </select>
    </div>

    <!-- Video Quality -->
    <div>
      <label class="block text-xs font-medium mb-1 text-gray-600">Quality</label>
      <select
        bind:value={videoQuality}
        disabled={isGif}
        class="w-full rounded border border-gray-300 px-2 py-1 text-xs focus:outline-none focus:ring-1 focus:ring-blue-500 disabled:bg-gray-200"
      >
        {#each qualities as quality}
          <option value={quality.value}>{quality.label}</option>
        {/each}
      </select>
    </div>

    <!-- Options -->
    <div class="flex flex-col justify-center gap-1">
      <label class="flex items-center gap-1 cursor-pointer">
        <input
          type="checkbox"
          bind:checked={audioEnabled}
          disabled={isGif}
          class="w-3 h-3 text-blue-600 rounded focus:ring-blue-500 disabled:opacity-50"
        />
        <span class="text-xs {isGif ? 'text-gray-400' : 'text-gray-700'}">Audio</span>
      </label>
      
      <label class="flex items-center gap-1 cursor-pointer">
        <input
          type="checkbox"
          bind:checked={isGif}
          class="w-3 h-3 text-green-600 rounded focus:ring-green-500"
        />
        <span class="text-xs text-gray-700">GIF</span>
      </label>
    </div>
  </div>

  <!-- Summary -->
  <div class="mt-2 text-xs text-gray-500 text-center">
    {isGif ? "GIF (no audio)" : `${outputFormat.toUpperCase()} ${videoQuality} ${audioEnabled ? '+audio' : 'no audio'}`}
  </div>
</div>
