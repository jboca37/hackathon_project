<script lang="ts">
    import { onMount } from "svelte";
    import confetti from "canvas-confetti";

    // Check if we're in a browser environment
    const isBrowser = typeof window !== 'undefined';

    // Types for our garden system
    interface Flower {
        id: string;
        name: string;
        cost: number;
        image: string;
        description: string;
        rarity: "common" | "uncommon" | "rare" | "legendary";
    }

    interface GardenTile {
        id: number;
        flowerId: string | null;
        plantedDate: string | null;
    }

    interface Garden {
        tiles: GardenTile[];
        unlockedFlowers: string[];
    }

    // Available flowers to plant
    const availableFlowers: Flower[] = [
        {
            id: "daisy",
            name: "Daisy",
            cost: 10,
            image: "🌼",
            description:
                "A simple and beautiful flower that brightens your garden.",
            rarity: "common",
        },
        {
            id: "tulip",
            name: "Tulip",
            cost: 20,
            image: "🌷",
            description: "A classic and elegant flower with vibrant colors.",
            rarity: "common",
        },
        {
            id: "rose",
            name: "Rose",
            cost: 30,
            image: "🌹",
            description: "A symbol of love and beauty.",
            rarity: "uncommon",
        },
        {
            id: "sunflower",
            name: "Sunflower",
            cost: 50,
            image: "🌻",
            description: "A tall and cheerful flower that follows the sun.",
            rarity: "uncommon",
        },
        {
            id: "lotus",
            name: "Lotus",
            cost: 75,
            image: "🪷",
            description:
                "A symbol of purity, enlightenment, self-regeneration, and rebirth.",
            rarity: "rare",
        },
        {
            id: "cherry-blossom",
            name: "Cherry Blossom",
            cost: 100,
            image: "🌸",
            description:
                "A delicate and beautiful flower that symbolizes the beauty of life.",
            rarity: "legendary",
        },
    ];

    // Local storage keys
    const GARDEN_KEY = "todo-app-garden";
    const USER_STATS_KEY = "todo-app-user-stats";

    // Garden data
    let garden = $state<Garden>({
        tiles: Array(16)
            .fill(null)
            .map((_, index) => ({
                id: index,
                flowerId: null,
                plantedDate: null,
            })),
        unlockedFlowers: ["daisy", "tulip", "rose", "sunflower", "lotus", "cherry-blossom"], // All flowers unlocked for testing
    });

    // User stats (imported from Missions component)
    let userPoints = $state(200);  // Start with 200 points, enough for all flowers

    // Selected flower for planting
    let selectedFlower = $state<string | null>(null);

    // UI states
    let isPlantingMode = $state(false);
    let showFlowerDetails = $state<string | null>(null);

    // Load data from local storage on mount
    onMount(() => {
        if (isBrowser) {
            console.log("Mounting Garden component");
            
            // Load garden data first
            loadGardenData();
            
            // Then load user points
            loadUserStats();
            
            // Only update garden flowers if they're not already unlocked
            const currentGarden = JSON.parse(localStorage.getItem(GARDEN_KEY) || '{}');
            if (currentGarden && currentGarden.unlockedFlowers) {
                if (currentGarden.unlockedFlowers.length < 3) {
                    // Ensure all flowers are unlocked without resetting other garden data
                    garden = {
                        ...currentGarden,
                        unlockedFlowers: ["daisy", "tulip", "rose", "sunflower", "lotus", "cherry-blossom"]
                    };
                    // Save updated garden with all flowers unlocked
                    saveGardenData();
                }
            }
            
            // Set up an interval to refresh points every 2 seconds
            const refreshInterval = setInterval(() => {
                loadUserStats();
            }, 2000);
            
            return () => {
                clearInterval(refreshInterval);
            };
        }
    });

    // Load garden data from local storage
    function loadGardenData() {
        try {
            if (isBrowser) {
                console.log("Loading garden data from localStorage");
                const savedGarden = localStorage.getItem(GARDEN_KEY);
                
                if (savedGarden) {
                    garden = JSON.parse(savedGarden);
                    console.log("Loaded garden data:", garden);
                }
            }
        } catch (error) {
            console.error('Error loading garden data:', error);
        }
    }

    // Load user stats from local storage
    function loadUserStats() {
        try {
            if (isBrowser) {
                console.log("Loading user stats from localStorage");
                const savedStats = localStorage.getItem(USER_STATS_KEY);
                
                if (savedStats) {
                    const stats = JSON.parse(savedStats);
                    userPoints = stats.points;
                    console.log("Loaded user points:", userPoints);
                }
            }
        } catch (error) {
            console.error('Error loading user stats:', error);
        }
    }

    // Save garden data to local storage
    function saveGardenData() {
        try {
            if (isBrowser) {
                console.log("Saving garden data to localStorage");
                localStorage.setItem(GARDEN_KEY, JSON.stringify(garden));
            }
        } catch (error) {
            console.error('Error saving garden data:', error);
        }
    }

    // Update user points after spending
    function updateUserPoints(pointsSpent: number) {
        try {
            if (isBrowser) {
                console.log(`Spending ${pointsSpent} points`);
                const savedStats = localStorage.getItem(USER_STATS_KEY);
                
                if (savedStats) {
                    const stats = JSON.parse(savedStats);
                    stats.points -= pointsSpent;
                    userPoints = stats.points;
                    console.log("Updated user points:", userPoints);
                    localStorage.setItem(USER_STATS_KEY, JSON.stringify(stats));
                }
            }
        } catch (error) {
            console.error('Error updating user points:', error);
        }
    }

    // Start planting mode
    function startPlanting(flowerId: string) {
        const flower = availableFlowers.find((f) => f.id === flowerId);
        
        if (flower && userPoints >= flower.cost) {
            selectedFlower = flowerId;
            isPlantingMode = true;
            console.log("Starting planting mode with flower:", flower.name);
        } else {
            alert("You don't have enough points to plant this flower!");
        }
    }

    // Plant a flower in a tile
    function plantFlower(tileId: number) {
        if (!selectedFlower || !isPlantingMode) return;
        
        const flower = availableFlowers.find((f) => f.id === selectedFlower);
        
        if (flower && userPoints >= flower.cost) {
            console.log(`Planting ${flower.name} in tile ${tileId}`);
            
            // Update the tile
            garden.tiles = garden.tiles.map((tile) =>
                tile.id === tileId
                    ? {
                          ...tile,
                          flowerId: selectedFlower,
                          plantedDate: new Date().toISOString(),
                      }
                    : tile,
            );
            
            // Spend points
            updateUserPoints(flower.cost);
            
            // Exit planting mode
            isPlantingMode = false;
            selectedFlower = null;
            
            // Save garden data
            saveGardenData();
            
            // Celebrate with confetti
            triggerConfetti();
        }
    }

    // Cancel planting mode
    function cancelPlanting() {
        isPlantingMode = false;
        selectedFlower = null;
    }

    // Trigger confetti celebration
    function triggerConfetti() {
        if (isBrowser) {
            confetti({
                particleCount: 80,
                spread: 100,
                origin: { y: 0.6 },
            });
        }
    }

    // Get the flower for a tile
    function getFlowerForTile(tileId: number): Flower | null {
        const tile = garden.tiles.find((t) => t.id === tileId);
        if (tile && tile.flowerId) {
            return availableFlowers.find((f) => f.id === tile.flowerId) || null;
        }
        return null;
    }

    // Check if user can afford a flower
    function canAffordFlower(flower: Flower): boolean {
        return userPoints >= flower.cost;
    }

    // Get filtered list of flowers the user has unlocked
    const unlockedFlowers = $derived(
        availableFlowers.filter((flower) =>
            garden.unlockedFlowers.includes(flower.id),
        ),
    );
</script>

<div class="flex flex-col p-4 bg-base-100 rounded-lg shadow-lg max-w-md w-full">
    <h2 class="text-2xl font-bold mb-4 text-base-content">Your Garden</h2>

    <!-- Points Display -->
    <div class="mb-4 flex items-center">
        <span class="text-lg font-bold">{userPoints}</span>
        <span class="ml-2 text-sm">points available</span>
    </div>

    <!-- Garden Grid -->
    <div class="grid grid-cols-4 gap-2 mb-6">
        {#each garden.tiles as tile (tile.id)}
            <button
                class={`w-16 h-16 rounded-lg flex items-center justify-center text-3xl
                ${isPlantingMode ? "border-2 border-dashed border-primary hover:bg-primary/10" : "bg-base-200"}`}
                onclick={() => isPlantingMode ? plantFlower(tile.id) : null}
                disabled={!isPlantingMode && !tile.flowerId}
            >
                {#if tile.flowerId}
                    <span>{getFlowerForTile(tile.id)?.image || "🌱"}</span>
                {:else}
                    {#if isPlantingMode}
                        <span class="text-primary/50 text-xs">Plant here</span>
                    {:else}
                        <span class="text-xs text-base-content/30">Empty</span>
                    {/if}
                {/if}
            </button>
        {/each}
    </div>

    {#if isPlantingMode}
        <div class="mb-4 p-3 bg-primary/10 rounded-lg">
            <p class="text-sm mb-2">
                Select a spot in your garden to plant your flower.
            </p>
            <button class="btn btn-sm btn-ghost" onclick={cancelPlanting}>
                Cancel
            </button>
        </div>
    {:else}
        <!-- Available Flowers to Plant -->
        <div>
            <h3 class="text-lg font-semibold mb-2">Plant New Flowers</h3>
            <div class="grid grid-cols-2 gap-2">
                {#each unlockedFlowers as flower (flower.id)}
                    <div class="bg-base-200 p-3 rounded-lg">
                        <div class="flex items-center mb-2">
                            <span class="text-2xl mr-2">{flower.image}</span>
                            <div>
                                <h4 class="font-medium">{flower.name}</h4>
                                <p class="text-xs text-base-content/70">
                                    {flower.rarity.charAt(0).toUpperCase() +
                                        flower.rarity.slice(1)}
                                </p>
                            </div>
                        </div>
                        <p class="text-xs mb-2">{flower.description}</p>
                        <button
                            class={`btn btn-xs ${canAffordFlower(flower) ? "btn-primary" : "btn-disabled"} w-full`}
                            disabled={!canAffordFlower(flower)}
                            onclick={() => startPlanting(flower.id)}
                        >
                            Plant ({flower.cost} pts)
                        </button>
                    </div>
                {/each}
            </div>
        </div>
    {/if}
</div>
