<script lang="ts">
	import { goto } from '$app/navigation';
	import { resolve } from '$app/paths';

	// A single card displayed inside a pane's grid.
	type PaneItem = {
		id: string;          // Unique key, e.g. "strategy-1"
		label: string;       // Product name, e.g. "Whole Milk"
		icon: string;        // Emoji shown on the card, e.g. "🥛"
		note: string;        // Package size shown in the detail drawer, e.g. "1 gal"
		price: number;       // Unit price in USD cents, e.g. 999 = $9.99
		description: string; // Extended detail shown in the drawer
	};

	// A vertical column (lane) that holds a scrollable grid of cards.
	type Pane = {
		id: string;       // Unique key, e.g. "strategy"
		title: string;    // Column heading, e.g. "Strategy Lane"
		accent: string;   // CSS color used for border / card tints
		items: PaneItem[]; // All cards in this pane
	};

	// One horizontal band that groups several panes side-by-side.
	type PaneRow = {
		id: string;    // Unique key, e.g. "row-a"
		panes: Pane[]; // The panes laid out left-to-right in this row
	};

	// ─── Grid dimensions ────────────────────────────────────────────────────────
	const GRID_COLUMNS = 4;        // Cards per row inside each pane
	const GRID_ROWS = 12;          // Total card rows generated per pane
	const VISIBLE_GRID_ROWS = 8;   // How many rows are visible before scrolling

	// Bound to the outer scrollable container that stacks pane rows vertically.
	let paneStackElement: HTMLDivElement | null = null;

	// Per-item cart data: maps item id → { label, price, count }.
	let cartItems: Record<string, { label: string; price: number; count: number }> = {};

	// Total number of individual items across all cart entries.
	$: cartCount = Object.values(cartItems).reduce((sum, v) => sum + v.count, 0);

	// Grand total price in cents (only positive-count items).
	$: cartTotal = Object.values(cartItems).reduce((sum, v) => sum + v.price * v.count, 0);

	/** Formats cents to a USD dollar string, e.g. 999 → "$9.99". */
	function fmt(cents: number): string {
		return '$' + (cents / 100).toFixed(2);
	}

	// Whether the cart panel dropdown is open.
	let cartOpen = false;

	// Whether the context menu is open.
	let menuOpen = false;

	// The item currently shown in the detail drawer (null = drawer closed).
	let activeItem: PaneItem | null = null;
	let activePane: Pane | null = null;

	/** Opens the detail drawer for a given item. Closes other panels. */
	function openDetail(item: PaneItem, pane: Pane) {
		activeItem = item;
		activePane = pane;
		cartOpen = false;
		menuOpen = false;
	}

	/** Closes the detail drawer. */
	function closeDetail() {
		activeItem = null;
		activePane = null;
	}

	// Timers pending hard-removal of zeroed entries (2-second grace period).
	let removalTimers: Record<string, ReturnType<typeof setTimeout>> = {};

	/** Cancels any pending removal timer for a given id. */
	function cancelRemoval(id: string) {
		if (removalTimers[id] !== undefined) {
			clearTimeout(removalTimers[id]);
			const t = { ...removalTimers };
			delete t[id];
			removalTimers = t;
		}
	}

	/** Adds one of the given item to the cart; label includes pane name for clarity. */
	function addToCart(item: PaneItem, pane: Pane) {
		cancelRemoval(item.id);
		cartItems = {
			...cartItems,
			[item.id]: {
				label: `${item.label} — ${pane.title}`,
				price: item.price,
				count: (cartItems[item.id]?.count ?? 0) + 1
			}
		};
	}

	/** Increments count of an already-carted item by id (used from cart panel). */
	function addOneById(id: string) {
		if (!cartItems[id]) return;
		cancelRemoval(id);
		cartItems = { ...cartItems, [id]: { ...cartItems[id], count: cartItems[id].count + 1 } };
	}

	/** Decrements count; zeros it out and schedules removal after 2 s instead of deleting immediately. */
	function removeFromCart(id: string) {
		const current = cartItems[id]?.count ?? 0;
		if (current <= 1) {
			// Zero out the entry so it lingers visually
			cartItems = { ...cartItems, [id]: { ...cartItems[id], count: 0 } };
			// Schedule hard removal after 2 seconds
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

	/** Toggles the cart panel; closes the menu if open. */
	function toggleCart() {
		cartOpen = !cartOpen;
		if (cartOpen) menuOpen = false;
	}

	/** Toggles the context menu; closes the cart if open. */
	function toggleMenu() {
		menuOpen = !menuOpen;
		if (menuOpen) cartOpen = false;
	}

	/** Closes all panels (used by the shared backdrop). */
	function closeAll() {
		cartOpen = false;
		menuOpen = false;
		activeItem = null;
		activePane = null;
	}

	/** Saves cart to sessionStorage (only positive-count items) and navigates to the checkout page. */
	function checkoutCart() {
		cartOpen = false;
		const toSave = Object.fromEntries(
			Object.entries(cartItems).filter(([, v]) => v.count > 0)
		);
		sessionStorage.setItem('checkoutItems', JSON.stringify(toSave));
		goto(resolve('/checkout'));
	}

	/**
	 * Wheel-event bridge: when a card grid reaches its scroll boundary (top or
	 * bottom), forward the remaining delta to the outer pane-stack so the user
	 * can continue scrolling between rows without lifting the mouse.
	 *
	 * Without this, the wheel event is consumed by the inner grid and the outer
	 * container never moves.
	 */
	function handoffGridWheel(event: WheelEvent) {
		// Nothing to do if the outer container isn't mounted or there's no vertical delta.
		if (!paneStackElement || event.deltaY === 0) {
			return;
		}

		const grid = event.currentTarget as HTMLDivElement;

		// Detect whether the inner grid has reached either scroll boundary.
		const atTop = grid.scrollTop <= 0;
		const atBottom = grid.scrollTop + grid.clientHeight >= grid.scrollHeight - 1;
		const scrollingUp = event.deltaY < 0;
		const scrollingDown = event.deltaY > 0;

		// If the grid still has room to scroll in the desired direction, let it.
		if (!((scrollingUp && atTop) || (scrollingDown && atBottom))) {
			return;
		}

		// Also check whether the outer stack itself is already at its boundary;
		// if so, there's nowhere to scroll to, so don't steal the event.
		const stackAtTop = paneStackElement.scrollTop <= 0;
		const stackAtBottom =
			paneStackElement.scrollTop + paneStackElement.clientHeight >= paneStackElement.scrollHeight - 1;

		if ((scrollingUp && stackAtTop) || (scrollingDown && stackAtBottom)) {
			return;
		}

		// Block the default browser scroll on the inner grid and drive the outer one.
		event.preventDefault();
		paneStackElement.scrollBy({ top: event.deltaY, behavior: 'auto' });
	}

	// Shape of a single product definition used to seed a pane's item list.
	type ProductDef = { label: string; icon: string; note: string };

	/** Generates GRID_COLUMNS × GRID_ROWS PaneItems by cycling through the given product list. */
	function createItems(id: string, products: ProductDef[]): PaneItem[] {
		const descriptions = [
			'Fresh and wholesome — a pantry staple loved by families everywhere.',
			'Farm-fresh quality you can taste in every bite. A household favorite.',
			'Carefully selected for peak freshness. Stock up and save.',
			'A trusted brand with a rich flavor profile. Perfect for any recipe.',
		];
		return Array.from({ length: GRID_COLUMNS * GRID_ROWS }, (_, index) => {
			const p = products[index % products.length];
			const price = ((index % 12) + 1) * 199 + (index % 7) * 100;
			const description = descriptions[index % descriptions.length];
			return { id: `${id}-${index + 1}`, label: p.label, icon: p.icon, note: p.note, price, description };
		});
	}

	/** Builds a Pane object using category-specific product data. */
	function createPane(id: string, title: string, accent: string, products: ProductDef[]): Pane {
		return { id, title, accent, items: createItems(id, products) };
	}

	// ─── Category product data ───────────────────────────────────────────────────

	const dairyProducts: ProductDef[] = [
		{ label: 'Whole Milk',         icon: '🥛', note: '1 gal'    },
		{ label: 'Skim Milk',          icon: '🥛', note: '½ gal'    },
		{ label: '2% Milk',            icon: '🥛', note: '1 gal'    },
		{ label: 'Oat Milk',           icon: '🥛', note: '32 oz'    },
		{ label: 'Almond Milk',        icon: '🥛', note: '64 oz'    },
		{ label: 'Butter',             icon: '🧈', note: '16 oz'    },
		{ label: 'Unsalted Butter',    icon: '🧈', note: '16 oz'    },
		{ label: 'Cheddar Cheese',     icon: '🧀', note: '8 oz'     },
		{ label: 'Mozzarella',         icon: '🧀', note: '8 oz'     },
		{ label: 'Parmesan',           icon: '🧀', note: '6 oz'     },
		{ label: 'Greek Yogurt',       icon: '🫙', note: '32 oz'    },
		{ label: 'Cream Cheese',       icon: '🫙', note: '8 oz'     },
		{ label: 'Sour Cream',         icon: '🫙', note: '16 oz'    },
		{ label: 'Heavy Cream',        icon: '🥄', note: '1 pt'     },
		{ label: 'Eggs',               icon: '🥚', note: '12 ct'    },
		{ label: 'Large Eggs',         icon: '🥚', note: '18 ct'    },
	];

	const meatProducts: ProductDef[] = [
		{ label: 'Chicken Breast',     icon: '🍗', note: '2 lb'     },
		{ label: 'Chicken Thighs',     icon: '🍗', note: '1.5 lb'   },
		{ label: 'Ground Beef',        icon: '🥩', note: '1 lb'     },
		{ label: 'Beef Steak',         icon: '🥩', note: '10 oz'    },
		{ label: 'Pork Chops',         icon: '🥩', note: '1.5 lb'   },
		{ label: 'Bacon',              icon: '🥓', note: '12 oz'    },
		{ label: 'Ham Steak',          icon: '🥓', note: '8 oz'     },
		{ label: 'Hot Dogs',           icon: '🌭', note: '8 ct'     },
		{ label: 'Ground Turkey',      icon: '🍗', note: '1 lb'     },
		{ label: 'Salmon Fillet',      icon: '🐟', note: '6 oz'     },
		{ label: 'Cod Fillet',         icon: '🐟', note: '6 oz'     },
		{ label: 'Shrimp',             icon: '🦐', note: '12 oz'    },
		{ label: 'Tilapia',            icon: '🐟', note: '8 oz'     },
		{ label: 'Lamb Chops',         icon: '🥩', note: '1 lb'     },
		{ label: 'Pork Sausages',      icon: '🌭', note: '14 oz'    },
		{ label: 'Crab Cakes',         icon: '🦀', note: '4 ct'     },
	];

	const produceProducts: ProductDef[] = [
		{ label: 'Bananas',            icon: '🍌', note: '3 lb'     },
		{ label: 'Apples',             icon: '🍎', note: '2 lb'     },
		{ label: 'Oranges',            icon: '🍊', note: '4 ct'     },
		{ label: 'Strawberries',       icon: '🍓', note: '1 lb'     },
		{ label: 'Grapes',             icon: '🍇', note: '2 lb'     },
		{ label: 'Blueberries',        icon: '🫐', note: '1 pt'     },
		{ label: 'Broccoli',           icon: '🥦', note: '1 head'   },
		{ label: 'Baby Spinach',       icon: '🥬', note: '5 oz'     },
		{ label: 'Carrots',            icon: '🥕', note: '2 lb'     },
		{ label: 'Tomatoes',           icon: '🍅', note: '1 lb'     },
		{ label: 'Bell Peppers',       icon: '🫑', note: '3 ct'     },
		{ label: 'Avocado',            icon: '🥑', note: '2 ct'     },
		{ label: 'Onions',             icon: '🧅', note: '3 lb'     },
		{ label: 'Garlic',             icon: '🧄', note: '3 ct'     },
		{ label: 'Potatoes',           icon: '🥔', note: '5 lb'     },
		{ label: 'Cucumber',           icon: '🥒', note: '1 ct'     },
	];

	const bakeryProducts: ProductDef[] = [
		{ label: 'White Bread',        icon: '🍞', note: '20 oz'    },
		{ label: 'Wheat Bread',        icon: '🍞', note: '24 oz'    },
		{ label: 'Sourdough Loaf',     icon: '🍞', note: '24 oz'    },
		{ label: 'Rye Bread',          icon: '🍞', note: '16 oz'    },
		{ label: 'Baguette',           icon: '🥖', note: '12 oz'    },
		{ label: 'Ciabatta',           icon: '🥖', note: '14 oz'    },
		{ label: 'Bagels',             icon: '🥯', note: '6 ct'     },
		{ label: 'Croissants',         icon: '🥐', note: '4 ct'     },
		{ label: 'Cinnamon Rolls',     icon: '🥐', note: '4 ct'     },
		{ label: 'Blueberry Muffins',  icon: '🧁', note: '4 ct'     },
		{ label: 'Chocolate Muffins',  icon: '🧁', note: '4 ct'     },
		{ label: 'Dinner Rolls',       icon: '🫓', note: '12 ct'    },
		{ label: 'Pita Bread',         icon: '🫓', note: '8 ct'     },
		{ label: 'Flour Tortillas',    icon: '🫓', note: '10 ct'    },
		{ label: 'English Muffins',    icon: '🫓', note: '6 ct'     },
		{ label: 'Pretzel Rolls',      icon: '🥨', note: '6 ct'     },
	];

	const cannedProducts: ProductDef[] = [
		{ label: 'Tomato Sauce',       icon: '🥫', note: '24 oz'    },
		{ label: 'Diced Tomatoes',     icon: '🥫', note: '14.5 oz'  },
		{ label: 'Tomato Soup',        icon: '🥫', note: '10.75 oz' },
		{ label: 'Chicken Broth',      icon: '🫙', note: '32 oz'    },
		{ label: 'Beef Broth',         icon: '🫙', note: '32 oz'    },
		{ label: 'Black Beans',        icon: '🥫', note: '15 oz'    },
		{ label: 'Kidney Beans',       icon: '🥫', note: '15 oz'    },
		{ label: 'Chickpeas',          icon: '🥫', note: '15 oz'    },
		{ label: 'Sweet Corn',         icon: '🥫', note: '15 oz'    },
		{ label: 'Green Beans',        icon: '🥫', note: '14.5 oz'  },
		{ label: 'Tuna in Water',      icon: '🐟', note: '5 oz'     },
		{ label: 'Coconut Milk',       icon: '🥥', note: '13.5 oz'  },
		{ label: 'Pumpkin Puree',      icon: '🎃', note: '15 oz'    },
		{ label: 'Salsa',              icon: '🫙', note: '16 oz'    },
		{ label: 'Sardines',           icon: '🐟', note: '3.75 oz'  },
		{ label: 'Artichoke Hearts',   icon: '🥫', note: '14 oz'    },
	];

	const pastaProducts: ProductDef[] = [
		{ label: 'Spaghetti',          icon: '🍝', note: '16 oz'    },
		{ label: 'Penne',              icon: '🍝', note: '12 oz'    },
		{ label: 'Fettuccine',         icon: '🍝', note: '12 oz'    },
		{ label: 'Rigatoni',           icon: '🍝', note: '12 oz'    },
		{ label: 'Linguine',           icon: '🍝', note: '12 oz'    },
		{ label: 'Bowtie Pasta',       icon: '🍝', note: '12 oz'    },
		{ label: 'Long Grain Rice',    icon: '🍚', note: '2 lb'     },
		{ label: 'Brown Rice',         icon: '🍚', note: '2 lb'     },
		{ label: 'Jasmine Rice',       icon: '🍚', note: '2 lb'     },
		{ label: 'Basmati Rice',       icon: '🍚', note: '2 lb'     },
		{ label: 'Rolled Oats',        icon: '🥣', note: '42 oz'    },
		{ label: 'Quick Oats',         icon: '🥣', note: '18 oz'    },
		{ label: 'Quinoa',             icon: '🌾', note: '16 oz'    },
		{ label: 'Couscous',           icon: '🌾', note: '10 oz'    },
		{ label: 'Ramen Noodles',      icon: '🍜', note: '3 oz'     },
		{ label: 'Mac & Cheese',       icon: '🍜', note: '7.25 oz'  },
	];

	const beverageProducts: ProductDef[] = [
		{ label: 'Orange Juice',       icon: '🍊', note: '52 oz'    },
		{ label: 'Apple Juice',        icon: '🍎', note: '64 oz'    },
		{ label: 'Lemonade',           icon: '🍋', note: '52 oz'    },
		{ label: 'Sparkling Water',    icon: '💧', note: '1 L'      },
		{ label: 'Coconut Water',      icon: '🥥', note: '33.8 oz'  },
		{ label: 'Grape Juice',        icon: '🍇', note: '64 oz'    },
		{ label: 'Cranberry Juice',    icon: '🍒', note: '64 oz'    },
		{ label: 'Iced Tea',           icon: '🧋', note: '52 oz'    },
		{ label: 'Green Tea',          icon: '🍵', note: '16 ct'    },
		{ label: 'Coffee Grounds',     icon: '☕', note: '12 oz'    },
		{ label: 'Espresso Pods',      icon: '☕', note: '10 ct'    },
		{ label: 'Hot Chocolate Mix',  icon: '☕', note: '10 oz'    },
		{ label: 'Sports Drink',       icon: '🏃', note: '32 oz'    },
		{ label: 'Protein Shake',      icon: '💪', note: '11 oz'    },
		{ label: 'Kombucha',           icon: '🧃', note: '16 oz'    },
		{ label: 'Tomato Juice',       icon: '🍅', note: '46 oz'    },
	];

	const snackProducts: ProductDef[] = [
		{ label: 'Potato Chips',       icon: '🥔', note: '8 oz'     },
		{ label: 'Tortilla Chips',     icon: '🌮', note: '13 oz'    },
		{ label: 'Pretzels',           icon: '🥨', note: '16 oz'    },
		{ label: 'Popcorn',            icon: '🍿', note: '3 ct'     },
		{ label: 'Granola Bars',       icon: '🌾', note: '6 ct'     },
		{ label: 'Trail Mix',          icon: '🥜', note: '9 oz'     },
		{ label: 'Chocolate Bar',      icon: '🍫', note: '3.5 oz'   },
		{ label: 'Dark Chocolate',     icon: '🍫', note: '3.5 oz'   },
		{ label: 'Gummy Bears',        icon: '🐻', note: '5 oz'     },
		{ label: 'Jelly Beans',        icon: '🫘', note: '9 oz'     },
		{ label: 'Oreo Cookies',       icon: '🍪', note: '14.3 oz'  },
		{ label: 'Crackers',           icon: '🍘', note: '13.7 oz'  },
		{ label: 'Rice Cakes',         icon: '🍘', note: '4.5 oz'   },
		{ label: 'Mixed Nuts',         icon: '🥜', note: '16 oz'    },
		{ label: 'Caramel Popcorn',    icon: '🍿', note: '7 oz'     },
		{ label: 'Peanut Butter Cups', icon: '🥜', note: '1.5 oz'   },
	];

	// ─── Static data ────────────────────────────────────────────────────────────
	// Two horizontal rows, each containing four themed panes.
	const paneRows: PaneRow[] = [
		{
			id: 'row-a',
			panes: [
				createPane('dairy',   'Dairy & Eggs',   '#0f766e', dairyProducts),
				createPane('meat',    'Meat & Seafood', '#b45309', meatProducts),
				createPane('produce', 'Fresh Produce',  '#1d4ed8', produceProducts),
				createPane('bakery',  'Bakery',         '#7c3aed', bakeryProducts),
			]
		},
		{
			id: 'row-b',
			panes: [
				createPane('canned',    'Canned & Pantry', '#be185d', cannedProducts),
				createPane('pasta',     'Pasta & Grains',  '#0369a1', pastaProducts),
				createPane('beverages', 'Beverages',       '#ca8a04', beverageProducts),
				createPane('snacks',    'Snacks & Sweets', '#166534', snackProducts),
			]
		}
	];
</script>

<main class="canvas">
	<!-- Introductory text at the top of the page -->
	<section class="hero">
		<p class="eyebrow">SvelteKit Pane Board</p>
		<h1>Two pane rows with horizontal scrolling and 12 card rows per pane.</h1>
		<p>
			Vertical scrolling happens across pane rows, and each pane shows 8 visible card rows out of 12.
		</p>
	</section>

	<!--
		Outer vertically-scrollable container.
		bind:this captures the DOM node so handoffGridWheel can drive it
		programmatically when an inner grid reaches its edge.
	-->
	<div class="pane-stack" bind:this={paneStackElement} aria-label="Two rows of horizontally scrollable panes">

		<!-- Iterate over the two horizontal row bands (row-a, row-b) -->
		{#each paneRows as row, rowIndex (row.id)}

			<!-- Each row scrolls horizontally and snaps to pane boundaries -->
			<section class="pane-row" aria-label={`Pane row ${rowIndex + 1}`}>

				<!-- Each pane is a themed column; --accent CSS variable drives its color scheme -->
				{#each row.panes as pane (pane.id)}
					<article class="pane" style={`--accent:${pane.accent}`}>

						<!-- Column header: title on the left, grid-size summary on the right -->
						<header class="pane-head">
							<h2>{pane.title}</h2>
							<p>{GRID_COLUMNS}x{GRID_ROWS} (show {VISIBLE_GRID_ROWS})</p>
						</header>

						<!--
							Card grid: scrolls vertically inside the fixed pane height.
							on:wheel calls handoffGridWheel to bubble overflow scroll
							up to the pane-stack when this grid is at its boundary.
						-->
						<div class="item-grid" on:wheel={handoffGridWheel} aria-label={`${pane.title} items`}>
							{#each pane.items as item (item.id)}
								<!--
									Card body is a button so keyboard/click opens the detail drawer.
									The + / − buttons use stopPropagation so they don't also open it.
								-->
								<div class="card" role="button" tabindex="0"
									on:click={() => openDetail(item, pane)}
									on:keydown={(e) => e.key === 'Enter' && openDetail(item, pane)}
									aria-label="View details for {item.label}">
									<!-- Count badge: only visible once this card has been added to the cart -->
									{#if (cartItems[item.id]?.count ?? 0) > 0}
										<span class="card-count">{cartItems[item.id].count}</span>
									{/if}
									<span class="card-icon" aria-hidden="true">{item.icon}</span>
									<p class="card-label">{item.label}</p>
									<p class="card-price">{fmt(item.price)}</p>
									<!-- Plus button: top-right of card -->
									<button class="add-btn" on:click|stopPropagation={() => addToCart(item, pane)} aria-label="Add {item.label} to cart">+</button>
									<!-- Minus button: bottom-right, only when count > 0 -->
									{#if (cartItems[item.id]?.count ?? 0) > 0}
										<button class="sub-btn" on:click|stopPropagation={() => removeFromCart(item.id)} aria-label="Remove {item.label} from cart">−</button>
									{/if}
								</div>
							{/each}
						</div>
					</article>
				{/each}
			</section>
		{/each}
	</div>

	<!-- Shared backdrop: clicking anywhere outside panels closes everything -->
	{#if cartOpen || menuOpen || activeItem}
		<div class="panel-backdrop" on:click={closeAll} aria-hidden="true"></div>
	{/if}

	<!-- ── Detail drawer ──────────────────────────────────────────────────────── -->
	{#if activeItem && activePane}
		{@const item = activeItem}
		{@const pane = activePane}
		<div class="detail-drawer" role="dialog" aria-label="Product detail: {item.label}" style="--drawer-accent:{pane.accent}">
			<!-- Header -->
			<header class="detail-head">
				<div>
					<span class="detail-category" style="background:color-mix(in srgb,{pane.accent} 12%,white);color:{pane.accent}">
						{pane.title}
					</span>
					<h2 class="detail-title">{item.label}</h2>
				</div>
				<button class="detail-close" on:click={closeDetail} aria-label="Close detail">✕</button>
			</header>

			<!-- Body -->
			<div class="detail-body">
				<!-- Price -->
				<p class="detail-price">{fmt(item.price)}</p>

				<!-- Meta pill -->
				<p class="detail-meta">{item.note}</p>

				<!-- Description -->
				<p class="detail-desc">{item.description}</p>
			</div>

			<!-- Footer: qty controls + Add to Cart -->
			<footer class="detail-footer">
				{#if (cartItems[item.id]?.count ?? 0) > 0}
					<div class="detail-qty">
						<button class="detail-qty-btn" on:click={() => removeFromCart(item.id)} disabled={(cartItems[item.id]?.count ?? 0) === 0} aria-label="Remove one">−</button>
						<span class="detail-qty-count">{cartItems[item.id]?.count ?? 0}</span>
						<button class="detail-qty-btn" on:click={() => addOneById(item.id)} aria-label="Add one">+</button>
					</div>
				{/if}
				<button class="detail-add-btn" on:click={() => addToCart(item, pane)}
					style="background:{pane.accent}">
					Add to Cart &mdash; {fmt(item.price)}
				</button>
			</footer>
		</div>
	{/if}

	<!-- Context menu: fixed top-left -->
	<div class="menu-wrap">
		<button class="menu-btn" on:click={toggleMenu} aria-label="Menu, {menuOpen ? 'open' : 'closed'}" aria-expanded={menuOpen}>
			<span class="burger"></span>
		</button>
		{#if menuOpen}
			<div class="menu-panel" role="dialog" aria-label="Navigation menu">
				<nav class="menu-nav">
					<a href={resolve('/')} class="menu-link" on:click={closeAll}>Home</a>
					<a href={resolve('/demo')} class="menu-link" on:click={closeAll}>Demo</a>
				</nav>
				<hr class="menu-divider" />
				<button class="menu-action" on:click={() => { cartItems = {}; removalTimers = {}; closeAll(); }}>Clear cart</button>
			</div>
		{/if}
	</div>

	<!--
		Fixed cart widget: always visible in the top-right corner.
		aria-live="polite" announces badge changes to screen readers.
	-->
	<div class="cart-wrap" aria-live="polite">
		<!-- Clicking opens/closes the cart panel -->
		<button class="cart-btn" on:click={toggleCart} aria-label="Shopping cart, {cartCount} item{cartCount === 1 ? '' : 's'}, {cartOpen ? 'open' : 'closed'}">
			<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
				<circle cx="9" cy="21" r="1"/>
				<circle cx="20" cy="21" r="1"/>
				<path d="M1 1h4l2.68 13.39a2 2 0 0 0 2 1.61h9.72a2 2 0 0 0 2-1.61L23 6H6"/>
			</svg>
		</button>
		<!-- Badge only shown when at least one item has been added -->
		{#if cartCount > 0}
			<span class="cart-badge">{cartCount}</span>
		{/if}

		<!-- Dropdown panel listing every item and its quantity -->
		{#if cartOpen}
			<div class="cart-panel" role="dialog" aria-label="Cart contents">
				<header class="cart-panel-head">
					<h3>Cart</h3>
					<button class="cart-close" on:click={toggleCart} aria-label="Close cart">✕</button>
				</header>
				{#if Object.keys(cartItems).length === 0}
					<p class="cart-empty">No items added yet.</p>
				{:else}
						<!-- Scrollable list of items -->
						<div class="cart-body">
							<ul class="cart-list">
								{#each Object.entries(cartItems) as [id, entry] (id)}
									<!-- Faded style applied when entry is zeroed and awaiting removal -->
									<li class="cart-entry" class:cart-entry-zeroed={entry.count === 0}>
										<div class="entry-info">
											<span class="cart-entry-label">{entry.label}</span>
											<span class="cart-entry-unit-price">{fmt(entry.price)} each</span>
										</div>
										<div class="entry-qty">
											<button class="qty-btn" on:click={() => removeFromCart(id)} disabled={entry.count === 0} aria-label="Remove one">−</button>
											<span class="cart-entry-count">{entry.count}</span>
											<button class="qty-btn" on:click={() => addOneById(id)} aria-label="Add one">+</button>
											<span class="cart-entry-line">{fmt(entry.price * entry.count)}</span>
										</div>
									</li>
								{/each}
							</ul>
						</div>
						<!-- Footer always pinned at the bottom of the panel -->
						<div class="cart-footer">
							<div class="cart-totals">
								<span class="cart-total-label">{cartCount} item{cartCount === 1 ? '' : 's'}</span>
								<span class="cart-total-price">{fmt(cartTotal)}</span>
							</div>
							<button class="checkout-btn" on:click={checkoutCart} disabled={cartCount === 0}>Go to Cart →</button>
						</div>
				{/if}
			</div>
		{/if}
	</div>
</main>

<style>
	@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;700&family=IBM+Plex+Mono:wght@400;500&display=swap');

	/* Remove default body margin and block page-level scroll;
	   all scrolling is handled inside .pane-stack / .pane-row / .item-grid. */
	:global(body) {
		margin: 0;
		overflow: hidden;
	}

	/* Full-viewport grid: hero text on top, pane-stack fills remaining space. */
	.canvas {
		--ink: #102533;
		--shadow: 0 16px 30px rgb(6 33 56 / 0.1);
		--pane-width: 22rem;
		--pane-height: 35rem;
		--card-row-height: 6rem;
		--grid-gap: 0.45rem;
		height: 100vh;
		display: grid;
		grid-template-rows: auto 1fr; /* hero = natural height, stack = rest */
		gap: 0.85rem;
		overflow: hidden;
		padding: clamp(1rem, 2vw + 0.8rem, 2.4rem);
		font-family: 'Space Grotesk', 'Trebuchet MS', sans-serif;
		color: var(--ink);
		/* Three-layer decorative background: two radial color spots + base gradient */
		background:
			radial-gradient(circle at 10% 8%, rgb(250 208 153 / 0.4), transparent 33%),
			radial-gradient(circle at 90% 90%, rgb(137 195 255 / 0.35), transparent 28%),
			linear-gradient(170deg, #efe8d8 0%, #f8fafc 45%, #e6eef8 100%);
	}

	/* Centered hero text block */
	.hero {
		max-width: 70ch;
		margin-inline: auto;
		text-align: center;
	}

	/* Small monospaced label above the main heading */
	.eyebrow {
		font-family: 'IBM Plex Mono', monospace;
		letter-spacing: 0.11em;
		font-size: 0.76rem;
		text-transform: uppercase;
		margin-bottom: 0.55rem;
	}

	/* Fluid heading size: scales between ~1.6rem and ~2.8rem with viewport width */
	h1 {
		font-size: clamp(1.6rem, 2.4vw + 1rem, 2.8rem);
		line-height: 1.1;
		margin: 0;
	}

	.hero p {
		margin-top: 0.8rem;
		line-height: 1.45;
	}

	/* Outer stack: rows stacked vertically, scrolls when rows overflow the viewport. */
	.pane-stack {
		display: grid;
		grid-auto-rows: max-content; /* each row shrinks to its natural height */
		align-content: start;
		gap: 1rem;
		min-height: 0;   /* allow the grid track to shrink below its content */
		overflow-y: auto;
		padding: 0.2rem 0.2rem 0.4rem;
		scrollbar-width: thin;
	}

	/* A horizontal band of panes; scrolls left-right with proximity snap. */
	.pane-row {
		display: flex;
		gap: 1rem;
		min-height: 0;
		overflow-x: auto;
		overflow-y: hidden;
		scroll-snap-type: x proximity;  /* snaps to nearest pane, not mandatory */
		padding: 0.2rem 0.2rem 0.9rem;  /* bottom padding exposes the scrollbar */
		scrollbar-width: thin;
		overscroll-behavior-x: contain; /* prevent horizontal swipe from going to the browser */
		overscroll-behavior-y: auto;
	}

	/* Individual pane card: fixed width/height, frosted-glass look. */
	.pane {
		display: grid;
		grid-template-rows: auto 1fr; /* header = natural, grid = rest of pane height */
		gap: 0.9rem;
		padding: 1rem;
		flex: 0 0 var(--pane-width); /* non-shrinking, non-growing fixed-width flex item */
		height: var(--pane-height);
		min-height: 0;
		border-radius: 1rem;
		background: linear-gradient(180deg, rgb(255 255 255 / 0.86), rgb(255 255 255 / 0.72));
		box-shadow: var(--shadow);
		border: 2px solid color-mix(in srgb, var(--accent) 55%, white); /* tinted border */
		backdrop-filter: blur(3px);
		scroll-snap-align: start;
	}

	/* Pane header: title left-aligned, stat right-aligned */
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

	/*
		Card grid: 4 explicit columns, 12 explicit rows.
		Height is locked to exactly 8 visible rows + 7 gaps so the 9th–12th rows
		require scrolling. Overflow-y:auto enables that inner scroll.
	*/
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

	/* Individual card: soft accent fill + tinted border using the pane's --accent color.
	   position:relative creates a containing block for the absolutely-placed add button. */
	.card {
		position: relative;
		padding: 0.45rem;
		padding-right: 1.7rem; /* reserve space so label text doesn't sit under the + button */
		border-radius: 0.85rem;
		background: color-mix(in srgb, var(--accent) 9%, white);
		border: 1px solid color-mix(in srgb, var(--accent) 26%, white);
	}

	/* Small circular + button anchored to the top-right of the card */
	.add-btn {
		position: absolute;
		top: 0.18rem;
		right: 0.28rem;
		width: 1.1rem;
		height: 1.1rem;
		border-radius: 50%;
		border: none;
		background: var(--accent);
		color: #fff;
		font-size: 0.95rem;
		line-height: 1;
		cursor: pointer;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0;
		opacity: 0.75;
		transition: opacity 0.15s, transform 0.15s;
	}

	.add-btn:hover {
		opacity: 1;
		transform: scale(1.15);
	}

	.add-btn:active {
		transform: scale(0.92);
	}

	/* Minus button anchored to the bottom-right of the card; only shown when count > 0 */
	.sub-btn {
		position: absolute;
		bottom: 0.18rem;
		right: 0.28rem;
		width: 1.1rem;
		height: 1.1rem;
		border-radius: 50%;
		border: none;
		background: #94a3b8;
		color: #fff;
		font-size: 0.95rem;
		line-height: 1;
		cursor: pointer;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0;
		opacity: 0.75;
		transition: opacity 0.15s, transform 0.15s;
	}

	.sub-btn:hover {
		opacity: 1;
		transform: scale(1.15);
	}

	.sub-btn:active {
		transform: scale(0.92);
	}

	/* Accent-colored count badge pinned to the top-left of the card;
	   only rendered when the card has been added at least once. */
	.card-count {
		position: absolute;
		top: -0.3rem;
		left: -0.3rem;
		background: var(--accent);
		color: #fff;
		border-radius: 999px;
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.55rem;
		font-weight: 700;
		min-width: 1rem;
		height: 1rem;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0 0.15rem;
		pointer-events: none;
	}

	.card-icon {
		display: block;
		font-size: 1.6rem;
		line-height: 1;
		margin-bottom: 0.2rem;
		pointer-events: none;
		user-select: none;
	}

	.card-label {
		margin: 0;
		font-weight: 700;
		font-size: 0.78rem;
	}

	/* Unit price shown below the label */
	.card-price {
		margin: 0.1rem 0 0;
		font-size: 0.7rem;
		font-family: 'IBM Plex Mono', monospace;
		font-weight: 700;
		color: var(--accent);
	}

	/* ── Cart widget ─────────────────────────────────────────────────────────── */

	/* Shared full-screen invisible backdrop; clicking closes all panels */
	.panel-backdrop {
		position: fixed;
		inset: 0;
		z-index: 199;
	}

	/* ── Context menu ──────────────────────────────────────────────────── */

	/* Fixed wrapper top-left, mirrors the cart-wrap on the right */
	.menu-wrap {
		position: fixed;
		top: 1rem;
		left: 1rem;
		z-index: 200;
	}

	/* Dark circular button matching the cart button style */
	.menu-btn {
		width: 2.8rem;
		height: 2.8rem;
		border-radius: 50%;
		border: none;
		background: #102533;
		color: #fff;
		display: flex;
		align-items: center;
		justify-content: center;
		cursor: pointer;
		box-shadow: 0 4px 14px rgb(0 0 0 / 0.22);
		transition: background 0.15s, transform 0.15s;
	}

	.menu-btn:hover {
		background: #1e3a4f;
		transform: scale(1.06);
	}

	/* Three-line hamburger icon drawn with box-shadow */
	.burger,
	.burger::before,
	.burger::after {
		display: block;
		width: 1.1rem;
		height: 2px;
		background: #fff;
		border-radius: 2px;
		position: relative;
		transition: transform 0.15s;
	}

	.burger::before,
	.burger::after {
		content: '';
		position: absolute;
		left: 0;
	}

	.burger::before { top: -0.34rem; }
	.burger::after  { top:  0.34rem; }

	/* Dropdown panel: top-left aligned */
	.menu-panel {
		position: absolute;
		top: calc(100% + 0.6rem);
		left: 0;
		width: 14rem;
		background: #fff;
		border-radius: 1rem;
		box-shadow: 0 8px 32px rgb(0 0 0 / 0.18);
		padding: 0.75rem 0.85rem;
		z-index: 210;
	}

	/* Vertical stack of nav links */
	.menu-nav {
		display: flex;
		flex-direction: column;
		gap: 0.1rem;
	}

	.menu-link {
		display: block;
		padding: 0.45rem 0.5rem;
		border-radius: 0.5rem;
		color: #102533;
		text-decoration: none;
		font-size: 0.88rem;
		transition: background 0.1s;
	}

	.menu-link:hover {
		background: #f1f5f9;
	}

	.menu-divider {
		border: none;
		border-top: 1px solid #e2e8f0;
		margin: 0.5rem 0;
	}

	/* Destructive-style action button (e.g. Clear cart) */
	.menu-action {
		display: block;
		width: 100%;
		padding: 0.45rem 0.5rem;
		border-radius: 0.5rem;
		border: none;
		background: none;
		color: #dc2626;
		font-family: 'Space Grotesk', 'Trebuchet MS', sans-serif;
		font-size: 0.88rem;
		text-align: left;
		cursor: pointer;
		transition: background 0.1s;
	}

	.menu-action:hover {
		background: #fef2f2;
	}

	/* Fixed wrapper provides the positioning context for both button and badge */
	.cart-wrap {
		position: fixed;
		top: 1rem;
		right: 1rem;
		z-index: 200;
	}

	/* Dark circular button housing the cart SVG icon */
	.cart-btn {
		width: 2.8rem;
		height: 2.8rem;
		border-radius: 50%;
		border: none;
		background: #102533;
		color: #fff;
		display: flex;
		align-items: center;
		justify-content: center;
		cursor: pointer;
		box-shadow: 0 4px 14px rgb(0 0 0 / 0.22);
		transition: background 0.15s, transform 0.15s;
	}

	.cart-btn:hover {
		background: #1e3a4f;
		transform: scale(1.06);
	}

	/* Dropdown panel: flex column so the footer never scrolls away */
	.cart-panel {
		position: absolute;
		top: calc(100% + 0.6rem);
		right: 0;
		width: 17rem;
		background: #fff;
		border-radius: 1rem;
		box-shadow: 0 8px 32px rgb(0 0 0 / 0.18);
		padding: 0.85rem 1rem;
		z-index: 210;
		max-height: 60vh;
		display: flex;
		flex-direction: column;
	}

	/* Scrollable area containing the item list; grows to fill available space */
	.cart-body {
		flex: 1;
		overflow-y: auto;
		min-height: 0;
		scrollbar-width: thin;
	}

	/* Pinned footer: total count + checkout button, never scrolls out of view */
	.cart-footer {
		flex-shrink: 0;
		border-top: 1px solid #e2e8f0;
		padding-top: 0.6rem;
		margin-top: 0.5rem;
	}

	/* Full-width checkout button */
	.checkout-btn {
		width: 100%;
		margin-top: 0.45rem;
		padding: 0.55rem;
		border-radius: 0.6rem;
		border: none;
		background: #102533;
		color: #fff;
		font-family: 'Space Grotesk', 'Trebuchet MS', sans-serif;
		font-size: 0.85rem;
		font-weight: 700;
		cursor: pointer;
		transition: background 0.15s;
	}

	.checkout-btn:hover {
		background: #1e3a4f;
	}

	/* Panel header: title left, close button right */
	.cart-panel-head {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.75rem;
	}

	.cart-panel-head h3 {
		margin: 0;
		font-size: 0.95rem;
	}

	/* Close (✕) icon button */
	.cart-close {
		background: none;
		border: none;
		cursor: pointer;
		font-size: 0.9rem;
		color: #666;
		padding: 0.1rem 0.3rem;
		border-radius: 0.3rem;
		line-height: 1;
	}

	.cart-close:hover {
		background: #f1f5f9;
		color: #102533;
	}

	.cart-empty {
		margin: 0;
		font-size: 0.82rem;
		color: #888;
		text-align: center;
		padding: 0.5rem 0;
	}

	/* Unstyled list of added items */
	.cart-list {
		list-style: none;
		margin: 0;
		padding: 0;
		display: flex;
		flex-direction: column;
		gap: 0.4rem;
	}

	/* One row per unique item */
	.cart-entry {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 0.4rem;
		font-size: 0.8rem;
		padding: 0.35rem 0.5rem;
		border-radius: 0.5rem;
		background: #f8fafc;
	}

	/* Left column: item name + unit price stacked */
	.entry-info {
		flex: 1;
		min-width: 0;
		display: flex;
		flex-direction: column;
		gap: 0.05rem;
	}

	.cart-entry-label {
		color: #102533;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.cart-entry-unit-price {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.68rem;
		color: #888;
	}

	/* Line total shown after the +/- count */
	.cart-entry-line {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.75rem;
		font-weight: 700;
		color: #0f766e;
		margin-left: 0.25rem;
		flex-shrink: 0;
	}

	/* Zeroed-out entry: faded and struck through, awaiting removal */
	.cart-entry-zeroed {
		opacity: 0.4;
	}

	.cart-entry-zeroed .cart-entry-label {
		text-decoration: line-through;
	}

	/* Wrapper for −, count, + controls */
	.entry-qty {
		display: flex;
		align-items: center;
		gap: 0.25rem;
		margin-left: 0.4rem;
		flex-shrink: 0;
	}

	/* Small square qty button used in the cart panel list */
	.qty-btn {
		width: 1.1rem;
		height: 1.1rem;
		border-radius: 0.25rem;
		border: 1px solid #cbd5e1;
		background: #fff;
		color: #102533;
		font-size: 0.85rem;
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

	/* Footer totals row: item count left, grand total right */
	.cart-totals {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.4rem;
	}

	.cart-total-label {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.78rem;
		color: #555;
	}

	.cart-total-price {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.9rem;
		font-weight: 700;
		color: #102533;
	}

	/* Red badge overlapping the top-right of the cart button */
	.cart-badge {
		position: absolute;
		top: -0.3rem;
		right: -0.3rem;
		background: #ef4444;
		color: #fff;
		border-radius: 999px;
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.62rem;
		font-weight: 700;
		min-width: 1.15rem;
		height: 1.15rem;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0 0.22rem;
		pointer-events: none; /* clicks should go through to the button */
	}

	/* Narrower paddings on small screens */
	@media (max-width: 760px) {
		.canvas {
			padding-inline: 0.8rem;
			padding-block: 0.7rem;
			grid-template-rows: auto 1fr;
		}
	}

	/* ── Detail drawer ──────────────────────────────────────────────────────── */

	/* Card body is interactive: show pointer and a subtle hover lift */
	.card {
		cursor: pointer;
		transition: box-shadow 0.15s, transform 0.15s;
	}

	.card:hover {
		transform: translateY(-1px);
		box-shadow: 0 4px 12px rgb(0 0 0 / 0.1);
	}

	/* Right-side drawer: slides in from the right edge */
	.detail-drawer {
		position: fixed;
		top: 0;
		right: 0;
		width: min(22rem, 92vw);
		height: 100vh;
		background: #fff;
		box-shadow: -8px 0 40px rgb(0 0 0 / 0.16);
		z-index: 205;
		display: flex;
		flex-direction: column;
		animation: drawer-slide-in 0.22s cubic-bezier(0.22, 1, 0.36, 1) both;
	}

	@keyframes drawer-slide-in {
		from { transform: translateX(100%); }
		to   { transform: translateX(0); }
	}

	/* Drawer header: category badge + title on the left, close button on the right */
	.detail-head {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		padding: 1.25rem 1.2rem 0.75rem;
		border-bottom: 1px solid #e2e8f0;
		flex-shrink: 0;
	}

	/* Pill badge showing the lane/category */
	.detail-category {
		display: inline-block;
		font-size: 0.7rem;
		font-family: 'IBM Plex Mono', monospace;
		font-weight: 700;
		letter-spacing: 0.06em;
		text-transform: uppercase;
		padding: 0.18rem 0.55rem;
		border-radius: 999px;
		margin-bottom: 0.4rem;
	}

	.detail-title {
		margin: 0;
		font-size: 1.3rem;
		line-height: 1.15;
	}

	/* Close button */
	.detail-close {
		background: none;
		border: none;
		cursor: pointer;
		font-size: 1rem;
		color: #666;
		padding: 0.2rem 0.4rem;
		border-radius: 0.3rem;
		line-height: 1;
		flex-shrink: 0;
		margin-left: 0.5rem;
	}

	.detail-close:hover {
		background: #f1f5f9;
		color: #102533;
	}

	/* Scrollable body */
	.detail-body {
		flex: 1;
		overflow-y: auto;
		padding: 1.1rem 1.2rem;
		display: flex;
		flex-direction: column;
		gap: 0.6rem;
		scrollbar-width: thin;
	}

	/* Large price display */
	.detail-price {
		margin: 0;
		font-family: 'IBM Plex Mono', monospace;
		font-size: 2rem;
		font-weight: 700;
		color: var(--drawer-accent);
		line-height: 1;
	}

	/* Small meta pill (row/col) */
	.detail-meta {
		margin: 0;
		font-family: 'IBM Plex Mono', monospace;
		font-size: 0.75rem;
		color: #888;
	}

	/* Description paragraph */
	.detail-desc {
		margin: 0;
		font-size: 0.92rem;
		line-height: 1.6;
		color: #334155;
	}

	/* Pinned footer: qty controls + Add to Cart button */
	.detail-footer {
		flex-shrink: 0;
		padding: 1rem 1.2rem;
		border-top: 1px solid #e2e8f0;
		display: flex;
		flex-direction: column;
		gap: 0.7rem;
	}

	/* +/− qty row inside the drawer */
	.detail-qty {
		display: flex;
		align-items: center;
		gap: 0.6rem;
	}

	.detail-qty-btn {
		width: 2rem;
		height: 2rem;
		border-radius: 50%;
		border: 1.5px solid #cbd5e1;
		background: #fff;
		color: #102533;
		font-size: 1.1rem;
		line-height: 1;
		display: flex;
		align-items: center;
		justify-content: center;
		cursor: pointer;
		transition: background 0.1s;
	}

	.detail-qty-btn:hover:not(:disabled) {
		background: #f1f5f9;
	}

	.detail-qty-btn:disabled {
		opacity: 0.35;
		cursor: default;
	}

	.detail-qty-count {
		font-family: 'IBM Plex Mono', monospace;
		font-size: 1.1rem;
		font-weight: 700;
		min-width: 1.5rem;
		text-align: center;
		color: #102533;
	}

	/* Full-width accent "Add to Cart — $X.XX" button */
	.detail-add-btn {
		width: 100%;
		padding: 0.75rem;
		border-radius: 0.75rem;
		border: none;
		color: #fff;
		font-family: 'Space Grotesk', 'Trebuchet MS', sans-serif;
		font-size: 0.95rem;
		font-weight: 700;
		cursor: pointer;
		transition: filter 0.15s;
	}

	.detail-add-btn:hover {
		filter: brightness(1.12);
	}
</style>
