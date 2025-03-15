<script lang="ts">
    import { onDestroy, createEventDispatcher } from "svelte";
    import confetti from "canvas-confetti";

    // Create event dispatcher
    const dispatch = createEventDispatcher();

    let timeRemaining = $state(15); // Set to 15 seconds for testing, switch to 1500 for production
    let totalTime = $state(15); // Same as above
    let isRunning = $state(false);
    let animationPercentage = $state(0); // Start empty
    let isCompleted = $state(false);

    // Animation variables
    let timer = $state<ReturnType<typeof setInterval> | null>(null);
    let animationTimer = $state<ReturnType<typeof setInterval> | null>(null);

    // For smoother animation, update more frequently
    const ANIMATION_INTERVAL = 50; // ms
    const ANIMATION_STEPS = 1000 / ANIMATION_INTERVAL; // steps per second

    // Used to accurately display the timers progress
    let targetPercentage = $derived(
        100 - Math.round((timeRemaining / totalTime) * 100),
    );

    let formattedTime = $derived(formatTime(timeRemaining));

    function formatTime(seconds: number): string {
        const mins = Math.floor(seconds / 60);
        const secs = seconds % 60;
        return `${mins.toString().padStart(2, "0")}:${secs.toString().padStart(2, "0")}`;
    }

    // Start the timer
    function startTimer() {
        if (!isRunning) {
            isRunning = true;
            isCompleted = false;
        }

        if (isRunning) {
            // Timer for countdown logic
            timer = setInterval(() => {
                if (timeRemaining > 0) {
                    timeRemaining -= 1;

                    // Check if timer just completed
                    if (timeRemaining === 0) {
                        timerComplete();
                    }
                } else {
                    stopTimer();
                }
            }, 1000);

            // Separate timer for smooth animation
            animationTimer = setInterval(() => {
                // Smoothly animate towards the target percentage
                const diff = targetPercentage - animationPercentage;
                if (Math.abs(diff) > 0.1) {
                    animationPercentage += diff / (ANIMATION_STEPS / 4);
                }
            }, ANIMATION_INTERVAL);
        }
    }

    // Timer complete celebration
    function timerComplete() {
        isCompleted = true;
        stopTimer();

        // Emit completed event for mission tracking
        dispatch("completed");

        // Trigger confetti celebration
        const duration = 3 * 1000;
        const animationEnd = Date.now() + duration;
        const defaults = {
            startVelocity: 30,
            spread: 360,
            ticks: 60,
            zIndex: 0,
        };

        const randomInRange = (min: number, max: number) => {
            return Math.random() * (max - min) + min;
        };

        const interval = setInterval(() => {
            const timeLeft = animationEnd - Date.now();

            if (timeLeft <= 0) {
                return clearInterval(interval);
            }

            const particleCount = 50 * (timeLeft / duration);

            // Since particles fall down, start a bit higher than random
            confetti({
                ...defaults,
                particleCount,
                origin: { x: randomInRange(0.1, 0.3), y: Math.random() - 0.2 },
            });
            confetti({
                ...defaults,
                particleCount,
                origin: { x: randomInRange(0.7, 0.9), y: Math.random() - 0.2 },
            });
        }, 250);
    }

    // Stop the timer
    function stopTimer() {
        if (isRunning) {
            if (timer) {
                clearInterval(timer);
                timer = null;
            }
            if (animationTimer) {
                clearInterval(animationTimer);
                animationTimer = null;
            }

            isRunning = false;
        }
    }

    // Reset the timer
    function resetTimer() {
        stopTimer();
        timeRemaining = totalTime;
        animationPercentage = 0;
        isCompleted = false;
    }

    // Set timer duration in minutes
    function setTimerDuration(minutes: number) {
        if (!isRunning) {
            timeRemaining = minutes * 60;
            totalTime = minutes * 60;
            animationPercentage = 0;
            isCompleted = false;
        }
    }

    // Update animation percentage when target changes and not running
    $effect(() => {
        if (
            !isRunning &&
            Math.abs(targetPercentage - animationPercentage) > 0.1
        ) {
            animationPercentage = targetPercentage;
        }
    });

    // Clean up on component destroy
    onDestroy(() => {
        if (timer) {
            clearInterval(timer);
        }
        if (animationTimer) {
            clearInterval(animationTimer);
        }
    });
</script>

<div class="flex flex-col items-center justify-center min-h-[60vh] p-4 gap-y-9">
    <div class="flex justify-center gap-4 w-full mb-6">
        <button
            class="btn btn-lg {!isCompleted ? 'btn-primary' : ''}"
            onclick={() => setTimerDuration(25)}
        >
            Pomodoro
        </button>
        <button
            class="btn btn-lg {!isCompleted ? 'btn-primary' : ''}"
            onclick={() => setTimerDuration(5)}
        >
            Short Break
        </button>
        <button
            class="btn btn-lg {!isCompleted ? 'btn-primary' : ''}"
            onclick={() => setTimerDuration(15)}
        >
            Long Break
        </button>
    </div>

    <div class="relative mb-8">
        <div
            class="radial-progress flex items-center justify-center bg-base-200 border-4 border-base-200"
            style="--value:{animationPercentage}; --size:32rem; --thickness: 4rem;"
            aria-valuenow={animationPercentage}
            role="progressbar"
        >
            <div class="absolute inset-0 flex items-center justify-center">
                <span class="text-4xl font-bold">{formattedTime}</span>
            </div>
        </div>
    </div>

    <div class="flex justify-center gap-4 w-full">
        {#if isRunning}
            <button class="btn btn-lg btn-warning" onclick={stopTimer}
                >Pause</button
            >
        {:else}
            <button class="btn btn-lg btn-primary" onclick={startTimer}>
                {timeRemaining < totalTime && timeRemaining > 0
                    ? "Resume"
                    : "Start"}
            </button>
        {/if}
        <button class="btn btn-lg btn-accent" onclick={resetTimer}>Reset</button
        >
    </div>
</div>
