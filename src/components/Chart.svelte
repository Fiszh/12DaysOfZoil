<script lang="ts">
    import { onMount, onDestroy } from "svelte";
    import { Chart, registerables } from "chart.js";
    import type { ChartType } from "chart.js";

    export let type: ChartType = "line"; // line, bar, etc.
    export let labels: string[] = [];
    export let dataPoints: number[] = [];
    export let label: string = "";
    export let color: string = "#FF00AE";

    let canvas: HTMLCanvasElement;
    let chart: Chart;

    onMount(() => {
        Chart.register(...registerables);

        chart = new Chart(canvas, {
            type,
            data: {
                labels,
                datasets: [
                    {
                        label,
                        data: dataPoints,
                        borderColor: color,
                        backgroundColor:
                            type === "line" ? "transparent" : color,
                        borderWidth: 1,
                        tension: 0.4,
                    },
                ],
            },
            options: {
                plugins: { legend: { display: false } },
                scales: {
                    x: { grid: { color: "#333" }, ticks: { color: "#ccc" } },
                    y: { grid: { color: "#333" }, ticks: { color: "#ccc" } },
                },
            },
        });
    });

    onDestroy(() => {
        chart?.destroy();
    });
</script>

<canvas bind:this={canvas}></canvas>
