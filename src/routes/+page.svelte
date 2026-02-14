<script lang="ts">
    import { onMount } from "svelte";
    import VideoPlayer from "../lib/VideoPlayer.svelte";
    import VideoTrimmer from "../lib/VideoTrimmer.svelte";
    import ExportSettings from "../lib/ExportSettings.svelte";
    import "../app.css";
    import { FFmpeg } from "@ffmpeg/ffmpeg";

    let loaded: boolean = false;
    let ffmpegRef: FFmpeg;
    let videoSrc = "";
    let videoDuration = 0;
    let currentTime = 0;
    let startTime = 0;
    let endTime = 0;
    let videoPlayer: VideoPlayer;
    let videoBlob: Blob | null = null;
    let isProcessing = false;
    let fileName: string;
    let ffmpegError: string | null = null;

    // Export settings
    let outputFormat = "mp4";
    let videoQuality = "high";
    let audioEnabled = true;
    let isGif = false;
    let exportFileName = "trimmed-video";

    // Drag and drop state
    let isDragging = false;
    let dragCounter = 0;

    onMount(() => {
        loadFfmpeg();
        const videoEl = document.querySelector("video");
        if (videoEl) {
            videoPlayer = videoEl as any;
        }
    });

    function handleDragEnter(e: DragEvent) {
        e.preventDefault();
        e.stopPropagation();
        dragCounter++;
        if (!videoSrc) {
            isDragging = true;
        }
    }

    function handleDragLeave(e: DragEvent) {
        e.preventDefault();
        e.stopPropagation();
        dragCounter--;
        if (dragCounter === 0) {
            isDragging = false;
        }
    }

    function handleDragOver(e: DragEvent) {
        e.preventDefault();
        e.stopPropagation();
    }

    function handleDrop(e: DragEvent) {
        e.preventDefault();
        e.stopPropagation();
        isDragging = false;
        dragCounter = 0;

        const files = e.dataTransfer?.files;
        if (files && files.length > 0) {
            const file = files[0];
            if (file.type.startsWith('video/')) {
                loadVideoFile(file);
            } else {
                alert('Please drop a video file');
            }
        }
    }

    function loadVideoFile(file: File) {
        videoSrc = URL.createObjectURL(file);
        videoBlob = file;
        startTime = 0;
        endTime = 0;
        currentTime = 0;
        videoDuration = 0;
        fileName = file.name;
        // Set default export name without extension
        exportFileName = file.name.replace(/\.[^/.]+$/, "");
    }

    async function readFile(file: Blob): Promise<Uint8Array> {
        return new Promise((resolve) => {
            const fileReader = new FileReader();
            fileReader.onload = () => {
                const { result } = fileReader;
                if (result instanceof ArrayBuffer) {
                    resolve(new Uint8Array(result));
                }
            };

            fileReader.onerror = (e) => {
                console.log("Could not read file", e);
            };

            fileReader.readAsArrayBuffer(file);
        });
    }

    const loadFfmpeg = async () => {
        const baseURL = "/ffmpeg";
        ffmpegRef = new FFmpeg();
        ffmpegRef.on("log", ({ message }) => {
            console.log(message);
        });
        ffmpegRef.on("progress", ({ progress, time }) => {
            console.log(progress, time);
        });
        try {
            await ffmpegRef.load({
                coreURL: `${baseURL}/ffmpeg-core.js`,
                wasmURL: `${baseURL}/ffmpeg-core.wasm`
            });
            loaded = true;
        } catch (error) {
            ffmpegError = "Failed load ffmpeg : " + error;
        }
    };

    function handleLoadedMetadata(event: CustomEvent) {
        videoDuration = event.detail.duration;
        endTime = videoDuration;
    }

    function handleTrimChange(event: CustomEvent) {
        const newStartTime = event.detail.startTime;
        const newEndTime = event.detail.endTime;

        if (newStartTime !== startTime) {
            startTime = newStartTime;
            if (currentTime < startTime) {
                videoPlayer.updateVideoTime(startTime);
            }
        }

        endTime = newEndTime;
    }

    function handlePreviewChange(event: CustomEvent) {
        if (videoPlayer) {
            videoPlayer.updateVideoTime(event.detail.time);
            currentTime = event.detail.time;
        }
    }

    function handleTimeUpdate(event: CustomEvent) {
        currentTime = event.detail.currentTime;
    }

    function handleFileSelect(event: Event) {
        const input = event.target as HTMLInputElement;
        if (input.files && input.files[0]) {
            loadVideoFile(input.files[0]);
        }
    }

    function getQualitySettings(quality: string): string[] {
        switch (quality) {
            case "high":
                return ["-vf", "scale=-2:1080", "-crf", "23"];
            case "medium":
                return ["-vf", "scale=-2:720", "-crf", "28"];
            case "low":
                return ["-vf", "scale=-2:480", "-crf", "35"];
            default:
                return ["-c:v", "copy"];
        }
    }

    function getAudioSettings(audioEnabled: boolean): string[] {
        if (audioEnabled) {
            return ["-c:a", "aac", "-b:a", "128k"];
        } else {
            return ["-an"];
        }
    }

    async function handleDownload() {
        if (!loaded) {
            alert("Ffmpeg not loaded yet");
            return;
        }
        if (!videoBlob) {
            alert("No video file found. Please load a video first.");
            return;
        }

        isProcessing = true;

        try {
            let result: Blob;
            const extension = isGif ? "gif" : outputFormat;

            if (isGif) {
                result = await createGif(videoBlob, startTime, endTime);
            } else {
                result = await trimVideo(videoBlob, startTime, endTime, outputFormat, videoQuality, audioEnabled);
            }

            const url = URL.createObjectURL(result);
            const a = document.createElement("a");
            a.href = url;
            
            const finalFileName = exportFileName.trim() || "trimmed-video";
            a.download = `${finalFileName}.${extension}`;
            
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        } catch (error) {
            console.error("Error processing video:", error);
            alert("An error occurred while processing the video. Please try again.");
        } finally {
            isProcessing = false;
        }
    }

    async function trimVideo(
        videoBlob: Blob,
        start: number,
        end: number,
        format: string,
        quality: string,
        audio: boolean
    ): Promise<Blob> {
        const inputFileName = "input" + getExtensionFromBlob(videoBlob);
        const outputFileName = `output.${format}`;
        
        await ffmpegRef.writeFile(inputFileName, await readFile(videoBlob));
        
        const args = [
            "-i", inputFileName,
            "-ss", start.toString(),
            "-to", end.toString(),
            ...getQualitySettings(quality),
            ...getAudioSettings(audio),
            "-movflags", "+faststart",
            outputFileName
        ];
        
        await ffmpegRef.exec(args);

        const data = await ffmpegRef.readFile(outputFileName) as Uint8Array;
        const mimeType = getMimeType(format);
        return new Blob([data.buffer as ArrayBuffer], { type: mimeType });
    }

    function getExtensionFromBlob(blob: Blob): string {
        const type = blob.type;
        if (type.includes("mp4")) return ".mp4";
        if (type.includes("webm")) return ".webm";
        if (type.includes("quicktime")) return ".mov";
        if (type.includes("x-msvideo")) return ".avi";
        if (type.includes("x-matroska")) return ".mkv";
        return ".mp4";
    }

    function getMimeType(format: string): string {
        const mimeTypes: Record<string, string> = {
            "mp4": "video/mp4",
            "webm": "video/webm",
            "mov": "video/quicktime",
            "avi": "video/x-msvideo",
            "mkv": "video/x-matroska"
        };
        return mimeTypes[format] || "video/mp4";
    }

    async function createGif(
        videoBlob: Blob,
        start: number,
        end: number,
    ): Promise<Blob> {
        const inputFileName = "input" + getExtensionFromBlob(videoBlob);
        const outputFileName = "output.gif";
        
        await ffmpegRef.writeFile(inputFileName, await readFile(videoBlob));
        
        const duration = end - start;
        const fps = duration > 10 ? 10 : 15;
        const scale = 480;
        
        const args = [
            "-i", inputFileName,
            "-ss", start.toString(),
            "-to", end.toString(),
            "-vf", `fps=${fps},scale=${scale}:-1:flags=lanczos,split[s0][s1];[s0]palettegen=max_colors=128[p];[s1][p]paletteuse`,
            "-loop", "0",
            outputFileName
        ];
        
        await ffmpegRef.exec(args);

        const data = await ffmpegRef.readFile(outputFileName) as Uint8Array;
        return new Blob([data.buffer as ArrayBuffer], { type: "image/gif" });
    }
</script>

<svelte:head>
    <title>VideoBarber - Trim Video Online</title>
    <meta name="description" content="Free Video Trimmer - VideoBarber" />
</svelte:head>

<div class="h-screen flex flex-col overflow-hidden">
    <!-- Header -->
    <header class="flex-shrink-0 bg-white border-b px-4 py-2 flex items-center justify-between">
        <div class="flex items-center gap-2">
            <svg xmlns="http://www.w3.org/2000/svg" width="28" height="28" viewBox="0 0 72 72">
                <path fill="#ea5a47" d="M24.488 52.117c-.544 2.69-.44 4.994.305 6.66c2.054 4.6 7.465 6.677 12.07 4.62a9.07 9.07 0 0 0 4.809-5.078a9.08 9.08 0 0 0-.19-6.992c-1.097-2.459-2.47-4.353-5.066-5.103c-.31-.09-.04-1.131.084-1.427l.84-2.813l-7.6-2.53c-3.42 7.73-5.104 11.923-5.252 12.663m10.032-2.005a4.68 4.68 0 0 1 2.938 2.083a4.68 4.68 0 0 1 .602 3.551a4.68 4.68 0 0 1-2.083 2.938a4.68 4.68 0 0 1-3.551.602a4.68 4.68 0 0 1-2.938-2.084a4.68 4.68 0 0 1-.602-3.55a4.67 4.67 0 0 1 2.084-2.938a4.68 4.68 0 0 1 3.55-.602"/>
                <path fill="#d22f27" d="M21.833 32.884c-4.082-1.822-10.4.207-12.222 4.288c-1.823 4.083-.179 10.431 3.903 12.254c1.478.66 3.943 1.056 6.33.573c.103-.02 3.367-.227 2.978.026l2.285-1.063c1.178-2.972 4.716-11.343 5.006-11.988l-2.738 1.313l-.006.012c-.018.284-3.478-4.495-5.536-5.415m-.249 10.35a4.5 4.5 0 0 1-2.837 2.013a4.55 4.55 0 0 1-5.442-3.419a4.5 4.5 0 0 1 .582-3.43a4.5 4.5 0 0 1 2.837-2.011a4.52 4.52 0 0 1 3.43.581a4.5 4.5 0 0 1 2.012 2.837a4.5 4.5 0 0 1-.582 3.43"/>
                <g fill="none" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-miterlimit="10" stroke-width="2">
                    <path d="M29.584 39.621c3.058-6.88 6.652-14.644 7.8-16.865c2.07-4.012 5.733-10.19 12.54-11.194l-12.66 30.554"/>
                    <path d="M41.462 50.976a9.08 9.08 0 0 0-5.791-5.034l1.593-3.826l-7.68-2.495c-2.506 5.64-4.88 11.216-5.092 12.265c-.362 1.787-.617 4.374.338 6.513a9.107 9.107 0 0 0 16.632-7.423m-3.55 4.736a4.553 4.553 0 1 1-8.878-2.027a4.553 4.553 0 0 1 8.878 2.027M26.373 38.238a9.08 9.08 0 0 0-4.915-5.41a9.107 9.107 0 0 0-7.423 16.632c2.14.955 4.726.7 6.513.338c.293-.059.898-.27 1.742-.597m-3.541-3.945a4.553 4.553 0 1 1-2.027-8.878a4.553 4.553 0 0 1 2.027 8.878"/>
                </g>
            </svg>
            <span class="font-bold text-lg">VideoBarber</span>
        </div>
        <span class="text-xs text-gray-500 hidden sm:inline">Free Video Trimmer</span>
    </header>

    <!-- Main Content -->
    <main 
        class="flex-1 flex flex-col lg:flex-row overflow-hidden"
        on:dragenter={handleDragEnter}
        on:dragleave={handleDragLeave}
        on:dragover={handleDragOver}
        on:drop={handleDrop}
    >
        {#if loaded}
            {#if videoSrc}
                <!-- Left Side: Video -->
                <div class="flex-1 flex flex-col p-4 overflow-auto">
                    <VideoPlayer
                        bind:this={videoPlayer}
                        {videoSrc}
                        bind:currentTime
                        {startTime}
                        {endTime}
                        on:loadedmetadata={handleLoadedMetadata}
                        on:timeupdate={handleTimeUpdate}
                    />
                    
                    <!-- Timeline -->
                    <div class="mt-4">
                        <VideoTrimmer
                            {videoDuration}
                            bind:startTime
                            bind:endTime
                            {videoSrc}
                            on:trimchange={handleTrimChange}
                            on:previewchange={handlePreviewChange}
                        />
                    </div>
                </div>

                <!-- Right Side: Controls -->
                <div class="lg:w-80 bg-gray-50 border-t lg:border-t-0 lg:border-l p-4 flex flex-col gap-4 overflow-auto">
                    <!-- File Info -->
                    <div class="bg-white rounded-lg p-3 border">
                        <h3 class="font-semibold text-sm mb-1">Source File</h3>
                        <p class="text-xs text-gray-600 truncate">{fileName}</p>
                    </div>

                    <!-- Export File Name -->
                    <div class="bg-white rounded-lg p-3 border">
                        <h3 class="font-semibold text-sm mb-2">Export Name</h3>
                        <div class="flex items-center gap-2">
                            <input
                                type="text"
                                bind:value={exportFileName}
                                placeholder="Enter file name"
                                class="flex-1 px-3 py-2 text-sm border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                            />
                            <span class="text-sm text-gray-500">.{isGif ? 'gif' : outputFormat}</span>
                        </div>
                    </div>

                    <!-- Export Settings -->
                    <div class="bg-white rounded-lg p-3 border">
                        <h3 class="font-semibold text-sm mb-2">Export Settings</h3>
                        <ExportSettings
                            bind:outputFormat
                            bind:videoQuality
                            bind:audioEnabled
                            bind:isGif
                        />
                    </div>

                    <!-- Download Button -->
                    <button
                        on:click={handleDownload}
                        class="w-full bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-4 rounded-lg disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
                        disabled={isProcessing}
                    >
                        {#if isProcessing}
                            <span class="flex items-center justify-center gap-2">
                                <svg class="animate-spin h-5 w-5" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                                </svg>
                                Processing...
                            </span>
                        {:else}
                            Download {isGif ? 'GIF' : outputFormat.toUpperCase()}
                        {/if}
                    </button>
                </div>
            {:else}
                <!-- Upload State with Drag & Drop -->
                <div class="flex-1 flex items-center justify-center p-4">
                    <div 
                        class="w-full max-w-md text-center transition-all duration-200 {isDragging ? 'scale-105' : ''}"
                    >
                        <div 
                            class="border-2 border-dashed rounded-xl p-8 transition-colors {isDragging ? 'border-blue-500 bg-blue-50' : 'border-gray-300 bg-white'}"
                        >
                            <div class="mb-6">
                                {#if isDragging}
                                    <svg class="mx-auto h-16 w-16 text-blue-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
                                    </svg>
                                {:else}
                                    <svg class="mx-auto h-16 w-16 text-gray-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 4v16M17 4v16M3 8h4m10 0h4M3 12h18M3 16h4m10 0h4M4 20h16a1 1 0 001-1V5a1 1 0 00-1-1H4a1 1 0 00-1 1v14a1 1 0 001 1z" />
                                    </svg>
                                {/if}
                            </div>
                            
                            <h2 class="text-xl font-semibold mb-2">
                                {#if isDragging}
                                    Drop video here
                                {:else}
                                    Upload Your Video
                                {/if}
                            </h2>
                            <p class="text-gray-500 mb-6 text-sm">
                                {isDragging ? 'Release to upload' : 'Drag & drop or click to select a video file'}
                            </p>
                            
                            <label class="cursor-pointer inline-flex items-center justify-center px-6 py-3 bg-blue-500 hover:bg-blue-600 text-white rounded-lg transition-colors">
                                <span>Choose file</span>
                                <input
                                    type="file"
                                    accept="video/*"
                                    on:change={handleFileSelect}
                                    class="hidden"
                                />
                            </label>
                        </div>
                    </div>
                </div>
            {/if}
        {:else if ffmpegError != null}
            <div class="flex-1 flex items-center justify-center">
                <p class="text-red-500">{ffmpegError}</p>
            </div>
        {:else}
            <div class="flex-1 flex items-center justify-center">
                <div class="text-center">
                    <svg class="animate-spin h-8 w-8 text-blue-500 mx-auto mb-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                    <p class="text-gray-500">Loading ffmpeg...</p>
                </div>
            </div>
        {/if}
    </main>

    <!-- Footer -->
    <footer class="flex-shrink-0 bg-white border-t px-4 py-2 text-center text-xs text-gray-500">
        @bagusindrayana
    </footer>
</div>
