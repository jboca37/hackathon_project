<script lang="ts">
    import { onMount } from "svelte";
    import confetti from "canvas-confetti";

    // Check if we're in a browser environment
    const isBrowser = typeof window !== "undefined";

    // Types for our mission system
    interface Mission {
        id: string;
        title: string;
        description: string;
        points: number;
        completed: boolean;
        reset: "daily" | "weekly" | "never";
    }

    interface UserStats {
        points: number;
        streak: number;
        lastLogin: string;
        completedMissions: string[];
    }

    // Local storage keys
    const MISSIONS_KEY = "todo-app-missions";
    const USER_STATS_KEY = "todo-app-user-stats";

    // Initialize missions
    let missions = $state<Mission[]>([
        {
            id: "complete-pomodoro",
            title: "Focus Master",
            description: "Complete one Pomodoro session",
            points: 10,
            completed: false,
            reset: "daily",
        },
        {
            id: "complete-todos",
            title: "Task Clearer",
            description: "Complete all your todos",
            points: 15,
            completed: false,
            reset: "daily",
        },
        {
            id: "add-3-todos",
            title: "Planner",
            description: "Add at least 3 todos to your list",
            points: 5,
            completed: false,
            reset: "daily",
        },
        {
            id: "complete-3-habits",
            title: "Habit Builder",
            description: "Complete at least 3 habits today",
            points: 15,
            completed: false,
            reset: "daily",
        },
        {
            id: "habit-streak-3",
            title: "Consistency King",
            description: "Get a 3-day streak on any habit",
            points: 20,
            completed: false,
            reset: "never",
        },
        {
            id: "complete-all-habits",
            title: "Perfect Day",
            description: "Complete all your habits for the day",
            points: 25,
            completed: false,
            reset: "daily",
        },
        {
            id: "login-streak",
            title: "Consistency",
            description: "Log in 5 days in a row",
            points: 25,
            completed: false,
            reset: "never",
        },
    ]);

    // User stats
    let userStats = $state<UserStats>({
        points: 0,
        streak: 0,
        lastLogin: "",
        completedMissions: [],
    });

    // Debug mode state
    let debugMode = $state(false);

    // Load data from local storage on mount
    onMount(() => {
        if (isBrowser) {
            console.log("Mounting Missions component");
            loadUserData();
            checkDailyLogin();
        }
    });

    // Load user data from local storage
    function loadUserData() {
        try {
            if (isBrowser) {
                console.log("Loading user data from localStorage");
                const savedMissions = localStorage.getItem(MISSIONS_KEY);
                const savedStats = localStorage.getItem(USER_STATS_KEY);

                if (savedMissions) {
                    missions = JSON.parse(savedMissions);
                    console.log("Loaded missions:", missions);
                }

                if (savedStats) {
                    userStats = JSON.parse(savedStats);
                    console.log("Loaded user stats:", userStats);
                }
            }
        } catch (error) {
            console.error("Error loading user data:", error);
        }
    }

    // Save user data to local storage
    function saveUserData() {
        try {
            if (isBrowser) {
                console.log("Saving user data to localStorage");
                console.log("Saving missions:", missions);
                console.log("Saving stats:", userStats);
                localStorage.setItem(MISSIONS_KEY, JSON.stringify(missions));
                localStorage.setItem(USER_STATS_KEY, JSON.stringify(userStats));
            }
        } catch (error) {
            console.error("Error saving user data:", error);
        }
    }

    // Check daily login and update streak
    function checkDailyLogin() {
        if (!isBrowser) return;

        const today = new Date().toISOString().split("T")[0];
        console.log(
            "Checking daily login, today:",
            today,
            "last login:",
            userStats.lastLogin,
        );

        if (userStats.lastLogin !== today) {
            // It's a new day
            if (isConsecutiveDay(userStats.lastLogin)) {
                userStats.streak++;
                console.log("Consecutive day! New streak:", userStats.streak);

                // Check if the streak mission should be completed
                if (
                    userStats.streak >= 5 &&
                    !isMissionCompleted("login-streak")
                ) {
                    completeMission("login-streak");
                }
            } else if (userStats.lastLogin !== "") {
                // Broke the streak (but not first time login)
                userStats.streak = 1;
                console.log("Broke streak or first login. Reset to 1.");
            } else {
                // First time login
                userStats.streak = 1;
                console.log("First login. Set streak to 1.");
            }

            // Update last login date
            userStats.lastLogin = today;

            // Reset daily missions
            resetDailyMissions();

            // Save updated data
            saveUserData();
        }
    }

    // Check if the last login is the previous day (to maintain streak)
    function isConsecutiveDay(lastLogin: string): boolean {
        if (!lastLogin) return false;

        const lastDate = new Date(lastLogin);
        const today = new Date();

        // Set hours to 0 to compare just the dates
        lastDate.setHours(0, 0, 0, 0);
        today.setHours(0, 0, 0, 0);

        // Calculate difference in days
        const diffTime = today.getTime() - lastDate.getTime();
        const diffDays = diffTime / (1000 * 60 * 60 * 24);

        return diffDays === 1;
    }

    // Reset daily missions
    function resetDailyMissions() {
        missions = missions.map((mission) => {
            if (mission.reset === "daily") {
                return { ...mission, completed: false };
            }
            return mission;
        });
    }

    // Check if a mission is already completed
    function isMissionCompleted(missionId: string): boolean {
        return userStats.completedMissions.includes(missionId);
    }

    // Complete a mission and award points
    function completeMission(missionId: string) {
        // Find the mission
        const mission = missions.find((m) => m.id === missionId);
        console.log("Attempting to complete mission:", missionId, mission);

        if (mission && !mission.completed) {
            console.log("Completing mission:", mission.title);

            // Mark as completed
            missions = missions.map((m) =>
                m.id === missionId ? { ...m, completed: true } : m,
            );

            // Add to completed missions list if it's a one-time mission
            if (mission.reset === "never") {
                userStats.completedMissions = [
                    ...userStats.completedMissions,
                    missionId,
                ];
            }

            // Award points
            userStats.points += mission.points;
            console.log(
                `Awarded ${mission.points} points. New total: ${userStats.points}`,
            );

            // Trigger confetti
            triggerConfetti();

            // Save data
            saveUserData();
        } else {
            console.log(`Mission ${missionId} not found or already completed`);
        }
    }

    // Trigger confetti celebration
    function triggerConfetti() {
        if (isBrowser) {
            confetti({
                particleCount: 100,
                spread: 70,
                origin: { y: 0.6 },
            });
        }
    }

    // External method to be called from other components
    export function completePomodoro() {
        console.log("Pomodoro completed! Triggering mission completion");
        completeMission("complete-pomodoro");
    }

    export function checkTodoCompletion(completed: number, total: number) {
        console.log(`Todo completion check: ${completed}/${total} completed`);
        if (total > 0 && completed === total) {
            completeMission("complete-todos");
        }
    }

    export function checkAddedTodos(total: number) {
        console.log(`Checking todos added: ${total} todos`);
        if (total >= 3) {
            completeMission("add-3-todos");
        }
    }

    // Check habit completion for missions
    export function checkHabitCompletion(completed: number, total: number) {
        console.log(`Checking habit completion: ${completed}/${total} completed`);
        
        // Check if completed at least 3 habits
        if (completed >= 3) {
            completeMission("complete-3-habits");
        }
        
        // Check if completed all habits (when there's at least one habit)
        if (total > 0 && completed === total) {
            completeMission("complete-all-habits");
        }
        
        // For the streak mission, we'll check directly in localStorage
        // since the habit streaks are stored in the habits themselves
        checkHabitStreaks();
    }
    
    // Check for any habits with streaks of 3 or more
    function checkHabitStreaks() {
        try {
            if (isBrowser) {
                const habitsData = localStorage.getItem("todo-app-habits");
                
                if (habitsData) {
                    const habits = JSON.parse(habitsData);
                    
                    // Check if any habit has a streak of 3 or more
                    const hasStreakOfThree = habits.some((habit: { streak: number }) => habit.streak >= 3);
                    
                    if (hasStreakOfThree) {
                        completeMission("habit-streak-3");
                    }
                }
            }
        } catch (error) {
            console.error("Error checking habit streaks:", error);
        }
    }

    // Debug functions
    function toggleDebugMode() {
        debugMode = !debugMode;
    }

    function addDebugPoints(amount: number) {
        if (isBrowser) {
            userStats.points += amount;
            console.log(
                `Added ${amount} debug points. New total: ${userStats.points}`,
            );
            saveUserData();
            triggerConfetti();
        }
    }
</script>

<div class="flex flex-col p-4 bg-base-100 rounded-lg shadow-lg max-w-md w-full">
    <h2 class="text-2xl font-bold mb-4 text-base-content">Daily Missions</h2>

    <!-- Streak Display -->
    <div class="mb-4">
        <h3 class="text-lg font-semibold">Streak: {userStats.streak} days</h3>
        <!-- Streak Visualization - Simple circles for now -->
        <div class="flex space-x-1 mt-2">
            {#each Array(7) as _, i}
                <div
                    class={`h-3 w-3 rounded-full ${i < userStats.streak ? "bg-primary" : "bg-base-300"}`}
                ></div>
            {/each}
        </div>
    </div>

    <!-- Points Display -->
    <div class="mb-6 flex items-center">
        <span class="text-lg font-bold">{userStats.points}</span>
        <span class="ml-2 text-sm">points</span>
        <button class="btn btn-xs ml-4" onclick={toggleDebugMode}>
            {debugMode ? "Hide Debug" : "Debug Mode"}
        </button>
    </div>

    <!-- Debug Controls (only visible in debug mode) -->
    {#if debugMode}
        <div class="mb-6 p-3 bg-warning/20 rounded-lg">
            <h3 class="font-medium mb-2">Debug Controls</h3>
            <div class="flex space-x-2">
                <button
                    class="btn btn-sm btn-success"
                    onclick={() => addDebugPoints(10)}
                >
                    +10 Points
                </button>
                <button
                    class="btn btn-sm btn-success"
                    onclick={() => addDebugPoints(50)}
                >
                    +50 Points
                </button>
                <button
                    class="btn btn-sm btn-success"
                    onclick={() => addDebugPoints(100)}
                >
                    +100 Points
                </button>
            </div>
        </div>
    {/if}

    <!-- Missions List -->
    <div class="space-y-3">
        {#each missions as mission (mission.id)}
            <div class="flex items-center gap-3 bg-base-200 p-3 rounded-lg">
                <input
                    type="checkbox"
                    checked={mission.completed}
                    disabled={true}
                    class="checkbox checkbox-primary"
                />
                <div class="flex-grow">
                    <h4
                        class={mission.completed
                            ? "font-medium line-through text-base-content/50"
                            : "font-medium"}
                    >
                        {mission.title}
                    </h4>
                    <p class="text-sm text-base-content/70">
                        {mission.description}
                    </p>
                </div>
                <span class="badge badge-primary">+{mission.points}</span>
            </div>
        {/each}
    </div>
</div>
