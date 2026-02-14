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
    { value: "high", label: "High (1080p)" },
    { value: "medium", label: "Medium (720p)" },
    { value: "low", label: "Low (480p)" },
  ];

  $: if (isGif) {
    audioEnabled = false;
  }
</script>

<div class="w-full max-w-3xl mx-auto mt-6 p-4 bg-gray-50 rounded-lg border">
  <h3 class="text-lg font-semibold mb-4">Export Settings</h3>
  
  <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
    <!-- Output Format -->
    <div>
      <label class="block text-sm font-medium mb-2">Output Format</label>
      <select
        bind:value={outputFormat}
        disabled={isGif}
        class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:bg-gray-200"
      >
        {#each formats as format}
          <option value={format.value}>{format.label}</option>
        {/each}
      </select>
    </div>

    <!-- Video Quality -->
    <div>
      <label class="block text-sm font-medium mb-2">Video Quality</label>
      <select
        bind:value={videoQuality}
        disabled={isGif}
        class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:bg-gray-200"
      >
        {#each qualities as quality}
          <option value={quality.value}>{quality.label}</option>
        {/each}
      </select>
    </div>
  </div>

  <!-- Audio Toggle -->
  <div class="mt-4 flex items-center justify-between">
    <label class="flex items-center space-x-2 cursor-pointer">
      <input
        type="checkbox"
        bind:checked={audioEnabled}
        disabled={isGif}
        class="w-4 h-4 text-blue-600 rounded focus:ring-blue-500 disabled:opacity-50"
      />
      <span class="text-sm font-medium {isGif ? 'text-gray-400' : ''}">Include Audio</span>
    </label>
  </div>

  <!-- GIF Toggle -->
  <div class="mt-4 pt-4 border-t border-gray-200">
    <label class="flex items-center space-x-2 cursor-pointer">
      <input
        type="checkbox"
        bind:checked={isGif}
        class="w-4 h-4 text-green-600 rounded focus:ring-green-500"
      />
      <span class="text-sm font-medium">Export as GIF</span>
      <span class="text-xs text-gray-500">(Animated image without audio)</span>
    </label>
  </div>

  <!-- Summary -->
  <div class="mt-4 pt-4 border-t border-gray-200 text-sm text-gray-600">
    <p>Output: {isGif ? "GIF" : `${outputFormat.toUpperCase()} ${videoQuality !== 'original' ? `(${videoQuality})` : ''} ${audioEnabled ? 'with audio' : 'no audio'}`}</p>
  </div>
</div>
