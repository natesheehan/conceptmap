<script>
	import { onMount, tick, onDestroy } from 'svelte';
	import { fly } from 'svelte/transition';
	import * as d3 from 'd3';
    import Menu from '$lib/components/Menu.svelte';

	let nodes = [], links = [], data = [];
	let searchTerm = '', searchSuggestions = [], selectedNode = null;
	const csvUrl = 'data.csv';

	let svg, simulation, zoomContainer;
	let width, height, colorScale;

	const exportFormats = ['json', 'csv', 'txt'];

	function parseRelation(text) {
		const match = text.match(/\((.*?)\)\s*(.+)/);
		return match ? { type: match[1].trim().toLowerCase(), target: match[2].trim().toLowerCase() } : null;
	}

function handleClickOutside(event) {
	setTimeout(async () => {
		const popup = document.getElementById('node-popup');
		if (popup && !popup.contains(event.target)) {
			selectedNode = null;
			searchSuggestions = [];
			await tick();
			drawGraph(); // <- restore all connections
		}
	}, 0);
}



onMount(() => {
	document.addEventListener('mousedown', handleClickOutside);
});
onDestroy(() => {
	document.removeEventListener('mousedown', handleClickOutside);
});


	$: selected = nodes.find(n => n.id === selectedNode);

onMount(async () => {
	data = await d3.csv(csvUrl);
	const nodeSet = new Set(data.map(d => d.Term?.toLowerCase()).filter(Boolean));

	links = [];
	data.forEach(d => {
		const source = d.Term?.toLowerCase();
		d.Relations?.split(';').forEach(r => {
			const parsed = parseRelation(r.trim());
			if (parsed && nodeSet.has(parsed.target)) {
				links.push({ source, target: parsed.target, type: parsed.type });
			}
		});
	});

	nodes = Array.from(nodeSet).map(id => {
		const original = data.find(d => d.Term?.toLowerCase() === id);
		return {
			id,
			citation: original?.Citation || 'No citation available.',
			example: original?.Examples || 'No example available.',
			definition: original?.Definition || 'No definition available.'
		};
	});

	const relationTypes = [...new Set(links.map(l => l.type).filter(Boolean))].sort();

	colorScale = d3.scaleOrdinal()
		.domain(relationTypes)
		.range(["#60a5fa", "#facc15", "#34d399", "#f87171", "#a78bfa", "#309898", "#F3F3E0", "#183B4E", "#FFC1DA"]);

	// ✅ Must reset this to trigger reactive update of filteredLinks
	selectedRelationTypes = new Set(relationTypes);

	// ✅ Ensure filteredLinks has correct initial value
	await tick(); // import { tick } from 'svelte'

	drawGraph();

	window.addEventListener('keydown', e => {
		if (e.key === 'Escape') selectedNode = null;
	});
});


	function drawGraph() {
		width = window.innerWidth;
		height = window.innerHeight;

		d3.select("#graph").selectAll("*").remove();


svg = d3.select("#graph")
	.attr("width", width)
	.attr("height", height)
	.style("background", "white")
.on("dblclick", async () => {
	selectedNode = null;
	searchSuggestions = [];
	await tick();
	drawGraph();
});



		zoomContainer = svg.append("g");
		svg.call(d3.zoom().scaleExtent([0.5, 5]).on("zoom", ({ transform }) => {
			zoomContainer.attr("transform", transform);
		}));

		const relationTypes = [...new Set(links.map(l => l.type).filter(Boolean))].sort();
		colorScale = d3.scaleOrdinal()
			.domain(relationTypes)
			.range(["#60a5fa", "#facc15", "#34d399", "#f87171", "#a78bfa", "#309898", "#F3F3E0", "#683B4E", "#FFC1DA"]);

		simulation = d3.forceSimulation(nodes)
			.force("link", d3.forceLink(links).id(d => d.id).distance(150))
			.force("charge", d3.forceManyBody().strength(-300))
			.force("center", d3.forceCenter(width / 2, height / 2));

const visibleLinks = links.filter(link =>
	selectedRelationTypes.size === 0 || selectedRelationTypes.has(link.type)
);

const link = zoomContainer.append("g")
	.selectAll("line")
	.data(visibleLinks)
	.join("line")
	.attr("stroke-width", 2)
	.attr("stroke", d => colorScale(d.type))
	.attr("stroke-opacity", 0.4);




		const node = zoomContainer.append("g")
			.selectAll("circle")
			.data(nodes)
			.join("circle")
			.attr("r", 10)
			.attr("fill", d => selectedNode === d.id ? 'blue' : 'green')
			.call(drag(simulation))
			.on("click", (_, d) => {
				selectedNode = d.id;
				centerOnNode(d.id);
				highlightConnections(d.id);
			})
			.on("mouseover", e => d3.select(e.currentTarget).transition().attr("r", 14))
			.on("mouseout", e => d3.select(e.currentTarget).transition().attr("r", 10));

		const label = zoomContainer.append("g")
			.selectAll("text")
			.data(nodes)
			.join("text")
			.text(d => d.id)
			.attr("font-size", 14)
			.attr("dx", 12)
			.attr("dy", "0.35em")
			.attr("fill", "black");

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

	function drag(sim) {
		return d3.drag()
			.on("start", (event, d) => {
				if (!event.active) sim.alphaTarget(0.3).restart();
				d.fx = d.x; d.fy = d.y;
			})
			.on("drag", (event, d) => { d.fx = event.x; d.fy = event.y; })
			.on("end", (event, d) => {
				if (!event.active) sim.alphaTarget(0);
				d.fx = null; d.fy = null;
			});
	}

	function centerOnNode(id) {
		const node = nodes.find(n => n.id === id);
		if (!node || node.x == null || node.y == null) return;
		const scale = 2;
		zoomContainer.transition().duration(750)
			.attr("transform", `translate(${width / 2 - node.x * scale},${height / 2 - node.y * scale}) scale(${scale})`);
	}

	function highlightConnections(id) {
		zoomContainer.selectAll("line")
			.attr("stroke-opacity", d => d.source.id === id || d.target.id === id ? 1 : 0.1);
		zoomContainer.selectAll("circle")
			.attr("fill", d =>
				d.id === id ? '#e11d48' :
					links.some(l => l.source.id === id && l.target.id === d.id || l.target.id === id && l.source.id === d.id)
						? '#7dd3fc' : '#94a3b8');
	}

	function resetView() {
		zoomContainer.transition().duration(750).attr("transform", `translate(0,0) scale(1)`);
		zoomContainer.selectAll("line").attr("stroke-opacity", 0.4);
		zoomContainer.selectAll("circle").attr("fill", d =>
			selectedNode === d.id ? '#e11d48' : '#f8fafc');
	}

	function zoomIn() {
		zoomContainer.transition().duration(500).attr("transform", "scale(1.5)");
	}
	function zoomOut() {
		zoomContainer.transition().duration(500).attr("transform", "scale(0.75)");
	}

	function onSearchInput() {
		const q = searchTerm.toLowerCase();
		searchSuggestions = nodes.filter(n => n.id.includes(q)).slice(0, 5).map(n => n.id);
	}
	function selectSuggestion(suggestion) {
		searchTerm = suggestion;
		selectedNode = suggestion;
		searchSuggestions = [];
		centerOnNode(suggestion);
		highlightConnections(suggestion);
	}

	function download(type) {
		let content = '';
		if (type === 'json') {
			content = JSON.stringify({ nodes, links }, null, 2);
		} else if (type === 'csv') {
			content = 'id,citation,example,definition\n' +
				nodes.map(n =>
					`"${n.id}","${n.citation.replace(/"/g, '""')}","${n.example.replace(/"/g, '""')}","${n.definition.replace(/"/g, '""')}"`
				).join('\n');
		} else if (type === 'txt') {
			content = nodes.map(n =>
				`Term: ${n.id}\nCitation: ${n.citation}\nExample: ${n.example}\nDefinition: ${n.definition}\n`
			).join('\n---\n');
		}

		const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `graph-data.${type}`;
		a.click();
	}

 function renderFormattedBlocks(text) {
	if (!text) return [];

	// Split by '//' to detect multiple example/definition blocks
	const entries = text.split('//').map(str => str.trim());

	return entries.map(entry => {
		const parts = entry.split(/(\(\[[^\]]+\]\))/g); // split by ([...])
		return parts.map(part => {
			const match = part.match(/\(\[(.+?)\]\)/); // match ([...])
			if (match) {
				const citationText = match[1].trim();
				const isURL = citationText.startsWith('http') || citationText.includes('doi.org');
				const url = isURL
					? citationText.replace(/\s+/g, '') // clean up line breaks
					: `https://scholar.google.com/scholar?q=${encodeURIComponent(citationText)}`;

				return {
					type: 'citation',
					text: citationText,
					url
				};
			}
			return { type: 'text', text: part };
		});
	});
}

function splitCitationsSmart(text) {
	if (!text) return [];

	const citations = [];
	let current = '';
	let insideUrl = false;

	const words = text.split(/\s+/);

	for (const word of words) {
		if (word.startsWith('http')) {
			insideUrl = true;
			current += word;
		} else if (insideUrl && (word.includes('.org') || word.includes('.com'))) {
			current += word;
		} else if (word === '//') {
			if (current) citations.push(current.trim());
			current = '';
			insideUrl = false;
		} else {
			current += (current ? ' ' : '') + word;
		}
	}
	if (current) citations.push(current.trim());

	return citations.map(raw => {
		const match = raw.match(/^(.*?\.\s+\d{4}\.?)\s+(“[^”]+”|"[^"]+").*?(http.*)$/i);
		const hasLink = raw.includes('http');

		if (match) {
			return {
				author: match[1]?.trim(),
				title: match[2]?.replace(/["“”]/g, '').trim(),
				url: match[3]?.trim(),
				type: 'doi'
			};
		} else {
			return {
				title: raw,
				url: hasLink ? raw.match(/http\S+/)?.[0] : null,
				type: hasLink ? 'doi' : 'scholar'
			};
		}
	});
}

function formatCitation(citation) {
	try {
		if (citation.includes('doi.org')) {
			const parts = citation.split('doi.org/');
			if (parts[1]) {
				return 'doi.org/' + parts[1].slice(0, 35) + (parts[1].length > 35 ? '...' : '');
			}
		}
		if (citation.startsWith('http')) {
			const url = new URL(citation);
			return url.hostname + url.pathname.slice(0, 20) + '...';
		}
	} catch (_) {}
	return citation;
}
let selectedRelationTypes = new Set();



$: if (svg && filteredLinks) {
	drawGraph();
}

</script>

<style>
	:global(body) {
		margin: 0;
		font-family: 'Inter', sans-serif;
	
		color: #f8fafc;
	}
</style>

<Menu/>

<svg id="graph" class="absolute top-0 left-0 w-full h-full z-0"></svg>

<!-- Floating Controls -->
<div class="absolute top-4 left-4 z-10 space-y-2 w-72">
	<input
		type="text"
		bind:value={searchTerm}
		on:input={onSearchInput}
		placeholder="Search concept..."
		class="w-full px-4 py-2 rounded shadow-md bg-white text-black"
	/>
	{#if searchSuggestions.length}
		<ul class="bg-white rounded shadow text-black">
			{#each searchSuggestions as s}
				<li class="px-4 py-1 hover:bg-gray-200 cursor-pointer" on:click={() => selectSuggestion(s)}>{s}</li>
			{/each}
		</ul>
	{/if}

	<!-- Legend -->
	<!-- UI Dropdown for selecting relation types -->
{#if colorScale}
	<div class="p-3 bg-white rounded shadow text-black mt-15">
		<h4 class="font-bold mb-2">Filter Relations</h4>

	<div class="flex gap-2 mb-2">
	<button
		class="text-xs px-2 py-1 bg-gray-200 rounded hover:bg-gray-300"
		on:click={() => {
			selectedRelationTypes = new Set(colorScale.domain());
			drawGraph(); // ✅ redraw
		}}
	>
		Select All
	</button>
	<button
		class="text-xs px-2 py-1 bg-gray-200 rounded hover:bg-gray-300"
		on:click={() => {
			selectedRelationTypes = new Set();
			drawGraph(); // ✅ redraw
		}}
	>
		Deselect All
	</button>
</div>



		<!-- Checkboxes here -->
		{#each colorScale.domain() as type}
			<label class="flex items-center space-x-2 text-sm mb-1">
				<input type="checkbox"
					checked={selectedRelationTypes.has(type)}
					on:change={() => {
						if (selectedRelationTypes.has(type)) {
							selectedRelationTypes.delete(type);
						} else {
							selectedRelationTypes.add(type);
						}
						selectedRelationTypes = new Set(selectedRelationTypes);
						drawGraph();
					}} />
				<span class="flex items-center space-x-2">
					<span class="w-3 h-3 rounded-full block" style="background-color: {colorScale(type)};"></span>
					<span>{type}</span>
				</span>
			</label>
		{/each}
	</div>
{/if}


	<!-- Map Controls -->
	<div class="flex gap-2">
		<button on:click={resetView} class="bg-white text-black px-3 py-1 rounded shadow hover:bg-gray-100">Reset</button>
		<button on:click={zoomIn} class="bg-white text-black px-3 py-1 rounded shadow hover:bg-gray-100">+</button>
		<button on:click={zoomOut} class="bg-white text-black px-3 py-1 rounded shadow hover:bg-gray-100">−</button>
	</div>

	<!-- Export Controls -->
<!-- Export Dropdown Button -->
<div class="relative group inline-block">
	<!-- Main Button -->
	<button class="bg-white text-black px-4 py-2 rounded shadow hover:bg-gray-100">
		Download ▼
	</button>

	<!-- Dropdown -->
	<div class="absolute left-0 mt-1 hidden group-hover:block bg-white text-black shadow-lg rounded z-20 w-32">
		{#each exportFormats as fmt}
			<div
				class="px-4 py-2 text-sm hover:bg-gray-200 cursor-pointer capitalize"
				on:click={() => download(fmt)}
			>
				{fmt}
			</div>
		{/each}
	</div>
</div>

</div>

{#if selectedNode}
<div
	id="node-popup"
	in:fly={{ x: 300 }}
	out:fly={{ x: 300 }}
	class="absolute top-20 right-0 w-full max-w-md h-90% bg-white text-gray-800 p-6 shadow-xl overflow-y-auto z-20 border-l border-gray-200"
>

		<!-- Header -->
		<div class="flex justify-between items-start mb-6">
			<div>
				<h2 class="text-2xl font-bold capitalize mb-1">{selectedNode}</h2>
				<p class="text-sm text-gray-500">Node details</p>
			</div>
<button
	on:click={async () => {
		selectedNode = null;
		await tick();
		drawGraph();
	}}
	class="text-gray-400 hover:text-gray-600 text-xl font-bold leading-none"
	aria-label="Close"
>
	✕
</button>

		</div>

		<!-- Citations -->
<div class="mb-8">
	<h3 class="text-sm font-semibold text-gray-600 uppercase tracking-wider mb-3">Citations</h3>
	{#if selected?.citation}
		<ul class="space-y-4">
			{#each splitCitationsSmart(selected.citation) as c}
				<li class="flex items-start gap-3">
					<!-- Icon -->
					<div class="text-gray-500 mt-1">
						<span title="Citation">📖</span>
					</div>

					<!-- Citation Info -->
					<div>
						<p class="text-sm text-gray-800 font-medium">{c.title}</p>
						{#if c.author}
							<p class="text-sm text-gray-500">{c.author}</p>
						{/if}
						{#if c.url}
							<a
								href={c.url}
								target="_blank"
								rel="noopener noreferrer"
								class="text-sm text-blue-600 hover:underline break-all"
							>
								{formatCitation(c.url)}
							</a>
						{/if}
					</div>
				</li>
			{/each}
		</ul>
       <button class="mt-5 float-right bg-green-400 hover:bg-blue-700 text-white font-semibold py-2 px-4 rounded-lg shadow-sm transition-colors duration-200">
	Add Citation
</button>
	{:else}
		<p class="text-gray-500 italic">No citations available.</p>
	{/if}
</div>


	<!-- Example -->
<div class="mb-8">
	<h3 class="text-sm mt-15 font-semibold text-gray-600 uppercase tracking-wider mb-2">Example</h3>
	<div class="rounded border border-gray-200 bg-gray-50 px-4 py-3 space-y-4">
		{#each renderFormattedBlocks(selected?.example) as block}
			<p class="text-gray-700 text-sm leading-relaxed">
				{#each block as part}
					{#if part.type === 'citation' && part.url}
						<a
							href={part.url}
							target="_blank"
							rel="noopener noreferrer"
							class="text-blue-600 hover:underline"
						>
							([{part.text}])
						</a>
					{:else}
						{@html part.text.replace(/\n/g, '<br>')}
					{/if}
				{/each}
			</p>
		{/each}
         
	</div>
        <button class="mt-5 float-right bg-green-400 hover:bg-blue-700 text-white font-semibold py-2 px-4 rounded-lg shadow-sm transition-colors duration-200">
	Add Example
</button>
</div>


	<!-- Definition -->
<div>
	<h3 class="text-sm mt-15 font-semibold text-gray-600 uppercase tracking-wider mb-2">Definition</h3>
	<div class="space-y-6 text-base leading-relaxed text-gray-700">
		{#each renderFormattedBlocks(selected?.definition) as block}
			{#if block.length && block[0].text.trim().startsWith('"')}
				<blockquote class="border-l-4 border-blue-400 bg-blue-50 px-4 py-3 italic text-gray-700 rounded">
					{#each block as part}
						{#if part.type === 'citation' && part.url}
							<a
								href={part.url}
								target="_blank"
								rel="noopener noreferrer"
								class="text-blue-600 hover:underline"
							>
								([{part.text}])
							</a>
						{:else}
							{@html part.text.replace(/\n/g, '<br>')}
						{/if}
					{/each}
				</blockquote>
			{:else}
				<p>
					{#each block as part}
						{#if part.type === 'citation' && part.url}
							<a
								href={part.url}
								target="_blank"
								rel="noopener noreferrer"
								class="text-blue-600 hover:underline"
							>
								([{part.text}])
							</a>
						{:else}
							{@html part.text.replace(/\n/g, '<br>')}
						{/if}
					{/each}
				</p>
			{/if}
		{/each}
	</div>
</div>
<button class="mt-5 float-right bg-green-400 hover:bg-blue-700 text-white font-semibold py-2 px-4 rounded-lg shadow-sm transition-colors duration-200">
	Add Definition
</button>

	</div>
{/if}

