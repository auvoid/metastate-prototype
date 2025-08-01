<script lang="ts">
    import SplashScreen from "$lib/fragments/SplashScreen/SplashScreen.svelte";
    import { onMount, setContext } from "svelte";
    import "../app.css";
    import { onNavigate } from "$app/navigation";
    import { GlobalState } from "$lib/global/state";

    import { runtime } from "$lib/global/runtime.svelte";
    import { type Status, checkStatus } from "@tauri-apps/plugin-biometric";

    const { children } = $props();

    let globalState: GlobalState | undefined = $state(undefined);

    let showSplashScreen = $state(false);
    let previousRoute = null;
    let navigationStack: string[] = [];

    setContext("globalState", () => globalState);

    // replace with actual data loading logic
    async function loadData() {
        await new Promise((resolve) => setTimeout(resolve, 1500));
    }

    async function ensureMinimumDelay() {
        await new Promise((resolve) => setTimeout(resolve, 500));
    }

    onMount(async () => {
        let status: Status | undefined = undefined;
        try {
            status = await checkStatus();
        } catch (error) {
            status = {
                biometryType: 0,
                isAvailable: false,
            };
        }
        runtime.biometry = status.biometryType;
        try {
            globalState = await GlobalState.create();
        } catch (error) {
            console.error("Failed to initialize global state:", error);
            // Consider adding fallback behavior or user notification
        }

        showSplashScreen = true; // Can't set up the original state to true or animation won't start
        navigationStack.push(window.location.pathname);

        await Promise.all([loadData(), ensureMinimumDelay()]);

        showSplashScreen = false;
    });

    const safeAreaTop = $derived.by(
        () =>
            Number.parseFloat(
                getComputedStyle(document.documentElement).getPropertyValue(
                    "--safe-top",
                ),
            ) || 0,
    );

    $effect(() => console.log("top", safeAreaTop));

    onNavigate((navigation) => {
        if (!document.startViewTransition) return;

        const from = navigation.from?.url.pathname;
        const to = navigation.to?.url.pathname;

        if (!from || !to || from === to) return;

        let direction: "left" | "right" = "right";

        const fromIndex = navigationStack.lastIndexOf(from);
        const toIndex = navigationStack.lastIndexOf(to);

        if (toIndex !== -1 && toIndex < fromIndex) {
            // Backward navigation
            direction = "left";
            navigationStack = navigationStack.slice(0, toIndex + 1);
        } else {
            // Forward navigation (or new path)
            direction = "right";
            navigationStack.push(to);
        }

        document.documentElement.setAttribute("data-transition", direction);
        previousRoute = to;

        return new Promise((resolve) => {
            document.startViewTransition(async () => {
                resolve();
                await navigation.complete;
            });
        });
    });
</script>

{#if showSplashScreen}
    <SplashScreen />
{:else}
    <div
        class="fixed top-0 left-0 right-0 h-[env(safe-area-inset-top)] bg-primary z-50"
    ></div>
    <div class="bg-white h-screen overflow-scroll pt-10">
        {@render children?.()}
    </div>
{/if}

<style>
    :root {
        --safe-bottom: env(safe-area-inset-bottom);
        --safe-top: env(safe-area-inset-top);
    }

    :global(body),
    * {
        -webkit-overflow-scrolling: touch; /* keeps momentum scrolling on iOS */
        scrollbar-width: none; /* Firefox */
        -ms-overflow-style: none; /* IE 10+ */
    }

    /* Hide scrollbar for WebKit (Chrome, Safari) */
    :global(body::-webkit-scrollbar),
    *::-webkit-scrollbar {
        display: none;
    }
</style>
