<script lang="ts">
    import TodoList from "./TodoList.svelte";
    import Pomodoro from "./Pomodoro.svelte";
    import Missions from "./Missions.svelte";
    import Garden from "./Garden.svelte";
    import HabitTracker from "./HabitTracker.svelte";
    import { onMount } from "svelte";

    // References to components for cross-component communication with proper typing and reactivity
    let todoListComponent = $state<TodoList | null>(null);
    let pomodoroComponent = $state<Pomodoro | null>(null);
    let missionsComponent = $state<Missions | null>(null);
    let habitTrackerComponent = $state<HabitTracker | null>(null);

    // Active tab state
    let activeTab = $state("todo");

    // Function to handle when a Pomodoro is completed
    function handlePomodoroComplete() {
        if (missionsComponent) {
            missionsComponent.completePomodoro();
        }
    }

    // Function to handle todo list updates (for missions)
    function handleTodoListUpdate(
        e: CustomEvent<{ completed: number; total: number }>,
    ) {
        if (missionsComponent) {
            const { completed, total } = e.detail;
            missionsComponent.checkTodoCompletion(completed, total);
        }
    }

    // Function to handle when todos are added
    function handleTodosAdded(e: CustomEvent<{ total: number }>) {
        if (missionsComponent) {
            const { total } = e.detail;
            missionsComponent.checkAddedTodos(total);
        }
    }

    // Function to handle habit stats updates
    function handleHabitStats(
        e: CustomEvent<{ completed: number; total: number }>,
    ) {
        if (missionsComponent) {
            const { completed, total } = e.detail;
            missionsComponent.checkHabitCompletion(completed, total);
        }
    }

    // Tab switching functions
    function switchToTodo() {
        activeTab = "todo";
    }

    function switchToHabits() {
        activeTab = "habits";
    }

    function switchToMissions() {
        activeTab = "missions";
    }

    function switchToGarden() {
        activeTab = "garden";
    }
</script>

<div class="container mx-auto p-4">
    <header class="mb-8">
        <h1 class="text-3xl font-bold text-primary">Productivity Garden</h1>
        <p class="text-base-content/70">
            Stay productive, earn points, grow your garden!
        </p>
    </header>

    <!-- Tab Navigation -->
    <div class="tabs tabs-boxed mb-6">
        <button
            class={`tab ${activeTab === "todo" ? "tab-active" : ""}`}
            onclick={switchToTodo}
        >
            Tasks & Focus
        </button>
        <button
            class={`tab ${activeTab === "habits" ? "tab-active" : ""}`}
            onclick={switchToHabits}
        >
            Habits
        </button>
        <button
            class={`tab ${activeTab === "missions" ? "tab-active" : ""}`}
            onclick={switchToMissions}
        >
            Missions
        </button>
        <button
            class={`tab ${activeTab === "garden" ? "tab-active" : ""}`}
            onclick={switchToGarden}
        >
            Garden
        </button>
    </div>

    <!-- Content based on active tab -->
    <div class="flex justify-center">
        {#if activeTab === "todo"}
            <div class="flex flex-row gap-8 justify-center items-start">
                <Pomodoro
                    bind:this={pomodoroComponent}
                    on:completed={handlePomodoroComplete}
                />
                <TodoList
                    bind:this={todoListComponent}
                    on:listUpdated={handleTodoListUpdate}
                    on:todosAdded={handleTodosAdded}
                />
            </div>
        {:else if activeTab === "habits"}
            <HabitTracker 
                bind:this={habitTrackerComponent}
                on:habitStats={handleHabitStats}
            />
        {:else if activeTab === "missions"}
            <Missions bind:this={missionsComponent} />
        {:else if activeTab === "garden"}
            <Garden />
        {/if}
    </div>
</div>
