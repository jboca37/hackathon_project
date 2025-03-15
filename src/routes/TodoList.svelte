<script lang="ts">
    import confetti from "canvas-confetti";
    import { createEventDispatcher, onMount } from "svelte";

    // Check if we're in a browser environment
    const isBrowser = typeof window !== "undefined";

    // Create event dispatcher
    const dispatch = createEventDispatcher();

    // Todo type definition
    interface Todo {
        id: number;
        task: string;
        completed: boolean;
    }

    // Initialize todo list with $state
    let todoList = $state<Todo[]>([]);

    // Form input value
    let newTask = $state("");

    // Track mouse position for confetti
    let mouseX = $state(0);
    let mouseY = $state(0);

    // Track mouse movement globally
    function handleMouseMove(e: MouseEvent) {
        mouseX = e.clientX;
        mouseY = e.clientY;
    }

    // Generate unique ID for todos
    let nextId = $state(1);

    // State to track which todo is being edited
    let editingId = $state<number | null>(null);
    let editText = $state("");

    // Local storage key
    const TODOS_KEY = "todo-app-todos";
    const NEXT_ID_KEY = "todo-app-next-id";

    // Load todos from localStorage on mount
    onMount(() => {
        if (isBrowser) {
            console.log("Mounting TodoList component");
            loadTodos();

            // Emit initial events for mission tracking
            setTimeout(() => {
                emitListUpdatedEvent();
                emitTodosAddedEvent();
            }, 500);
        }
    });

    // Load todos from localStorage
    function loadTodos() {
        try {
            if (isBrowser) {
                console.log("Loading todos from localStorage");
                const savedTodos = localStorage.getItem(TODOS_KEY);
                const savedNextId = localStorage.getItem(NEXT_ID_KEY);

                if (savedTodos) {
                    todoList = JSON.parse(savedTodos);
                    console.log("Loaded todos:", todoList);
                }

                if (savedNextId) {
                    nextId = JSON.parse(savedNextId);
                }
            }
        } catch (error) {
            console.error("Error loading todos:", error);
        }
    }

    // Save todos to localStorage
    function saveTodos() {
        try {
            if (isBrowser) {
                console.log("Saving todos to localStorage");
                localStorage.setItem(TODOS_KEY, JSON.stringify(todoList));
                localStorage.setItem(NEXT_ID_KEY, JSON.stringify(nextId));
            }
        } catch (error) {
            console.error("Error saving todos:", error);
        }
    }

    // Add a new todo
    function addTodo() {
        if (newTask.trim() !== "") {
            console.log("Adding new todo:", newTask);
            todoList = [
                ...todoList,
                {
                    id: nextId,
                    task: newTask,
                    completed: false,
                },
            ];
            nextId++;
            newTask = ""; // Clear input after adding

            // Save to localStorage
            saveTodos();

            // Emit event for mission tracking
            emitTodosAddedEvent();
        }
    }

    // Toggle todo completion status
    function toggleCompleted(id: number) {
        const todo = todoList.find((t) => t.id === id);
        const wasCompleted = todo?.completed;

        console.log(`Toggling completion for todo ${id}`);
        todoList = todoList.map((todo) =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo,
        );

        // Save to localStorage
        saveTodos();

        // Celebrate with confetti when a task is newly marked as completed
        if (!wasCompleted) {
            celebrateCompletion();
        }

        // Emit event for mission tracking
        emitListUpdatedEvent();
    }

    // Function to trigger confetti
    function celebrateCompletion() {
        if (!isBrowser) return;

        // Convert mouse position to relative coordinates (0-1)
        const x = mouseX / window.innerWidth;
        const y = mouseY / window.innerHeight;

        // Create a smaller, more subtle confetti effect at cursor position
        confetti({
            particleCount: 80,
            spread: 40,
            origin: { y, x },
            colors: ["#4CAF50", "#8BC34A", "#CDDC39", "#FFEB3B"],
            angle: 90,
            startVelocity: 25,
            gravity: 0.8,
            zIndex: 1000,
            scalar: 0.7,
            disableForReducedMotion: true,
        });
    }

    // Delete a todo
    function deleteTodo(id: number) {
        console.log(`Deleting todo ${id}`);
        todoList = todoList.filter((todo) => todo.id !== id);

        // Save to localStorage
        saveTodos();

        // Emit event for mission tracking
        emitListUpdatedEvent();
    }

    // Edit a todo
    function editTodo(id: number, newText: string) {
        if (newText.trim() !== "") {
            console.log(`Editing todo ${id}`);
            todoList = todoList.map((todo) =>
                todo.id === id ? { ...todo, task: newText } : todo,
            );

            // Save to localStorage
            saveTodos();

            // Emit event for mission tracking
            emitListUpdatedEvent();
        }
    }

    // Start editing a todo
    function startEdit(todo: Todo) {
        editingId = todo.id;
        editText = todo.task;
    }

    // Save edits
    function saveEdit() {
        if (editingId !== null) {
            editTodo(editingId, editText);
            editingId = null;
        }
    }

    // Cancel editing
    function cancelEdit() {
        editingId = null;
    }

    // Emit event for todo list updates
    function emitListUpdatedEvent() {
        const completedCount = todoList.filter((t) => t.completed).length;
        const totalCount = todoList.length;

        console.log(
            `Emitting listUpdated event: ${completedCount}/${totalCount} completed`,
        );
        dispatch("listUpdated", {
            completed: completedCount,
            total: totalCount,
        });
    }

    // Emit event for added todos
    function emitTodosAddedEvent() {
        console.log(`Emitting todosAdded event: ${todoList.length} todos`);
        dispatch("todosAdded", {
            total: todoList.length,
        });
    }

    // Derived state for task summary
    let completedCount = $derived(todoList.filter((t) => t.completed).length);
    let totalCount = $derived(todoList.length);
</script>

<div
    class="flex flex-col p-4 bg-base-100 rounded-lg shadow-lg max-w-md w-full"
    onmousemove={handleMouseMove}
    role="region"
    aria-label="Todo list container"
>
    <h2 class="text-2xl font-bold mb-6 text-base-content/70">
        {new Date().toLocaleDateString("en-US", {
            weekday: "long",
            day: "numeric",
            month: "short",
        })}
    </h2>

    <!-- Task Summary -->
    {#if todoList.length > 0}
        <div class="mb-6 text-sm text-base-content/70">
            <p>
                {completedCount} of {totalCount} tasks completed
            </p>
        </div>
    {/if}

    <!-- Todo List -->
    <div class="space-y-3">
        {#if todoList.length === 0}
            <p class="text-center text-base-content/70">
                No tasks yet. Add one Below!
            </p>
        {:else}
            {#each todoList as todo (todo.id)}
                <div class="flex flex-wrap items-center gap-3 bg-base-200 p-3 rounded-lg">
                    {#if editingId === todo.id}
                        <!-- Edit Mode -->
                        <input
                            bind:value={editText}
                            class="input input-bordered flex-grow"
                            onkeydown={(e) => e.key === "Enter" && saveEdit()}
                        />
                        <div class="flex gap-2 mt-2 sm:mt-0">
                            <button
                                class="btn btn-sm btn-success"
                                onclick={saveEdit}
                            >
                                Save
                            </button>
                            <button
                                class="btn btn-sm btn-ghost"
                                onclick={cancelEdit}
                            >
                                Cancel
                            </button>
                        </div>
                    {:else}
                        <!-- View Mode -->
                        <div class="flex items-center gap-3 min-w-0 flex-grow">
                            <input
                                type="checkbox"
                                checked={todo.completed}
                                onchange={() => toggleCompleted(todo.id)}
                                class="checkbox checkbox-primary flex-shrink-0"
                            />

                            <span
                                class={todo.completed
                                    ? "line-through flex-grow text-base-content/50 break-words overflow-hidden"
                                    : "flex-grow break-words overflow-hidden"}
                                style="min-width: 0; word-wrap: break-word;"
                            >
                                {todo.task}
                            </span>
                        </div>

                        <div class="flex gap-2 ml-auto">
                            <button
                                class="btn btn-sm btn-ghost flex-shrink-0"
                                onclick={() => startEdit(todo)}
                            >
                                Edit
                            </button>

                            <button
                                class="btn btn-sm btn-error btn-ghost flex-shrink-0"
                                onclick={() => deleteTodo(todo.id)}
                            >
                                Delete
                            </button>
                        </div>
                    {/if}
                </div>
            {/each}
        {/if}
    </div>

    <!-- Add Todo Form -->
    <form
        onsubmit={(e) => {
            e.preventDefault();
            addTodo();
        }}
        class="mt-6"
    >
        <div class="flex gap-2">
            <input
                bind:value={newTask}
                type="text"
                placeholder="Add a task"
                class="input input-bordered flex-grow"
            />
            <button type="submit" class="btn btn-primary">Add</button>
        </div>
    </form>
</div>
