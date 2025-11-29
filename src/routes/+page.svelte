<script lang="ts">
    import { onMount } from "svelte";
    import { writable } from "svelte/store";
    import { afterNavigate } from "$app/navigation";

    import ChartComponent from "$components/Chart.svelte";

    let currentParam: string = "";

    afterNavigate((navigation) => {
        const url = navigation.to?.url;
        if (url) {
            currentParam = url.searchParams.get("time") || "";
        }
    });

    const navLinks = {
        today: {
            navLink: undefined as HTMLElement | undefined,
            param: "today",
        },
        week: {
            navLink: undefined as HTMLElement | undefined,
            param: "week",
        },
        alltime: {
            navLink: undefined as HTMLElement | undefined,
            param: "alltime",
        },
    };

    $: if (currentParam) {
        loadData();

        for (const navLink of Object.values(navLinks)) {
            const hasHash = navLink["param"] == currentParam;

            if (currentParam) {
                if (navLink.navLink instanceof HTMLElement) {
                    if (hasHash) {
                        navLink.navLink.classList.add("active");
                    } else {
                        navLink.navLink.classList.remove("active");
                    }
                }
            } else {
                navLinks.today.navLink?.classList.add("active");
            }
        }
    }

    const refreshTime = writable(20);
    let topData = writable({});
    let mainData = writable<any[]>([]);
    let otherData = writable<{ new: number; total: number }>({
        new: 0,
        total: 0,
    });
    let graphData = writable<Record<string, any>>({ labels: [], data: [] });

    async function loadData() {
        const aviableTimeFrames = ["today", "week", "alltime"];

        let timeParam: string = currentParam;
        if (!timeParam || !aviableTimeFrames.includes(timeParam)) {
            timeParam = "today";
        }

        const response = await fetch(`http://localhost:9530/zoil/${timeParam}`);

        if (!response.ok) {
            throw new Error(`Failed to get ${timeParam} data!`);
        }

        const data = await response.json();

        if (data["data"] || data["otherData"]) {
            const rows: any = {};

            for (const item of Object.values(data["data"]) as Record<
                string,
                any
            >[]) {
                if (!item || typeof item !== "object" || !item.row) continue;

                if (!rows[item.row]) {
                    rows[item.row] = { row: item.row, data: [] };
                }

                rows[item.row].data.push(item);
            }

            const mapped = Object.values(rows) as any[];

            const top_data = Object.fromEntries(
                Object.entries(data.data).filter(
                    ([key, value]) => typeof value === "number",
                ),
            );

            let other_data = {
                new: 0,
                total: 0,
            };

            const date_keys = Object.keys(data["otherData"]);
            for (const date_key of date_keys) {
                const time_keys = Object.keys(data["otherData"][date_key]);

                for (const time_key of time_keys) {
                    const sub_data = data["otherData"][date_key][time_key];

                    other_data["new"] += sub_data["channel_subs_gained"];
                    if (sub_data["channel_subs"]) {
                        other_data["total"] = sub_data["channel_subs"];
                    }
                }
            }

            graphData.set({
                labels: date_keys
                    .flatMap((key) => Object.keys(data["otherData"][key]))
                    .slice(-10),
                data: [
                    {
                        name: "Viewers",
                        values: date_keys
                            .flatMap((key) =>
                                Object.values(data["otherData"][key]).flatMap(
                                    (key: any) => key["channel_viewers"],
                                ),
                            )
                            .slice(-10),
                    },
                    {
                        name: "Subs Gained",
                        values: date_keys
                            .flatMap((key) =>
                                Object.values(data["otherData"][key]).flatMap(
                                    (key: any) => key["channel_subs_gained"],
                                ),
                            )
                            .slice(-10),
                    },
                    {
                        name: "Channel Active Subscriptions",
                        type: "bar",
                        values: date_keys
                            .flatMap((key) => {
                                const obj = data.otherData[key] as Record<
                                    string,
                                    { channel_subs: number[] }
                                >;
                                return obj
                                    ? obj[Object.keys(obj)[0]].channel_subs
                                    : [];
                            })
                            .slice(-10),

                        labels: date_keys.flatMap((key: any) => key).slice(-10),
                    },
                ],
            });

            topData.set(top_data);
            mainData.set(mapped);
            otherData.set(other_data);
        }
    }

    onMount(() => {
        loadData();

        const interval = setInterval(() => {
            refreshTime.update((n: number) => {
                if (n <= 0) {
                    // REFRESH FUNCTION HERE
                    loadData();

                    return 20;
                }
                return n - 1;
            });
        }, 1000);

        return () => clearInterval(interval);
    });
</script>

{#snippet numberStat(name: string, value: number)}
    <div class="stat number">
        <p class="name">{name}</p>
        <p class="value">{value.toLocaleString()}</p>
    </div>
{/snippet}

{#snippet graphStat(name: string, values: any, labels: any, type: any)}
    <div class="stat graph">
        <p class="name">{name}</p>

        <ChartComponent
            {type}
            {labels}
            dataPoints={values}
            label={`${type} Stats`}
        />
    </div>
{/snippet}

{#snippet listStat(name: string, data: any[])}
    <div class="stat list">
        <p class="name">{name}</p>
        <ul>
            {#each data as stat, i}
                <li>
                    <p class="item_placement">#{i + 1}</p>
                    {#if stat.url}
                        <img src={stat.url} alt="stat_icon" />
                    {/if}
                    <div class="item_info">
                        <p class="item_name">{stat.name}</p>
                        <div class="count">
                            <p>Count:</p>
                            <p class="item_count">{stat.count}</p>
                        </div>
                    </div>
                </li>
            {/each}
        </ul>
    </div>
{/snippet}

<topbar>
    <img src="https://cdn.unii.dev/zoil.png" alt="icon" />
    <h1>12 Days Of Zoil Tracker</h1>

    <div id="time_buttons">
        <p>Refresh: {$refreshTime}s</p>
        <a
            id="today"
            href="?time=today"
            class="active"
            bind:this={navLinks.today.navLink}>Today</a
        >
        <a id="week" href="?time=week" bind:this={navLinks.week.navLink}>Week</a
        >
        <a
            id="alltime"
            href="?time=alltime"
            bind:this={navLinks.alltime.navLink}>All Time</a
        >
    </div>
</topbar>

<main>
    <section class="row">
        {@render numberStat("New Subscriptions", $otherData["new"])}
        {@render numberStat("Total Subscriptions", $otherData["total"])}
    </section>
    <section class="row">
        {#each Object.entries($topData) as [key, value]}
            {#if typeof value == "number"}
                {@render numberStat(
                    key
                        .replace(/_/g, " ")
                        .replace(/\b\w/g, (c) => c.toUpperCase()),
                    value,
                )}
            {/if}
        {/each}
    </section>
    <section class="row">
        {#each $graphData?.["data"] as stat}
            {@render graphStat(
                stat.name,
                stat.values,
                stat.values !== undefined
                    ? stat.values
                    : $graphData?.["labels"],
                typeof stat.type === "string" ? stat.type : "line",
            )}
        {/each}
    </section>
    {#each $mainData as row}
        <section class="row">
            {#each row.data as stat}
                {@render listStat(stat.stat_name, stat.data)}
            {/each}
        </section>
    {/each}
</main>

<style lang="scss">
    :root {
        color: white;
        height: 100dvh;
        width: 100dvw;
        font-family: "Inter", "Amiri", sans-serif, "Noto Color Emoji";
        user-select: none;
        overflow: hidden;
        display: flex;
        flex-direction: column;

        p {
            margin: 0;
            display: block;
        }

        & {
            scrollbar-width: thin;
            scrollbar-color: #555 #1e1e1e;
        }
    }

    // * > * {
    //     outline: rgba(255, 255, 255, 0.25) 1px solid;
    // }

    topbar {
        flex: 0 1;
        padding: 0.5dvw 1dvh;
        font-size: 35px;
        background-color: rgb(0, 0, 0);
        border-bottom: 1px solid #333;
        align-items: center;
        font-weight: bold;
        gap: 2dvw;
        pointer-events: none;
        font-size: 1rem;
        display: flex;

        img {
            aspect-ratio: 1/1;
            height: 4rem;
        }

        #time_buttons {
            position: absolute;
            right: 1rem;
            display: flex;
            gap: 0.5rem;

            p {
                width: 6.5rem;
                text-align: center;
            }

            & > * {
                all: unset;
                border: 1px solid #333;
                padding: 0.3rem 1rem;
                pointer-events: all;
                border-radius: 1rem;
                color: #ffffffba;
                transition: all 0.2s ease;
            }

            & > .active {
                border: 1px solid #5c5c5c;
                background-color: #ffffff1a;
                color: #fff;
            }

            & > :hover {
                border: 1px solid #5c5c5c;
                color: #ffffffd4;
            }

            a {
                cursor: pointer;
            }

            a:active {
                border: 1px solid #fff;
                background-color: #ffffff1a;
                color: #fff;
            }
        }
    }

    main {
        height: 100%;
        width: 100%;
        background-color: rgb(0, 0, 0);
        padding: 1rem 15rem;
        box-sizing: border-box;
        overflow-y: auto;
        display: flex;
        flex-direction: column;
        gap: 1rem;

        .row {
            display: inline-flex;
            box-sizing: border-box;
            justify-content: center;
            width: 100%;
            gap: 1rem;

            .stat {
                background-color: #ffffff0a;
                padding: 1rem 1rem;
                box-sizing: border-box;
                display: flex;
                flex-direction: column;
                gap: 1rem;
                border: #ffffff26 1px solid;
                border-radius: 1rem;
                width: 100%;

                &.list {
                    .name {
                        text-align: center;
                    }

                    .item_name {
                        font-size: 0.9rem;
                    }
                }

                ul {
                    padding: 0;
                    display: flex;
                    flex-direction: column;
                    gap: 1rem;
                    height: 30rem;
                    overflow: auto;
                    padding: 0rem 0.5rem;
                    box-sizing: border-box;
                    width: 100%;

                    li {
                        all: unset;
                        display: flex;
                        gap: 1rem;
                        background-color: #ffffff0a;
                        border-radius: 0.5rem;
                        font-size: 0.7rem;
                        align-items: center;
                        padding: 0.7rem 1rem;
                        border: #ffffff26 1px solid;

                        & > * {
                            font-weight: bold;
                        }

                        .item_placement {
                            border: #ffffff7d 1px solid;
                            padding: 0rem;
                            margin: 0;
                            border-radius: 0.3rem;
                            min-width: 1.5rem;
                            height: 1.5rem;
                            font-size: 0.7rem;
                            display: flex;
                            justify-content: center;
                            align-items: center;
                        }

                        img {
                            width: 2rem;
                            height: 2rem;
                            object-fit: contain;
                            box-sizing: border-box;
                            border-radius: 0.3rem;
                            padding: 0.1rem;
                            transition: all 0.2s ease;
                            cursor: pointer;

                            &:hover {
                                padding: 0rem;
                            }
                        }

                        .count {
                            display: flex;
                            gap: 0.15rem;

                            .item_count {
                                color: #ff00ae;
                            }
                        }
                    }
                }
            }
        }
    }
</style>
