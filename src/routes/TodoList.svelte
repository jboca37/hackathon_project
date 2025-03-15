<script lang="ts">
    import confetti from "canvas-confetti";

    // Todo type definition
    interface Todo {
        id: number;
        task: string;
        completed: boolean;
    }

    // Initialize todo list with $state instead of writable store
    let todoList = $state<Todo[]>([]);

    // Form input value
    let newTask = $state("");
    
    // Track mouse position for confetti
    let mouseX = $state(0);
    let mouseY = $state(0);

    // Update mouse position on mouse move
    function updateMousePosition(e: MouseEvent) {
        mouseX = e.clientX;
        mouseY = e.clientY;
    }

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

    // Add a new todo
    function addTodo() {
        if (newTask.trim() !== "") {
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
        }
    }

    // Toggle todo completion status
    function toggleCompleted(id: number) {
        const todo = todoList.find((t) => t.id === id);
        const wasCompleted = todo?.completed;

        todoList = todoList.map((todo) =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo,
        );

        // Celebrate with confetti when a task is newly marked as completed
        if (!wasCompleted) {
            celebrateCompletion();
        }
    }

    // Function to trigger confetti
    function celebrateCompletion() {
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
        todoList = todoList.filter((todo) => todo.id !== id);
    }

    // Edit a todo
    function editTodo(id: number, newText: string) {
        if (newText.trim() !== "") {
            todoList = todoList.map((todo) =>
                todo.id === id ? { ...todo, task: newText } : todo,
            );
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

    // Derived state for task summary
    const completedCount = $derived(todoList.filter((t) => t.completed).length);
    const totalCount = $derived(todoList.length);
</script>

<div class="flex flex-col p-4 bg-base-100 rounded-lg shadow-lg max-w-md w-full"
    onmousemove={handleMouseMove}
    role="region"
    aria-label="Todo list container">
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
                <div class="flex items-center gap-3 bg-base-200 p-3 rounded-lg">
                    {#if editingId === todo.id}
                        <!-- Edit Mode -->
                        <input
                            bind:value={editText}
                            class="input input-bordered flex-grow"
                            onkeydown={(e) => e.key === "Enter" && saveEdit()}
                        />
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
                    {:else}
                        <!-- View Mode -->
                        <input
                            type="checkbox"
                            checked={todo.completed}
                            onchange={() => toggleCompleted(todo.id)}
                            class="checkbox checkbox-primary"
                        />

                        <span
                            class={todo.completed
                                ? "line-through flex-grow text-base-content/50"
                                : "flex-grow"}
                        >
                            {todo.task}
                        </span>

                        <button
                            class="btn btn-sm btn-ghost"
                            onclick={() => startEdit(todo)}
                        >
                            Edit
                        </button>

                        <button
                            class="btn btn-sm btn-error btn-ghost"
                            onclick={() => deleteTodo(todo.id)}
                        >
                            Delete
                        </button>
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
