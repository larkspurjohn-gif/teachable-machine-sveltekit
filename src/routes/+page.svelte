<svelte:head>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest/dist/tf.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/speech-commands@latest/dist/speech-commands.min.js"></script>
</svelte:head>

<script>
    import { onDestroy, onMount } from 'svelte';
    import { browser } from '$app/environment';

    const MODEL_PATH = '/tm-my-audio-model/';

    let recognizer;
    let predictions = [];
    let isStarting = false;
    let isRunning = false;
    let libReady = false; // New state to track the library
    let errorMessage = "";
    
    let animalMood = "😊"; 
    let isScared = false;

    // Check if the library is loaded as soon as the page mounts
    onMount(() => {
        const checkLib = setInterval(() => {
            if (window.speechCommands) {
                libReady = true;
                clearInterval(checkLib);
                console.log("✅ Speech Commands library is now ready.");
            }
        }, 500);

        return () => clearInterval(checkLib);
    });

    async function init() {
        if (!browser || isStarting || isRunning || !libReady) return;
        isStarting = true;
        errorMessage = "";

        try {
            const host = window.location.origin;
            const checkpointURL = host + MODEL_PATH + 'model.json';
            const metadataURL = host + MODEL_PATH + 'metadata.json';

            // Double check
            if (!window.speechCommands) {
                throw new Error("Library disappeared! Please refresh.");
            }

            recognizer = window.speechCommands.create("BROWSER_FFT", undefined, checkpointURL, metadataURL);
            await recognizer.ensureModelLoaded();

            const labels = recognizer.wordLabels();
            predictions = labels.map(label => ({ className: label, probability: 0 }));

            await recognizer.listen(result => {
                predictions = labels.map((label, i) => ({
                    className: label,
                    probability: result.scores[i]
                }));
                
                // Trigger scare if any non-background sound is > 80%
                for (const p of predictions) {
                    if (p.className !== "Background Noise" && p.probability > 0.80) {
                        triggerScare();
                        break; 
                    }
                }
            }, {
                probabilityThreshold: 0.70,
                overlapFactor: 0.5
            });

            isRunning = true;
        } catch (err) {
            console.error("Start Error:", err);
            errorMessage = err.message;
        } finally {
            isStarting = false;
        }
    }

    function triggerScare() {
        if (isScared) return;
        animalMood = "🐇";
        isScared = true;
        setTimeout(() => {
            animalMood = "🐰";
            isScared = false;
        }, 1500);
    }

    function stop() {
        if (recognizer) recognizer.stopListening();
        isRunning = false;
    }

    onDestroy(stop);
</script>

<style>
    .shake { display: inline-block; animation: shake 0.2s infinite; }
    @keyframes shake {
        0% { transform: translate(1px, 1px); }
        50% { transform: translate(-2px, -1px); }
        100% { transform: translate(1px, 1px); }
    }
    button:disabled { cursor: not-allowed; opacity: 0.6; }
</style>

<main style="text-align: center; padding: 20px;">
    <div style="font-size: 80px;" class={isScared ? 'shake' : ''}>
        {animalMood}
    </div>

    {#if !libReady}
        <p style="color: orange;">⏳ Downloading AI libraries... please wait.</p>
    {/if}

    {#if errorMessage}
        <p style="color: red;">❌ {errorMessage}</p>
    {/if}

    <button on:click={init} disabled={!libReady || isStarting || isRunning}>
        {!libReady ? 'Waiting for Library...' : isStarting ? 'Starting...' : isRunning ? 'Listening' : 'Start Microphone'}
    </button>
    
    <button on:click={stop} disabled={!isRunning}>Stop</button>

    <div style="margin-top: 20px; text-align: left; max-width: 250px; margin-inline: auto;">
        {#each predictions as p}
            <div style="margin-bottom: 8px;">
                <small style="display:block">{p.className}: {(p.probability * 100).toFixed(0)}%</small>
                <div style="background: #eee; height: 6px; border-radius: 3px; width: 100%;">
                    <div style="background: {p.probability > 0.8 ? '#ff4d4d' : '#4caf50'}; 
                                height: 100%; width: {p.probability * 100}%; transition: width 0.1s;"></div>
                </div>
            </div>
        {/each}
    </div>
</main>