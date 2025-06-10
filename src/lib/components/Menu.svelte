<script>
	import { onMount } from 'svelte';
	let mapsOpen = false;
	let vignettesOpen = false;
	let isMobileOpen = false;

	const closeMenus = () => {
		mapsOpen = false;
		vignettesOpen = false;
	};

	const handleClickOutside = (event) => {
		if (!event.target.closest('.dropdown')) {
			closeMenus();
		}
	};

	onMount(() => {
		document.addEventListener('click', handleClickOutside);
		return () => {
			document.removeEventListener('click', handleClickOutside);
		};
	});
</script>

<nav class="bg-white shadow-md sticky top-0 z-50">
	<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
		<div class="flex justify-between h-16 items-center">
			<!-- Logo + App Name -->
			<div class="flex items-center space-x-2">
				<img src="/network.png" alt="Logo" class="h-8 w-8" />
				<span class="text-xl font-semibold text-gray-800"><a href="/">ConceptMap</a></span>
			</div>

			<!-- Desktop Menu -->
			<div class="hidden md:flex items-center space-x-6">
                		<!-- Other links -->
				<a href="/build" class="text-gray-700 hover:text-blue-600 transition">Build</a>
				<!-- Maps Dropdown -->
				<div class="relative dropdown">
					<button on:click={() => {
						mapsOpen = !mapsOpen;
						vignettesOpen = false;
					}} class="text-gray-700 hover:text-blue-600 transition">
						Maps
					</button>
					{#if mapsOpen}
						<div class="absolute mt-2 bg-white border rounded-md shadow-md w-56 z-10 py-2">
							<a href="/diversity" class="block px-4 py-2 text-sm hover:bg-gray-100 text-gray-800">Diversity</a>
							<span class="block px-4 py-2 text-sm text-gray-400 cursor-not-allowed">Normativity (coming soon)</span>
						</div>
					{/if}
				</div>

				<!-- Vignettes Dropdown -->
				<div class="relative dropdown">
					<button on:click={() => {
						vignettesOpen = !vignettesOpen;
						mapsOpen = false;
					}} class="text-gray-700 hover:text-blue-600 transition">
						Vignettes
					</button>
					{#if vignettesOpen}
						<div class="absolute mt-2 bg-white border rounded-md shadow-md w-64 z-10 py-2">
							<a href="/vignettes/how-to-build" class="block px-4 py-2 text-sm hover:bg-gray-100 text-gray-800">How to build a map</a>
							<a href="/vignettes/how-to-contribute" class="block px-4 py-2 text-sm hover:bg-gray-100 text-gray-800">How to contribute to a map</a>
						</div>
					{/if}
				</div>

		
			</div>

			<!-- Mobile Toggle -->
			<div class="md:hidden">
				<button on:click={() => isMobileOpen = !isMobileOpen} aria-label="Toggle menu">
					<svg class="w-6 h-6 text-gray-700" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						{#if !isMobileOpen}
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
						{:else}
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
						{/if}
					</svg>
				</button>
			</div>
		</div>
	</div>

	<!-- Mobile Dropdown -->
	{#if isMobileOpen}
		<div class="md:hidden px-4 pb-4 space-y-4 bg-white border-t border-gray-200">
			<div>
				<p class="font-medium text-gray-900">Maps</p>
				<a href="/diversity" class="block text-gray-700 pl-4 py-1">Diversity</a>
				<span class="block text-gray-400 pl-4 py-1">Normativity (coming soon)</span>
			</div>
			<div>
				<p class="font-medium text-gray-900">Vignettes</p>
				<a href="/vignettes/how-to-build" class="block text-gray-700 pl-4 py-1">How to build a map</a>
				<a href="/vignettes/how-to-contribute" class="block text-gray-700 pl-4 py-1">How to contribute to a map</a>
			</div>
			<a href="/about" class="block text-gray-700">About</a>
		</div>
	{/if}
</nav>

<footer class="fixed bottom-0 right-0 text-black text-justify float-right w-1/2">
				<div class="flex items-center space-x-2">
				<img src="/network.png" alt="Logo" class="h-8 w-8" />
				<span class="text-xl font-semibold text-gray-800"><a href="/">ConceptMap</a></span>
			</div>
				
				<i>This map was created based primarily on readings from an interdisciplinary, international, online reading group, beginning in mid-2024. We thank everyone who came along and shared their insights and ideas and contributed to a series of enlightening and provocative discussions.</i>
		
</footer>