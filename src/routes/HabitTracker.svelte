<script lang="ts">
    import { onMount, createEventDispatcher } from "svelte";
    import confetti from "canvas-confetti";

    // Check if we're in a browser environment
    const isBrowser = typeof window !== "undefined";

    // Create event dispatcher for mission integration
    const dispatch = createEventDispatcher();

    // Habit type definition
    interface Habit {
        id: string;
        name: string;
        description: string;
        emoji: string;
        frequency: "daily" | "weekly";
        completed: boolean;
        streak: number;
        lastCompleted: string | null;
        pointsPerCompletion: number;
    }

    // Initialize habits with $state
    let habits = $state<Habit[]>([
        {
            id: "water",
            name: "Drink Water",
            description: "Drink at least 8 glasses of water today",
            emoji: "💧",
            frequency: "daily",
            completed: false,
            streak: 0,
            lastCompleted: null,
            pointsPerCompletion: 5
        },
        {
            id: "exercise",
            name: "Exercise",
            description: "Get at least 30 minutes of physical activity",
            emoji: "🏃",
            frequency: "daily",
            completed: false,
            streak: 0,
            lastCompleted: null,
            pointsPerCompletion: 10
        },
        {
            id: "meditate",
            name: "Meditate",
            description: "Practice mindfulness for at least 10 minutes",
            emoji: "🧘",
            frequency: "daily",
            completed: false,
            streak: 0,
            lastCompleted: null,
            pointsPerCompletion: 8
        },
        {
            id: "read",
            name: "Read",
            description: "Read a book for at least 20 minutes",
            emoji: "📚",
            frequency: "daily",
            completed: false,
            streak: 0,
            lastCompleted: null,
            pointsPerCompletion: 7
        },
        {
            id: "sleep",
            name: "Early Sleep",
            description: "Go to bed before 11 PM",
            emoji: "😴",
            frequency: "daily",
            completed: false,
            streak: 0,
            lastCompleted: null,
            pointsPerCompletion: 8
        }
    ]);

    // State for adding new habits
    let isAddingHabit = $state(false);
    let newHabitName = $state("");
    let newHabitDescription = $state("");
    let newHabitEmoji = $state("✨");
    let newHabitFrequency = $state<"daily" | "weekly">("daily");

    // Common emojis for habits
    const commonEmojis = [
        "✨", "💪", "🏃", "🧘", "📚", "💧", "🥗", "🍎", "😴", "🌱", 
        "📝", "🎯", "⏰", "🧠", "🎨", "🎵", "🚶", "🧹", "🥦", "💤"
    ];

    // Fixed point values based on frequency
    const getPointsForFrequency = (frequency: "daily" | "weekly"): number => {
        return frequency === "daily" ? 5 : 10;
    };

    // Local storage key
    const HABITS_KEY = "todo-app-habits";
    const USER_STATS_KEY = "todo-app-user-stats";

    // Total habits completed today
    let completedToday = $derived(habits.filter(h => h.completed).length);
    let totalHabits = $derived(habits.length);

    // Load habits from localStorage on mount
    onMount(() => {
        if (isBrowser) {
            console.log("Mounting HabitTracker component");
            loadHabits();
            checkDailyReset();
            
            // Emit initial events for mission tracking
            setTimeout(() => {
                emitHabitStats();
            }, 500);
        }
    });

    // Load habits from localStorage
    function loadHabits() {
        try {
            if (isBrowser) {
                console.log("Loading habits from localStorage");
                const savedHabits = localStorage.getItem(HABITS_KEY);

                if (savedHabits) {
                    habits = JSON.parse(savedHabits);
                    console.log("Loaded habits:", habits);
                }
            }
        } catch (error) {
            console.error("Error loading habits:", error);
        }
    }

    // Save habits to localStorage
    function saveHabits() {
        try {
            if (isBrowser) {
                console.log("Saving habits to localStorage");
                localStorage.setItem(HABITS_KEY, JSON.stringify(habits));
            }
        } catch (error) {
            console.error("Error saving habits:", error);
        }
    }

    // Check if habits need to be reset for the day
    function checkDailyReset() {
        if (!isBrowser) return;

        const today = new Date().toISOString().split("T")[0];
        let resetNeeded = false;

        // Check if any habits need to be reset
        habits.forEach(habit => {
            if (habit.frequency === "daily" && habit.lastCompleted && habit.lastCompleted !== today) {
                resetNeeded = true;
            }
        });

        if (resetNeeded) {
            resetDailyHabits();
        }
    }

    // Reset daily habits
    function resetDailyHabits() {
        habits = habits.map(habit => {
            if (habit.frequency === "daily") {
                return {
                    ...habit,
                    completed: false
                    // Keep streak value intact
                };
            }
            return habit;
        });

        saveHabits();
    }

    // Toggle habit completion
    function toggleHabit(id: string) {
        const today = new Date().toISOString().split("T")[0];
        const habitIndex = habits.findIndex(h => h.id === id);
        
        if (habitIndex === -1) return;
        
        const habit = habits[habitIndex];
        const wasCompleted = habit.completed;

        // If marking as complete (checking the habit)
        if (!wasCompleted) {
            // Increase streak by 1
            const newStreak = habit.streak + 1;
            
            // Update the habit
            habits = habits.map(h => {
                if (h.id === id) {
                    return {
                        ...h,
                        completed: true,
                        lastCompleted: today,
                        streak: newStreak
                    };
                }
                return h;
            });
            
            // Award points
            awardPoints(habit.pointsPerCompletion);
            
            // Celebrate with appropriate confetti based on streak milestone
            if (newStreak === 3) {
                triggerConfetti('streak3', habit.emoji);
            } else if (newStreak === 7) {
                triggerConfetti('streak7', habit.emoji);
            } else if (newStreak === 14) {
                triggerConfetti('streak14', habit.emoji);
            } else if (newStreak === 30) {
                triggerConfetti('streak30', habit.emoji);
            } else if (newStreak === 100) {
                triggerConfetti('streak100', habit.emoji);
            } else {
                triggerConfetti('normal', habit.emoji);
            }
        } else {
            // If unchecking, decrease streak by 1 (minimum 0)
            const decreasedStreak = Math.max(0, habit.streak - 1);
            
            habits = habits.map(h => {
                if (h.id === id) {
                    return {
                        ...h,
                        completed: false,
                        streak: decreasedStreak
                    };
                }
                return h;
            });
            
            // Deduct points
            awardPoints(-habit.pointsPerCompletion);
        }
        
        // Emit event for mission tracking
        emitHabitStats();

        // Save changes
        saveHabits();
    }

    // Check if a date is consecutive to today
    function isConsecutiveDay(dateString: string): boolean {
        if (!dateString) return false;

        const date = new Date(dateString);
        const today = new Date();
        
        // Reset time parts for comparison
        date.setHours(0, 0, 0, 0);
        today.setHours(0, 0, 0, 0);
        
        // Yesterday's date
        const yesterday = new Date(today);
        yesterday.setDate(yesterday.getDate() - 1);
        
        return date.getTime() === yesterday.getTime();
    }

    // Award points to user
    function awardPoints(points: number) {
        try {
            if (isBrowser) {
                const savedStats = localStorage.getItem(USER_STATS_KEY);
                
                if (savedStats) {
                    const stats = JSON.parse(savedStats);
                    stats.points += points;
                    localStorage.setItem(USER_STATS_KEY, JSON.stringify(stats));
                    console.log(`Awarded ${points} points. New total: ${stats.points}`);
                }
            }
        } catch (error) {
            console.error('Error updating points:', error);
        }
    }

    // Add a new habit
    function addHabit() {
        if (newHabitName.trim() === "") return;
        
        // Create a unique ID based on name and timestamp
        const id = newHabitName.toLowerCase().replace(/\s+/g, '-') + 
                  '-' + Date.now().toString().slice(-4);
        
        const newHabit: Habit = {
            id,
            name: newHabitName,
            description: newHabitDescription || `Complete your "${newHabitName}" habit`,
            emoji: newHabitEmoji,
            frequency: newHabitFrequency,
            completed: false,
            streak: 0,
            lastCompleted: null,
            pointsPerCompletion: getPointsForFrequency(newHabitFrequency)
        };
        
        habits = [...habits, newHabit];
        
        // Reset form
        newHabitName = "";
        newHabitDescription = "";
        newHabitEmoji = "✨";
        newHabitFrequency = "daily";
        isAddingHabit = false;
        
        // Save changes
        saveHabits();
        
        // Emit event for mission tracking
        emitHabitStats();
    }

    // Delete a habit
    function deleteHabit(id: string) {
        habits = habits.filter(h => h.id !== id);
        saveHabits();
        emitHabitStats();
    }

    // Trigger confetti celebration
    function triggerConfetti(type: 'normal' | 'streak3' | 'streak7' | 'streak14' | 'streak30' | 'streak100' = 'normal', emoji?: string) {
        if (isBrowser) {
            // Base confetti options
            const baseOptions = {
                particleCount: 60,
                spread: 70,
                origin: { y: 0.7 },
                colors: ['#43A047', '#66BB6A', '#81C784', '#A5D6A7']
            };
            
            // Show streak milestone toast if applicable
            if (type !== 'normal') {
                showStreakMilestone(type, emoji);
            }
            
            // Adjust confetti based on achievement type
            switch (type) {
                case 'streak3':
                    confetti({
                        ...baseOptions,
                        particleCount: 100,
                        gravity: 0.7
                    });
                    break;
                case 'streak7':
                    confetti({
                        ...baseOptions,
                        particleCount: 150,
                        gravity: 0.6
                    });
                    break;
                case 'streak14':
                    confetti({
                        ...baseOptions,
                        particleCount: 200,
                        gravity: 0.5
                    });
                    break;
                case 'streak30':
                    // Special 30-day celebration
                    const duration = 3 * 1000;
                    const animationEnd = Date.now() + duration;
                    
                    const interval = setInterval(() => {
                        const timeLeft = animationEnd - Date.now();
                        
                        if (timeLeft <= 0) {
                            return clearInterval(interval);
                        }
                        
                        confetti({
                            ...baseOptions,
                            particleCount: 150 * (timeLeft / duration),
                            gravity: 0.4,
                            origin: { 
                                x: Math.random(), 
                                y: Math.random() - 0.2 
                            }
                        });
                    }, 200);
                    break;
                case 'streak100':
                    // Epic 100-day celebration
                    const longDuration = 5 * 1000;
                    const longAnimationEnd = Date.now() + longDuration;
                    
                    const longInterval = setInterval(() => {
                        const timeLeft = longAnimationEnd - Date.now();
                        
                        if (timeLeft <= 0) {
                            return clearInterval(longInterval);
                        }
                        
                        confetti({
                            ...baseOptions,
                            particleCount: 200,
                            startVelocity: 30,
                            spread: 360,
                            gravity: 0.3,
                            ticks: 60,
                            origin: { 
                                x: 0.5, 
                                y: 0.5 
                            }
                        });
                    }, 150);
                    break;
                default:
                    confetti(baseOptions);
                    break;
            }
        }
    }
    
    // Show streak milestone toast notification
    function showStreakMilestone(type: string, emoji?: string) {
        // Create a toast element
        const toast = document.createElement('div');
        toast.className = 'fixed bottom-4 right-4 bg-accent text-accent-content p-4 rounded-lg shadow-lg z-50 animate-bounce';
        
        // Set milestone message
        let message = '';
        let streakCount = '';
        
        switch (type) {
            case 'streak3':
                streakCount = '3';
                message = 'Great start! Keep it up!';
                break;
            case 'streak7':
                streakCount = '7';
                message = 'One week streak! Amazing consistency!';
                break;
            case 'streak14':
                streakCount = '14';
                message = 'Two week streak! You\'re on fire!';
                break;
            case 'streak30':
                streakCount = '30';
                message = 'One month streak! Incredible discipline!';
                break;
            case 'streak100':
                streakCount = '100';
                message = 'WOW! 100 day streak! You\'re legendary!';
                break;
        }
        
        // Set toast content
        toast.innerHTML = `
            <div class="flex items-center gap-2">
                <span class="text-2xl">${emoji || '🔥'}</span>
                <div>
                    <div class="font-bold">${streakCount} Day Streak!</div>
                    <div class="text-sm">${message}</div>
                </div>
            </div>
        `;
        
        // Add to document
        document.body.appendChild(toast);
        
        // Remove after 5 seconds
        setTimeout(() => {
            toast.classList.add('opacity-0', 'transition-opacity', 'duration-500');
            setTimeout(() => toast.remove(), 500);
        }, 5000);
    }

    // Emit habit statistics for mission tracking
    function emitHabitStats() {
        console.log(`Emitting habitStats event: ${completedToday}/${totalHabits} completed`);
        dispatch("habitStats", {
            completed: completedToday,
            total: totalHabits
        });
    }
</script>

<div class="flex flex-col p-4 bg-base-100 rounded-lg shadow-lg max-w-md w-full">
    <h2 class="text-2xl font-bold mb-4 text-base-content">Habit Tracker</h2>

    <!-- Habit Stats -->
    <div class="mb-6 flex justify-between items-center">
        <div class="text-sm text-base-content/70">
            <p>{completedToday} of {totalHabits} habits completed today</p>
        </div>
        <button class="btn btn-sm btn-primary" onclick={() => isAddingHabit = true}>
            Add Habit
        </button>
    </div>

    <!-- Habit List -->
    <div class="space-y-4 mb-6">
        {#if habits.length === 0}
            <p class="text-center text-base-content/70 py-4">
                No habits yet. Add some habits to track!
            </p>
        {:else}
            {#each habits as habit (habit.id)}
                <div class="bg-base-200 p-4 rounded-lg">
                    <div class="flex items-start gap-3">
                        <input 
                            type="checkbox" 
                            checked={habit.completed}
                            class="checkbox checkbox-primary mt-1"
                            onchange={() => toggleHabit(habit.id)}
                        />
                        
                        <div class="flex-grow">
                            <div class="flex justify-between items-start">
                                <h3 class={habit.completed ? "font-medium line-through text-base-content/50" : "font-medium"}>
                                    <span class="text-xl mr-2">{habit.emoji}</span>
                                    {habit.name}
                                </h3>
                                <span class="badge badge-outline ml-2">+{habit.pointsPerCompletion}</span>
                            </div>
                            
                            <p class="text-sm text-base-content/70 mt-1 break-words">
                                {habit.description}
                            </p>
                            
                            <div class="flex mt-2 text-xs items-center">
                                <span class={habit.frequency === "daily" ? "badge badge-sm" : "badge badge-sm badge-secondary"}>
                                    {habit.frequency}
                                </span>
                                {#if habit.streak > 0}
                                    <span class="badge badge-sm badge-accent ml-2 flex items-center">
                                        <span class="mr-1">{habit.streak}</span>
                                        <span class="text-xs">🔥</span>
                                    </span>
                                    <span class="ml-1 text-xs text-accent-content/70">day streak</span>
                                {/if}
                            </div>
                        </div>
                        
                        <button 
                            class="btn btn-sm btn-ghost btn-circle"
                            onclick={() => deleteHabit(habit.id)}
                            title="Delete habit"
                        >
                            ✕
                        </button>
                    </div>
                </div>
            {/each}
        {/if}
    </div>

    <!-- Add Habit Form Modal -->
    {#if isAddingHabit}
        <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
            <div class="bg-base-100 p-6 rounded-lg w-full max-w-md">
                <h3 class="text-lg font-bold mb-4">Add New Habit</h3>
                
                <form onsubmit={(e) => { e.preventDefault(); addHabit(); }} class="space-y-4">
                    <div class="form-control">
                        <label class="label" for="habit-name">
                            <span class="label-text">Habit Name</span>
                        </label>
                        <input 
                            type="text" 
                            id="habit-name"
                            placeholder="Enter habit name" 
                            class="input input-bordered w-full" 
                            bind:value={newHabitName}
                            required
                        />
                    </div>
                    
                    <div class="form-control">
                        <label class="label" for="habit-description">
                            <span class="label-text">Description (Optional)</span>
                        </label>
                        <textarea 
                            id="habit-description"
                            placeholder="Describe your habit" 
                            class="textarea textarea-bordered h-24" 
                            bind:value={newHabitDescription}
                        ></textarea>
                    </div>
                    
                    <div class="form-control">
                        <label class="label" for="habit-emoji">
                            <span class="label-text">Emoji</span>
                        </label>
                        <select 
                            id="habit-emoji"
                            class="select select-bordered w-full" 
                            bind:value={newHabitEmoji}
                        >
                            {#each commonEmojis as emoji}
                                <option value={emoji}>{emoji}</option>
                            {/each}
                        </select>
                    </div>
                    
                    <div class="form-control">
                        <label class="label" for="habit-frequency">
                            <span class="label-text">Frequency</span>
                        </label>
                        <select 
                            id="habit-frequency"
                            class="select select-bordered w-full" 
                            bind:value={newHabitFrequency}
                        >
                            <option value="daily">Daily</option>
                            <option value="weekly">Weekly</option>
                        </select>
                    </div>
                    
                    <div class="flex justify-end gap-2 mt-6">
                        <button 
                            type="button" 
                            class="btn btn-ghost" 
                            onclick={() => isAddingHabit = false}
                        >
                            Cancel
                        </button>
                        <button type="submit" class="btn btn-primary">Add Habit</button>
                    </div>
                </form>
            </div>
        </div>
    {/if}
</div> 