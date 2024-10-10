<script lang="ts">
    import { onMount } from "svelte";
    import { pwaInfo } from "virtual:pwa-info";
    let deferredInstallEvent: any;

    $: webManifestLink = pwaInfo ? pwaInfo.webManifest.linkTag : "";

    async function detectSWUpdate() {
        const registration = await navigator.serviceWorker.ready;

        registration.addEventListener("updatefound", (event) => {
            const newSW = registration.installing;
            newSW?.addEventListener("statechange", (event) => {
                if (newSW.state == "installed") {
                    if(confirm("New update, reload to update")){
                        newSW.postMessage({type:'SKIP_WAITING'});
                        window.location.reload();
                    }
                    
                }
            });
        });
    }

    onMount(() => {
        window.addEventListener("beforeinstallprompt", (e) => {
            e.preventDefault();
            deferredInstallEvent = e;
            console.log(deferredInstallEvent);
        });
        detectSWUpdate();
    });

    async function handleInstall() {
        deferredInstallEvent.prompt();
        let choice = await deferredInstallEvent.userChoice;
        if (choice.outcome === "accepted") {
            // User accepted to install the application
        } else {
            // User dismissed the prompt
        }
        deferredInstallEvent = undefined;
    }
</script>

<svelte:head>
    {@html webManifestLink}
</svelte:head>
{#if deferredInstallEvent}
    <button class="install-button" on:click={handleInstall}>Install</button>
{/if}
<slot />

<style>
    .install-button {
        position: absolute;
        top: 1px;
        left: 1px;
    }
</style>
