<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    let nodes = [];
    let links = [];

    // Form inputs
    let term = '';
    let relationType = '';
    let relationTarget = '';
    let citation = '';
    let example = '';
    let definition = '';

    let svg, simulation, zoomContainer;
    let width, height;
    let colorScale;

    // Update graph when data changes
    $: if (svg) drawGraph();

    function addNode() {
        const lowerTerm = term.trim().toLowerCase();
        const lowerTarget = relationTarget.trim().toLowerCase();
        if (!lowerTerm) return;

        // Add or update the source node
        let sourceNode = nodes.find(n => n.id === lowerTerm);
        if (!sourceNode) {
            nodes.push({
                id: lowerTerm,
                citation,
                example,
                definition
            });
        } else {
            sourceNode.citation = citation;
            sourceNode.example = example;
            sourceNode.definition = definition;
        }

        // Add the relation and target node if provided
        if (relationType && lowerTarget) {
            if (!nodes.some(n => n.id === lowerTarget)) {
                nodes.push({
                    id: lowerTarget,
                    citation: '',
                    example: '',
                    definition: ''
                });
            }
            links.push({
                source: lowerTerm,
                target: lowerTarget,
                type: relationType
            });
        }

        // Reset form
        term = relationType = relationTarget = citation = example = definition = '';

        // Zoom to the new node after a short delay so the simulation updates
        setTimeout(() => {
            centerOnNode(lowerTerm);
        }, 500);
    }

    function deleteNode(nodeId) {
        nodes = nodes.filter(n => n.id !== nodeId);
        links = links.filter(l => l.source !== nodeId && l.target !== nodeId);
    }

    function deleteLink(index) {
        links.splice(index, 1);
    }

    function drawGraph() {
        width = window.innerWidth - 320;
        height = window.innerHeight;

        d3.select("#graph").selectAll("*").remove();

        svg = d3.select("#graph")
            .attr("width", width)
            .attr("height", height)
            .style("background", "#f9fafb");

        zoomContainer = svg.append("g");
        svg.call(d3.zoom().scaleExtent([0.5, 5]).on("zoom", ({ transform }) => {
            zoomContainer.attr("transform", transform);
        }));

        const types = [...new Set(links.map(l => l.type))];
        colorScale = d3.scaleOrdinal()
            .domain(types)
            .range(["#60a5fa", "#facc15", "#34d399", "#f87171", "#a78bfa"]);

        simulation = d3.forceSimulation(nodes)
            .force("link", d3.forceLink(links).id(d => d.id).distance(150))
            .force("charge", d3.forceManyBody().strength(-350))
            .force("center", d3.forceCenter(width / 2, height / 2));

        const link = zoomContainer.append("g")
            .attr("stroke-opacity", 0.5)
            .selectAll("line")
            .data(links)
            .join("line")
            .attr("stroke-width", 2)
            .attr("stroke", d => colorScale(d.type));

        const node = zoomContainer.append("g")
            .selectAll("circle")
            .data(nodes)
            .join("circle")
            .attr("r", 10)
            .attr("fill", "#374151")
            .attr("stroke", "#f3f4f6")
            .attr("stroke-width", 1.5);

        const label = zoomContainer.append("g")
            .selectAll("text")
            .data(nodes)
            .join("text")
            .text(d => d.id)
            .attr("font-size", 12)
            .attr("dx", 12)
            .attr("dy", "0.35em")
            .attr("fill", "#6b7280");

        simulation.on("tick", () => {
            link
                .attr("x1", d => d.source.x)
                .attr("y1", d => d.source.y)
                .attr("x2", d => d.target.x)
                .attr("y2", d => d.target.y);

            node
                .attr("cx", d => d.x)
                .attr("cy", d => d.y);

            label
                .attr("x", d => d.x)
                .attr("y", d => d.y);
        });
    }

    function centerOnNode(nodeId) {
        const node = nodes.find(n => n.id === nodeId);
        if (!node || node.x === undefined || node.y === undefined) return;

        const scale = 2;
        const translateX = width / 2 - node.x * scale;
        const translateY = height / 2 - node.y * scale;

        zoomContainer.transition().duration(750)
            .attr("transform", `translate(${translateX},${translateY}) scale(${scale})`);
    }

    function resetZoom() {
        zoomContainer.transition().duration(750).attr("transform", "translate(0,0) scale(1)");
    }

    function exportJSON() {
        const json = JSON.stringify({ nodes, links }, null, 2);
        const blob = new Blob([json], { type: "application/json" });
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = "concept-graph.json";
        a.click();
    }

    function exportCSV() {
        let csv = "Term,Citation,Definition,Example,Relation_Type,Relation_Target\n";

        nodes.forEach(node => {
            const nodeLinks = links.filter(link => link.source === node.id);
            if (nodeLinks.length) {
                nodeLinks.forEach(link => {
                    csv += `${node.id},"${node.citation}","${node.definition}","${node.example}",${link.type},${link.target}\n`;
                });
            } else {
                // No link — include node alone
                csv += `${node.id},"${node.citation}","${node.definition}","${node.example}","",""\n`;
            }
        });

        const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = "concept-graph.csv";
        a.click();
    }
</script>

<div class="flex h-screen overflow-hidden font-sans">

    <!-- Sidebar -->
    <div class="w-80 bg-white border-r border-gray-200 p-4 flex flex-col space-y-4 overflow-y-auto">
        <h2 class="text-xl font-semibold text-gray-800">Build Your Graph</h2>

        <div class="space-y-2">
            <input placeholder="Concept / Term" bind:value={term} class="w-full border border-gray-300 rounded px-2 py-1"/>
            <input placeholder="Relation Type (e.g., type of)" bind:value={relationType} class="w-full border border-gray-300 rounded px-2 py-1"/>
            <input placeholder="Target Term (optional)" bind:value={relationTarget} class="w-full border border-gray-300 rounded px-2 py-1"/>
            <input placeholder="Citation" bind:value={citation} class="w-full border border-gray-300 rounded px-2 py-1"/>
            <input placeholder="Example" bind:value={example} class="w-full border border-gray-300 rounded px-2 py-1"/>
            <input placeholder="Definition" bind:value={definition} class="w-full border border-gray-300 rounded px-2 py-1"/>
            <button on:click={addNode} class="w-full px-3 py-2 rounded bg-blue-500 text-white hover:bg-blue-600">
                Add Concept
            </button>
        </div>

        <!-- Zoom / Export -->
        <div class="flex flex-wrap gap-2 mt-4">
            <button on:click={resetZoom} class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300 text-sm">Reset Zoom</button>
            <button on:click={exportJSON} class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300 text-sm">Export JSON</button>
            <button on:click={exportCSV} class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300 text-sm">Export CSV</button>
        </div>

        <!-- Legend -->
        <div class="mt-6">
            <h4 class="font-semibold text-gray-800">Relation Types</h4>
            {#if colorScale}
                <ul class="space-y-1">
                    {#each colorScale.domain() as type}
                        <li class="flex items-center gap-2">
                            <div class="w-4 h-4 rounded" style="background-color:{colorScale(type)}"></div>
                            <span class="text-gray-700 text-sm">{type}</span>
                        </li>
                    {/each}
                </ul>
            {/if}
        </div>

        <!-- Node List -->
        <div class="mt-6">
            <h4 class="font-semibold text-gray-800">Nodes</h4>
            <ul class="space-y-1">
                {#each nodes as n}
                    <li class="flex justify-between items-center">
                        <span class="text-gray-700 text-sm">{n.id}</span>
                        <button on:click={() => deleteNode(n.id)} class="text-red-500 text-xs">Delete</button>
                    </li>
                {/each}
            </ul>
        </div>

        <!-- Link List -->
        <div class="mt-4">
            <h4 class="font-semibold text-gray-800">Links</h4>
            <ul class="space-y-1">
                {#each links as l, i}
                    <li class="flex justify-between items-center text-sm">
                        {l.source} — {l.type} — {l.target}
                        <button on:click={() => deleteLink(i)} class="text-red-500 text-xs">Delete</button>
                    </li>
                {/each}
            </ul>
        </div>
    </div>

    <!-- Graph -->
    <svg id="graph" class="flex-1"></svg>
</div>
