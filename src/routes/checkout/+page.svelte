<script lang="ts">
	import { onMount } from 'svelte';
	import { resolve } from '$app/paths';

	type CartEntry = { label: string; price: number; count: number };

	/** Formats cents to USD, e.g. 999 → "$9.99". */
	function fmt(cents: number): string {
		return '$' + (cents / 100).toFixed(2);
	}

	// Reactive map of cart items loaded from sessionStorage.
	let cartItems: Record<string, CartEntry> = {};
	let loaded = false;
	let ordered = false;

	$: total = Object.values(cartItems).reduce((sum, v) => sum + v.count, 0);
	$: grandTotal = Object.values(cartItems).reduce((sum, v) => sum + v.price * v.count, 0);
	// All entries including zeroed ones (shown faded during grace period)
	$: allEntries = Object.entries(cartItems);
	// At least one positive-count entry means "not empty"
	$: hasItems = allEntries.length > 0;

	// Timers pending hard-removal of zeroed entries (2-second grace period)
	let removalTimers: Record<string, ReturnType<typeof setTimeout>> = {};

	function cancelRemoval(id: string) {
		if (removalTimers[id] !== undefined) {
			clearTimeout(removalTimers[id]);
			const t = { ...removalTimers };
			delete t[id];
			removalTimers = t;
		}
	}

	onMount(() => {
		try {
			const raw = sessionStorage.getItem('checkoutItems');
			if (raw) {
				cartItems = JSON.parse(raw);
			}
		} catch {
			// sessionStorage unavailable or data was invalid
		}
		loaded = true;
	});

	function addOne(id: string) {
		if (!cartItems[id]) return;
		cancelRemoval(id);
		cartItems = { ...cartItems, [id]: { ...cartItems[id], count: cartItems[id].count + 1 } };
	}

	function removeOne(id: string) {
		const current = cartItems[id]?.count ?? 0;
		if (current <= 1) {
			cartItems = { ...cartItems, [id]: { ...cartItems[id], count: 0 } };
			const timer = setTimeout(() => {
				const rest = { ...cartItems };
				delete rest[id];
				cartItems = rest;
				const t = { ...removalTimers };
				delete t[id];
				removalTimers = t;
			}, 2000);
			removalTimers = { ...removalTimers, [id]: timer };
		} else {
			cartItems = { ...cartItems, [id]: { ...cartItems[id], count: current - 1 } };
		}
	}

	function placeOrder() {
		sessionStorage.removeItem('checkoutItems');
		ordered = true;
	}
</script>

<main class="page">
	<header class="top-bar">
		<a href={resolve('/')} class="back-link">← Back to board</a>
		<h1>Order Summary</h1>
		{#if loaded && hasItems && !ordered}
			<p class="subtitle">{total} item{total === 1 ? '' : 's'} selected</p>
		{/if}
	</header>

	{#if !loaded}
		<p class="status">Loading…</p>
	{:else if ordered}
		<div class="success">
			<p class="success-icon">✓</p>
			<h2>Order placed!</h2>
			<p>Thanks for your order of {total} item{total === 1 ? '' : 's'}.</p>
			<a href={resolve('/')} class="back-link" style="margin-top:1rem;display:inline-block">← Back to board</a>
		</div>
	{:else if !hasItems}
		<p class="status">Your cart is empty. <a href={resolve('/')}>Go back to add items.</a></p>
	{:else}
		<ul class="order-list">
			{#each allEntries as [id, entry] (id)}
				<!-- Faded row during the 2-second grace period after reaching zero -->
				<li class="order-entry" class:order-entry-zeroed={entry.count === 0}>
					<div class="order-entry-info">
						<span class="order-label">{entry.label}</span>
						<span class="order-unit-price">{fmt(entry.price)} each</span>
					</div>
					<div class="order-qty-ctrl">
						<button class="qty-btn" on:click={() => removeOne(id)} disabled={entry.count === 0} aria-label="Remove one">−</button>
						<span class="order-qty">{entry.count}</span>
						<button class="qty-btn" on:click={() => addOne(id)} aria-label="Add one">+</button>
						<span class="order-line-total">{fmt(entry.price * entry.count)}</span>
					</div>
				</li>
			{/each}
		</ul>

		<div class="order-footer">
			<div class="order-totals">
				<span class="order-total-label">{total} item{total === 1 ? '' : 's'}</span>
				<span class="order-grand-total">{fmt(grandTotal)}</span>
			</div>
			<button class="place-order-btn" on:click={placeOrder} disabled={total === 0}>Checkout →</button>
		</div>
	{/if}
</main>

<style>
	@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;700&family=IBM+Plex+Mono:wght@400;500&display=swap');

	:global(body) {
		margin: 0;
		background:
			radial-gradient(circle at 10% 8%, rgb(250 208 153 / 0.4), transparent 33%),
			radial-gradient(circle at 90% 90%, rgb(137 195 255 / 0.35), transparent 28%),
			linear-gradient(170deg, #efe8d8 0%, #f8fafc 45%, #e6eef8 100%);
		min-height: 100vh;
	}

	.page {
		max-width: 36rem;
		margin: 0 auto;
		padding: 2.5rem 1.5rem;
		font-family: 'Space Grotesk', 'Trebuchet MS', sans-serif;
		color: #102533;
	}

	.top-bar {
		margin-bottom: 2rem;
	}

	.back-link {
		display: inline-block;
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.82rem;
		color: #0f766e;
		text-decoration: none;
		margin-bottom: 0.75rem;
	}

	.back-link:hover {
		text-decoration: underline;
	}

	h1 {
		margin: 0 0 0.25rem;
		font-size: clamp(1.8rem, 4vw, 2.4rem);
		line-height: 1.1;
	}

	.subtitle {
		margin: 0;
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.8rem;
		color: #666;
	}

	.status {
		color: #666;
		font-size: 0.9rem;
	}

	.status a {
		color: #0f766e;
	}

	/* List of ordered items */
	.order-list {
		list-style: none;
		margin: 0;
		padding: 0;
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	/* Zeroed row: faded + strikethrough during grace period */
	.order-entry-zeroed {
		opacity: 0.4;
	}

	.order-entry-zeroed .order-label {
		text-decoration: line-through;
	}

	.order-entry {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 0.5rem;
		padding: 0.75rem 1rem;
		border-radius: 0.75rem;
		background: rgb(255 255 255 / 0.8);
		box-shadow: 0 2px 8px rgb(0 0 0 / 0.06);
		font-size: 0.88rem;
	}

	/* Left column: name + unit price */
	.order-entry-info {
		flex: 1;
		min-width: 0;
		display: flex;
		flex-direction: column;
		gap: 0.1rem;
	}

	.order-label {
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
	}

	.order-unit-price {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.75rem;
		color: #888;
	}

	/* +/− qty control cluster */
	.order-qty-ctrl {
		display: flex;
		align-items: center;
		gap: 0.35rem;
		margin-left: 1rem;
		flex-shrink: 0;
	}

	.order-qty {
		font-family: 'IBM Plex Mono', monospace;
		font-weight: 700;
		font-size: 0.85rem;
		color: #0f766e;
		min-width: 1.4rem;
		text-align: center;
	}

	/* Shared qty button style */
	.qty-btn {
		width: 1.4rem;
		height: 1.4rem;
		border-radius: 0.35rem;
		border: 1px solid #cbd5e1;
		background: #fff;
		color: #102533;
		font-size: 1rem;
		line-height: 1;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0;
		cursor: pointer;
		transition: background 0.1s;
	}

	.qty-btn:hover {
		background: #f1f5f9;
	}

	/* Line total displayed after the qty buttons */
	.order-line-total {
		font-family: 'IBM Plex Mono', monospace;
		font-weight: 700;
		font-size: 0.85rem;
		color: #0f766e;
		margin-left: 0.3rem;
		flex-shrink: 0;
	}

	/* Footer row: total count + checkout button */
	.order-footer {
		margin-top: 1.25rem;
		border-top: 1px solid #e2e8f0;
		padding-top: 0.75rem;
	}

	/* Totals row: item count left, grand total right */
	.order-totals {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.75rem;
	}

	.order-total-label {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.82rem;
		color: #555;
	}

	.order-grand-total {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 1.1rem;
		font-weight: 700;
		color: #102533;
	}

	/* Checkout / Place Order button */
	.place-order-btn {
		padding: 0.55rem 1.1rem;
		border-radius: 0.6rem;
		border: none;
		background: #102533;
		color: #fff;
		font-family: 'Space Grotesk', 'Trebuchet MS', sans-serif;
		font-size: 0.88rem;
		font-weight: 700;
		cursor: pointer;
		transition: background 0.15s;
	}

	.place-order-btn:hover {
		background: #1e3a4f;
	}

	/* Success state */
	.success {
		text-align: center;
		padding: 2rem 0;
	}

	.success-icon {
		font-size: 3rem;
		margin: 0 0 0.5rem;
		color: #0f766e;
	}

	.success h2 {
		margin: 0 0 0.5rem;
		font-size: 1.6rem;
	}

	.success p {
		color: #555;
		margin: 0;
	}
</style>
