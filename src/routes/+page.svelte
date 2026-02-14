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

    onMount(() => {
        loadFfmpeg();
        const videoEl = document.querySelector("video");
        if (videoEl) {
            videoPlayer = videoEl as any;
        }
    });

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
            const file = input.files[0];
            videoSrc = URL.createObjectURL(file);
            videoBlob = file;
            startTime = 0;
            endTime = 0;
            currentTime = 0;
            videoDuration = 0;
            fileName = file.name;
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
            
            const baseFileName = fileName.replace(/\.[^/.]+$/, "");
            a.download = `${baseFileName}-trimmed.${extension}`;
            
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
        
        // Calculate fps based on duration to keep gif size reasonable
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

<main class="w-full h-screen overflow-x-hidden relative py-4 md:py-8">
    <div class="container mx-auto px-1 md:px-4 w-full relative mb-8">
        <div class="text-center mb-2 md:mb-6">
            <div class="text-3xl font-boldtext-center flex justify-center items-center">
                <span>Video Barber</span>
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="50"
                    height="50"
                    viewBox="0 0 72 72"
                >
                    <path
                        fill="#ea5a47"
                        d="M24.488 52.117c-.544 2.69-.44 4.994.305 6.66c2.054 4.6 7.465 6.677 12.07 4.62a9.07 9.07 0 0 0 4.809-5.078a9.08 9.08 0 0 0-.19-6.992c-1.097-2.459-2.47-4.353-5.066-5.103c-.31-.09-.04-1.131.084-1.427l.84-2.813l-7.6-2.53c-3.42 7.73-5.104 11.923-5.252 12.663m10.032-2.005a4.68 4.68 0 0 1 2.938 2.083a4.68 4.68 0 0 1 .602 3.551a4.68 4.68 0 0 1-2.083 2.938a4.68 4.68 0 0 1-3.551.602a4.68 4.68 0 0 1-2.938-2.084a4.68 4.68 0 0 1-.602-3.55a4.67 4.67 0 0 1 2.084-2.938a4.68 4.68 0 0 1 3.55-.602"
                    />
                    <path
                        fill="#d0cfce"
                        d="m37.009 42.077l12.133-30.044c-2.187.695-4.555 1.24-6.5 3.213c-2.443 2.48-4.225 6.392-4.225 6.392s-7.584 16.522-7.952 17.344c-.014.03 6.544 3.095 6.544 3.095"
                    />
                    <path
                        fill="#9b9b9a"
                        d="M61.011 24.5s-15.29 5.18-14.812 4.951l-.004-.004l-3.946 1.648l-5.136 12.966l5.709-2.744l.001-.015c-.102.034 9.415-5.642 9.589-5.731c4.024-2.078 7.027-6.144 8.6-11.07"
                    />
                    <path
                        fill="#d22f27"
                        d="M21.833 32.884c-4.082-1.822-10.4.207-12.222 4.288c-1.823 4.083-.179 10.431 3.903 12.254c1.478.66 3.943 1.056 6.33.573c.103-.02 3.367-.227 2.978.026l2.285-1.063c1.178-2.972 4.716-11.343 5.006-11.988l-2.738 1.313l-.006.012c-.018.284-3.478-4.495-5.536-5.415m-.249 10.35a4.5 4.5 0 0 1-2.837 2.013a4.55 4.55 0 0 1-5.442-3.419a4.5 4.5 0 0 1 .582-3.43a4.5 4.5 0 0 1 2.837-2.011a4.52 4.52 0 0 1 3.43.581a4.5 4.5 0 0 1 2.012 2.837a4.5 4.5 0 0 1-.582 3.43"
                    />
                    <g
                        fill="none"
                        stroke="#000"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-miterlimit="10"
                        stroke-width="2"
                    >
                        <path
                            d="M29.584 39.621c3.058-6.88 6.652-14.644 7.8-16.865c2.07-4.012 5.733-10.19 12.54-11.194l-12.66 30.554"
                        />
                        <path
                            d="M41.462 50.976a9.08 9.08 0 0 0-5.791-5.034l1.593-3.826l-7.68-2.495c-2.506 5.64-4.88 11.216-5.092 12.265c-.362 1.787-.617 4.374.338 6.513a9.107 9.107 0 0 0 16.632-7.423m-3.55 4.736a4.553 4.553 0 1 1-8.878-2.027a4.553 4.553 0 0 1 8.878 2.027M26.373 38.238a9.08 9.08 0 0 0-4.915-5.41a9.107 9.107 0 0 0-7.423 16.632c2.14.955 4.726.7 6.513.338c.293-.059.898-.27 1.742-.597m-3.541-3.945a4.553 4.553 0 1 1-2.027-8.878a4.553 4.553 0 0 1 2.027 8.878m27.876-14.909l14.247-5.982c-1.004 6.808-7.182 10.47-11.194 12.542c-1.312.677-3.311 1.693-7.165 3.463"
                        />
                    </g>
                </svg>
            </div>
            <p>Free Video Trimmer</p>
        </div>

        <div class="mb-4 w-full md:w-1/2 mx-auto">
            <label for="video-upload" class="block mb-2">Select a video file:</label>
            <input
                type="file"
                id="video-upload"
                accept="video/*"
                on:change={handleFileSelect}
                class="flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50 mb-4"
            />
        </div>

        {#if loaded}
            {#if videoSrc}
                <VideoPlayer
                    bind:this={videoPlayer}
                    {videoSrc}
                    bind:currentTime
                    {startTime}
                    {endTime}
                    on:loadedmetadata={handleLoadedMetadata}
                    on:timeupdate={handleTimeUpdate}
                />

                <VideoTrimmer
                    {videoDuration}
                    bind:startTime
                    bind:endTime
                    {videoSrc}
                    on:trimchange={handleTrimChange}
                    on:previewchange={handlePreviewChange}
                />

                <ExportSettings
                    bind:outputFormat
                    bind:videoQuality
                    bind:audioEnabled
                    bind:isGif
                />

                <div class="mt-6 text-center mb-8">
                    <button
                        on:click={handleDownload}
                        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-3 px-8 rounded-lg text-lg disabled:opacity-50 disabled:cursor-not-allowed"
                        disabled={isProcessing}
                    >
                        {#if isProcessing}
                            Processing...
                        {:else}
                            Download {isGif ? 'GIF' : outputFormat.toUpperCase()}
                        {/if}
                    </button>
                </div>
            {:else}
                <p class="text-center text-gray-500">
                    Please select a video file to start trimming.
                </p>
            {/if}
        {:else if ffmpegError != null}
            <p class="text-center text-red-500">{ffmpegError}</p>
        {:else}
            <p class="text-center text-gray-500">Loading ffmpeg...</p>
        {/if}
    </div>

    <div class="fixed bottom-0 p-0 md:p-2 left-0 right-0 text-center">
        <p>@bagusindrayana</p>
    </div>
</main>
