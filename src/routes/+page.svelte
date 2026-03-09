<script lang="ts">
	type PaneItem = {
		id: string;
		label: string;
		note: string;
	};

	type Pane = {
		id: string;
		title: string;
		accent: string;
		items: PaneItem[];
	};

	type PaneRow = {
		id: string;
		panes: Pane[];
	};

	const GRID_COLUMNS = 4;
	const GRID_ROWS = 12;
	const VISIBLE_GRID_ROWS = 8;

	let paneStackElement: HTMLDivElement | null = null;

	function handoffGridWheel(event: WheelEvent) {
		if (!paneStackElement || event.deltaY === 0) {
			return;
		}

		const grid = event.currentTarget as HTMLDivElement;
		const atTop = grid.scrollTop <= 0;
		const atBottom = grid.scrollTop + grid.clientHeight >= grid.scrollHeight - 1;
		const scrollingUp = event.deltaY < 0;
		const scrollingDown = event.deltaY > 0;

		if (!((scrollingUp && atTop) || (scrollingDown && atBottom))) {
			return;
		}

		const stackAtTop = paneStackElement.scrollTop <= 0;
		const stackAtBottom =
			paneStackElement.scrollTop + paneStackElement.clientHeight >= paneStackElement.scrollHeight - 1;

		if ((scrollingUp && stackAtTop) || (scrollingDown && stackAtBottom)) {
			return;
		}

		event.preventDefault();
		paneStackElement.scrollBy({ top: event.deltaY, behavior: 'auto' });
	}

	function createGridItems(prefix: string): PaneItem[] {
		return Array.from({ length: GRID_COLUMNS * GRID_ROWS }, (_, index) => {
			const cardNumber = index + 1;
			const row = Math.floor(index / GRID_COLUMNS) + 1;
			const col = (index % GRID_COLUMNS) + 1;

			return {
				id: `${prefix}-${cardNumber}`,
				label: `Card ${cardNumber}`,
				note: `R${row} C${col}`
			};
		});
	}

	function createPane(id: string, title: string, accent: string): Pane {
		return {
			id,
			title,
			accent,
			items: createGridItems(id)
		};
	}

	const paneRows: PaneRow[] = [
		{
			id: 'row-a',
			panes: [
				createPane('strategy', 'Strategy Lane', '#0f766e'),
				createPane('build', 'Build Lane', '#b45309'),
				createPane('review', 'Review Lane', '#1d4ed8'),
				createPane('ops', 'Ops Lane', '#7c3aed')
			]
		},
		{
			id: 'row-b',
			panes: [
				createPane('design', 'Design Lane', '#be185d'),
				createPane('content', 'Content Lane', '#0369a1'),
				createPane('growth', 'Growth Lane', '#ca8a04'),
				createPane('support', 'Support Lane', '#166534')
			]
		}
	];
</script>

<main class="canvas">
	<section class="hero">
		<p class="eyebrow">SvelteKit Pane Board</p>
		<h1>Two pane rows with horizontal scrolling and 12 card rows per pane.</h1>
		<p>
			Vertical scrolling happens across pane rows, and each pane shows 8 visible card rows out of 12.
		</p>
	</section>

	<div class="pane-stack" bind:this={paneStackElement} aria-label="Two rows of horizontally scrollable panes">
		{#each paneRows as row, rowIndex (row.id)}
			<section class="pane-row" aria-label={`Pane row ${rowIndex + 1}`}>
				{#each row.panes as pane (pane.id)}
					<article class="pane" style={`--accent:${pane.accent}`}>
						<header class="pane-head">
							<h2>{pane.title}</h2>
							<p>{GRID_COLUMNS}x{GRID_ROWS} (show {VISIBLE_GRID_ROWS})</p>
						</header>

						<div class="item-grid" on:wheel={handoffGridWheel} aria-label={`${pane.title} items`}>
							{#each pane.items as item (item.id)}
								<div class="card">
									<p class="card-label">{item.label}</p>
									<p class="card-note">{item.note}</p>
								</div>
							{/each}
						</div>
					</article>
				{/each}
			</section>
		{/each}
	</div>
</main>

<style>
	@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;700&family=IBM+Plex+Mono:wght@400;500&display=swap');

	:global(body) {
		margin: 0;
		overflow: hidden;
	}

	.canvas {
		--ink: #102533;
		--shadow: 0 16px 30px rgb(6 33 56 / 0.1);
		--pane-width: 22rem;
		--pane-height: 35rem;
		--card-row-height: 3.2rem;
		--grid-gap: 0.45rem;
		height: 100vh;
		display: grid;
		grid-template-rows: auto 1fr;
		gap: 0.85rem;
		overflow: hidden;
		padding: clamp(1rem, 2vw + 0.8rem, 2.4rem);
		font-family: 'Space Grotesk', 'Trebuchet MS', sans-serif;
		color: var(--ink);
		background:
			radial-gradient(circle at 10% 8%, rgb(250 208 153 / 0.4), transparent 33%),
			radial-gradient(circle at 90% 90%, rgb(137 195 255 / 0.35), transparent 28%),
			linear-gradient(170deg, #efe8d8 0%, #f8fafc 45%, #e6eef8 100%);
	}

	.hero {
		max-width: 70ch;
		margin-inline: auto;
		text-align: center;
	}

	.eyebrow {
		font-family: 'IBM Plex Mono', monospace;
		letter-spacing: 0.11em;
		font-size: 0.76rem;
		text-transform: uppercase;
		margin-bottom: 0.55rem;
	}

	h1 {
		font-size: clamp(1.6rem, 2.4vw + 1rem, 2.8rem);
		line-height: 1.1;
		margin: 0;
	}

	.hero p {
		margin-top: 0.8rem;
		line-height: 1.45;
	}

	.pane-stack {
		display: grid;
		grid-auto-rows: max-content;
		align-content: start;
		gap: 1rem;
		min-height: 0;
		overflow-y: auto;
		padding: 0.2rem 0.2rem 0.4rem;
		scrollbar-width: thin;
	}

	.pane-row {
		display: flex;
		gap: 1rem;
		min-height: 0;
		overflow-x: auto;
		overflow-y: hidden;
		scroll-snap-type: x proximity;
		padding: 0.2rem 0.2rem 0.9rem;
		scrollbar-width: thin;
		overscroll-behavior-x: contain;
		overscroll-behavior-y: auto;
	}

	.pane {
		display: grid;
		grid-template-rows: auto 1fr;
		gap: 0.9rem;
		padding: 1rem;
		flex: 0 0 var(--pane-width);
		height: var(--pane-height);
		min-height: 0;
		border-radius: 1rem;
		background: linear-gradient(180deg, rgb(255 255 255 / 0.86), rgb(255 255 255 / 0.72));
		box-shadow: var(--shadow);
		border: 2px solid color-mix(in srgb, var(--accent) 55%, white);
		backdrop-filter: blur(3px);
		scroll-snap-align: start;
	}

	.pane-head {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	h2 {
		margin: 0;
		font-size: 1.05rem;
	}

	.pane-head p {
		margin: 0;
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.78rem;
	}

	.item-grid {
		display: grid;
		gap: var(--grid-gap);
		grid-template-columns: repeat(4, minmax(0, 1fr));
		grid-template-rows: repeat(12, minmax(var(--card-row-height), auto));
		align-content: start;
		height: calc(
			(var(--card-row-height) * 8) + (var(--grid-gap) * 7)
		);
		overflow-y: auto;
		min-height: 0;
		padding-right: 0.2rem;
		scrollbar-width: thin;
	}

	.card {
		padding: 0.45rem;
		border-radius: 0.85rem;
		background: color-mix(in srgb, var(--accent) 9%, white);
		border: 1px solid color-mix(in srgb, var(--accent) 26%, white);
	}

	.card-label {
		margin: 0;
		font-weight: 700;
		font-size: 0.78rem;
	}

	.card-note {
		margin: 0.15rem 0 0;
		font-size: 0.7rem;
		line-height: 1.2;
		font-family: 'IBM Plex Mono', monospace;
	}

	@media (max-width: 760px) {
		.canvas {
			padding-inline: 0.8rem;
			padding-block: 0.7rem;
			grid-template-rows: auto 1fr;
		}
	}
</style>
