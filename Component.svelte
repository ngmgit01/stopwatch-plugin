<script>
  import { getContext, onDestroy, onMount } from "svelte";
  import { createEventDispatcher } from "svelte";

  export let fieldStart;
  export let fieldStop;
  export let fieldTotal;
  export let tableId;
  export let label = "Stopwatch";

  const dispatch = createEventDispatcher();
  const component = getContext("component");
  const formContext = getContext("form");
  const formApi = formContext?.formApi;

  let time = 0; // milliseconds
  let isRunning = false;
  let isPaused = false;
  let interval = null;
  let startTime = null;
  let elapsedBeforePause = 0;
  let startTimestamp = null;
  let stopTimestamp = null;
  let totalElapsed = 0;

  // Form field registration
  let fieldApiStart, fieldApiStop, fieldApiTotal;
  let fieldStateStart, fieldStateStop, fieldStateTotal;

  $: unsubscribeStart = formApi?.registerField(fieldStart, "number", 0, false, null, 1)
    ?.subscribe((value) => {
      fieldStateStart = value?.fieldState;
      fieldApiStart = value?.fieldApi;
    });

  $: unsubscribeStop = formApi?.registerField(fieldStop, "number", 0, false, null, 1)
    ?.subscribe((value) => {
      fieldStateStop = value?.fieldState;
      fieldApiStop = value?.fieldApi;
    });

  $: unsubscribeTotal = formApi?.registerField(fieldTotal, "number", 0, false, null, 1)
    ?.subscribe((value) => {
      fieldStateTotal = value?.fieldState;
      fieldApiTotal = value?.fieldApi;
    });

  onDestroy(() => {
    fieldApiStart?.deregister();
    unsubscribeStart?.();
    fieldApiStop?.deregister();
    unsubscribeStop?.();
    fieldApiTotal?.deregister();
    unsubscribeTotal?.();
    if (interval) clearInterval(interval);
  });

  // Format time as MM:SS:ss (minutes:seconds:split seconds)
  function formatTime(ms) {
    const totalSeconds = Math.floor(ms / 1000);
    const minutes = String(Math.floor(totalSeconds / 60)).padStart(2, "0");
    const seconds = String(totalSeconds % 60).padStart(2, "0");
    const splitSeconds = String(Math.floor((ms % 1000) / 10)).padStart(2, "0");
    return `${minutes}:${seconds}:${splitSeconds}`;
  }

  // Save data to configured table fields
  async function saveToTable(field, value) {
    try {
      const response = await fetch(`/api/rows/${tableId}`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          [field]: value
        })
      });
      if (!response.ok) throw new Error("Failed to save");
      return await response.json();
    } catch (error) {
      console.error("Error saving to table:", error);
    }
  }

  function startTimer() {
    if (!isRunning && !isPaused) {
      // Start from zero
      startTime = Date.now();
      startTimestamp = Date.now();
      isRunning = true;
      isPaused = false;
      elapsedBeforePause = 0;
      
      // Write current time to start field
      const currentTime = new Date().toISOString();
      fieldApiStart?.setValue(currentTime);
      saveToTable(fieldStart, currentTime);
      dispatch("start", { time: currentTime });

      interval = setInterval(() => {
        time = Date.now() - startTime;
      }, 10);
    } else if (isPaused) {
      // Resume from pause
      startTime = Date.now();
      isRunning = true;
      isPaused = false;
      interval = setInterval(() => {
        time = elapsedBeforePause + (Date.now() - startTime);
      }, 10);
    }
  }

  function pauseTimer() {
    if (isRunning) {
      isRunning = false;
      isPaused = true;
      elapsedBeforePause = time;
      if (interval) {
        clearInterval(interval);
        interval = null;
      }
    }
  }

  function stopTimer() {
    if (isRunning || isPaused) {
      isRunning = false;
      isPaused = false;
      totalElapsed = time;
      stopTimestamp = Date.now();
      
      if (interval) {
        clearInterval(interval);
        interval = null;
      }

      // Write stop time
      const stopTime = new Date().toISOString();
      fieldApiStop?.setValue(stopTime);
      saveToTable(fieldStop, stopTime);

      // Write total elapsed
      const elapsedSeconds = (totalElapsed / 1000).toFixed(2);
      fieldApiTotal?.setValue(elapsedSeconds);
      saveToTable(fieldTotal, elapsedSeconds);
      dispatch("stop", { total: elapsedSeconds });

      // Reset timer display but keep total
      time = 0;
      elapsedBeforePause = 0;
    }
  }

  function resetTimer() {
    if (!isRunning && !isPaused) {
      time = 0;
      totalElapsed = 0;
      startTimestamp = null;
      stopTimestamp = null;
      fieldApiStart?.setValue(null);
      fieldApiStop?.setValue(null);
      fieldApiTotal?.setValue(null);
    }
  }
</script>

<div class="stopwatch-wrapper">
  {#if label}
    <div class="label">{label}</div>
  {/if}

  <div class="stopwatch-container">
    <div class="timer-display" class:blinking={isPaused}>
      {formatTime(time || 0)}
    </div>

    <div class="button-group">
      {#if !isRunning && !isPaused}
        <button class="btn btn-start" on:click={startTimer}>
          Start
        </button>
      {/if}

      {#if isRunning}
        <button class="btn btn-pause" on:click={pauseTimer}>
          Pause
        </button>
      {/if}

      {#if isPaused}
        <button class="btn btn-resume" on:click={startTimer}>
          Resume
        </button>
      {/if}

      <button 
        class="btn btn-stop" 
        on:click={stopTimer}
        disabled={!isRunning && !isPaused}
      >
        Stop
      </button>
    </div>

    {#if totalElapsed > 0 && !isRunning && !isPaused}
      <div class="total-display">
        Total: {formatTime(totalElapsed)}
      </div>
    {/if}
  </div>
</div>

<style>
  .stopwatch-wrapper {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 16px;
    font-family: inherit;
  }

  .label {
    font-weight: 600;
    margin-bottom: 12px;
    color: var(--spectrum-global-color-gray-700);
  }

  .stopwatch-container {
    background: var(--spectrum-global-color-gray-100);
    border-radius: 12px;
    padding: 24px 32px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    text-align: center;
    min-width: 280px;
  }

  .timer-display {
    font-size: 3rem;
    font-weight: 700;
    font-variant-numeric: tabular-nums;
    letter-spacing: 2px;
    color: var(--spectrum-global-color-gray-800);
    padding: 16px 0;
    transition: opacity 0.3s;
  }

  .blinking {
    animation: blink 0.8s ease-in-out infinite;
  }

  @keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.2; }
  }

  .button-group {
    display: flex;
    gap: 12px;
    justify-content: center;
    flex-wrap: wrap;
    margin-top: 8px;
  }

  .btn {
    padding: 10px 24px;
    border: none;
    border-radius: 6px;
    font-weight: 600;
    font-size: 1rem;
    cursor: pointer;
    transition: all 0.2s ease;
    min-width: 80px;
  }

  .btn:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  .btn-start {
    background: var(--spectrum-global-color-green-500);
    color: white;
  }
  .btn-start:hover:not(:disabled) {
    background: var(--spectrum-global-color-green-600);
  }

  .btn-pause {
    background: var(--spectrum-global-color-yellow-500);
    color: white;
  }
  .btn-pause:hover:not(:disabled) {
    background: var(--spectrum-global-color-yellow-600);
  }

  .btn-resume {
    background: var(--spectrum-global-color-blue-500);
    color: white;
  }
  .btn-resume:hover:not(:disabled) {
    background: var(--spectrum-global-color-blue-600);
  }

  .btn-stop {
    background: var(--spectrum-global-color-red-500);
    color: white;
  }
  .btn-stop:hover:not(:disabled) {
    background: var(--spectrum-global-color-red-600);
  }

  .total-display {
    margin-top: 16px;
    padding: 8px;
    background: var(--spectrum-global-color-gray-200);
    border-radius: 6px;
    font-weight: 500;
    color: var(--spectrum-global-color-gray-700);
    font-size: 1.1rem;
  }
</style>