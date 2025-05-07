<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    let nodes = [];
    let links = [];
    let data = [];
    let searchTerm = '';
    let searchSuggestions = [];
    let selectedNode = null;

    const csvUrl = 'concepts.csv';

    function parseRelation(relationText) {
        const match = relationText.match(/\((.*?)\)\s*(.+)/);
        if (match) {
            return { type: match[1].trim().toLowerCase(), target: match[2].trim().toLowerCase() };
        }
        return null;
    }

    let svg, simulation, zoomContainer;
    let width, height;
    let colorScale;

    onMount(async () => {
        data = await d3.csv(csvUrl);

        const nodeSet = new Set();
        data.forEach(d => {
            if (d.Term) nodeSet.add(d.Term.toLowerCase());
        });

        const tempLinks = [];
        data.forEach(d => {
            const source = d.Term ? d.Term.toLowerCase() : null;
            if (source && d.Relations_lower) {
                const relations = d.Relations_lower.split(';');
                relations.forEach(r => {
                    const parsed = parseRelation(r.trim());
                    if (parsed && nodeSet.has(parsed.target)) {
                        tempLinks.push({
                            source,
                            target: parsed.target,
                            type: parsed.type
                        });
                    }
                });
            }
        });

        nodes = Array.from(nodeSet).map(name => {
            const original = data.find(d => d.Term && d.Term.toLowerCase() === name);
            return {
                id: name,
                citation: original?.Citation || 'No citation available.',
                example: original?.Examples || 'No example available.',
                definition: original?.Definition || 'No example available.'
            };
        });

        links = tempLinks;

        drawGraph();
    });

    $: selected = nodes.find(n => n.id === selectedNode);

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

        // Extract all valid, lowercased, trimmed relation types (avoid duplicates and undefined)
const relationTypes = [...new Set(
    links
        .map(l => l.type ? l.type.trim().toLowerCase() : null)
        .filter(t => t) // remove empty/undefined
)].sort(); // Optional: keep the legend consistent order

colorScale = d3.scaleOrdinal()
    .domain(relationTypes)
    .range(["#60a5fa", "#facc15", "#34d399", "#f87171", "#a78bfa", "#309898", "#F3F3E0", "#183B4E", "#FFC1DA"]);

     

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
            .attr("fill", d => selectedNode === d.id ? '#ef4444' : '#374151')
            .attr("stroke", "#f3f4f6")
            .attr("stroke-width", 1.5)
            .call(drag(simulation))
            .on("click", (event, d) => {
                selectedNode = d.id;
                centerOnNode(d.id);
                highlightConnections(d.id);
            })
            .on("mouseover", (event, d) => {
                d3.select(event.currentTarget).transition().attr("r", 14);
            })
            .on("mouseout", (event, d) => {
                d3.select(event.currentTarget).transition().attr("r", 10);
            });

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

    function drag(simulation) {
        return d3.drag()
            .on("start", (event, d) => {
                if (!event.active) simulation.alphaTarget(0.3).restart();
                d.fx = d.x;
                d.fy = d.y;
            })
            .on("drag", (event, d) => {
                d.fx = event.x;
                d.fy = event.y;
            })
            .on("end", (event, d) => {
                if (!event.active) simulation.alphaTarget(0);
                d.fx = null;
                d.fy = null;
            });
    }

    function onSearchInput() {
        const query = searchTerm.toLowerCase();
        searchSuggestions = nodes
            .filter(n => n.id.includes(query))
            .slice(0, 5)
            .map(n => n.id);
    }

    function selectSuggestion(suggestion) {
        searchTerm = suggestion;
        selectedNode = suggestion;
        searchSuggestions = [];
        centerOnNode(suggestion);
        highlightConnections(suggestion);
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

    function highlightConnections(nodeId) {
        zoomContainer.selectAll("line")
            .attr("stroke-opacity", d =>
                d.source.id === nodeId || d.target.id === nodeId ? 1 : 0.1);

        zoomContainer.selectAll("circle")
            .attr("fill", d =>
                d.id === nodeId ? '#ef4444' :
                    links.some(l => (l.source.id === nodeId && l.target.id === d.id) ||
                        (l.target.id === nodeId && l.source.id === d.id)) ? '#6b7280' : '#d1d5db');
    }

    function resetView() {
        zoomContainer.transition().duration(750)
            .attr("transform", `translate(0,0) scale(1)`);

        zoomContainer.selectAll("line").attr("stroke-opacity", 0.5);
        zoomContainer.selectAll("circle").attr("fill", d =>
            selectedNode === d.id ? '#ef4444' : '#374151');
    }

    function exportSVG() {
        const serializer = new XMLSerializer();
        const source = serializer.serializeToString(document.getElementById("graph"));
        const blob = new Blob([source], { type: "image/svg+xml;charset=utf-8" });
        const url = URL.createObjectURL(blob);
        const link = document.createElement("a");
        link.href = url;
        link.download = "concept-graph.svg";
        link.click();
    }

    function zoomIn() {
        zoomContainer.transition().duration(500)
            .attr("transform", "scale(1.5)");
    }

    function zoomOut() {
        zoomContainer.transition().duration(500)
            .attr("transform", "scale(0.75)");
    }
</script>

<div class="flex h-screen overflow-hidden font-sans">

    <!-- Sidebar -->
    <div class="w-80 bg-white border-r border-gray-200 p-4 flex flex-col space-y-4">
        <h2 class="text-xl font-semibold text-gray-800">Concept Graph</h2>

        <!-- Search -->
        <div>
            <input type="text" placeholder="Search term..." bind:value={searchTerm} on:input={onSearchInput}
                class="w-full border border-gray-300 rounded px-3 py-2" />
            {#if searchSuggestions.length > 0}
                <ul class="border border-gray-300 rounded bg-white mt-1 max-h-40 overflow-auto">
                    {#each searchSuggestions as suggestion}
                        <li on:click={() => selectSuggestion(suggestion)}
                            class="px-3 py-2 cursor-pointer hover:bg-gray-100">{suggestion}</li>
                    {/each}
                </ul>
            {/if}
        </div>

        <div class="mt-6">
            <h4 class="font-semibold text-gray-800">Relation Types</h4>
            {#if colorScale}
                <ul class="space-y-1">
                    {#each colorScale.domain() as type, i}
                        <li class="flex items-center gap-2">
                            <div class="w-4 h-4 rounded" style="background-color:{colorScale(type)}"></div>
                            <span class="text-gray-700 text-sm">{type}</span>
                        </li>
                    {/each}
                </ul>
            {/if}
        </div>

        <!-- Controls -->
        <div class="flex gap-2">
            <button on:click={resetView} class="px-3 py-2 rounded bg-gray-200 hover:bg-gray-300 text-sm">Reset View</button>
            <button on:click={exportSVG} class="px-3 py-2 rounded bg-gray-200 hover:bg-gray-300 text-sm">Export SVG</button>
        </div>
        <div class="flex gap-2">
            <button on:click={zoomIn} class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300 text-sm">Zoom In</button>
            <button on:click={zoomOut} class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300 text-sm">Zoom Out</button>
        </div>

        <!-- Selected Node Info -->
        <div class="flex-1 overflow-y-auto">
            {#if selectedNode}
                <div class="mt-4 space-y-2">
                    <h3 class="text-lg font-semibold text-gray-900 capitalize">{selectedNode}</h3>
                    <p class="text-gray-700"><strong>Citation:</strong> {selected ? selected.citation : 'Not found'}</p>
                    <p class="text-gray-700"><strong>Example:</strong> {selected ? selected.example : 'Not found'}</p>
                    <p class="text-gray-700"><strong>Definition:</strong> {selected ? selected.definition : 'Not found'}</p>
                </div>
            {/if}

        </div>
    </div>

    <!-- Graph -->
    <svg id="graph" class="flex-1"></svg>
</div>
