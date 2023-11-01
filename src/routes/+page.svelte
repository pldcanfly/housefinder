<script lang="ts">
	import { state, type district, type openplot, fetchState } from '$lib/api';
	let selectedSize = -1;
	let selectedRestriction = 'all';
	let selectedDistrict = 0;
	let selectedServer = 0;

	let baseplots: Array<openplot> = [];
	let plotmap: Map<number, Array<openplot>> = new Map();

	let discrictmap = new Map<number, district>();
	state.subscribe((state) => {
		for (const discrict of state?.districts || []) {
			if (selectedDistrict == 0) selectedDistrict = discrict.id;
			discrictmap.set(discrict.id, discrict);
		}
		discrictmap = discrictmap;
		baseplots = discrictmap.get(selectedDistrict)?.open_plots || [];
	});

	$: {
		plotmap = new Map();
		baseplots = (discrictmap.get(selectedDistrict)?.open_plots || []).filter((item) => {
			if (selectedRestriction == 'nofc' && item.purchase_system == 3) return false;
			if (selectedSize != -1 && item.size != selectedSize) return false;

			return true;
		});

		for (const plot of baseplots) {
			const plotnumber = plot.plot_number > 30 ? plot.plot_number - 30 : plot.plot_number;
			const plots = plotmap.get(plotnumber)?.values() || [];
			plotmap.set(plotnumber, [...plots, plot]);
		}
	}

	const restrictionmap = new Map<number, string>([
		[5, 'Indivudual'],
		[3, 'Free Company'],
		[7, 'Unrestricted']
	]);

	const sizemap = new Map<number, string>([
		[0, 'Small'],
		[1, 'Medium'],
		[2, 'Large']
	]);

	const servermap = new Map<number, string>([
		[67, 'Shiva'],
		[403, 'Raiden'],
		[402, 'Alpha'],
		[33, 'Twintania'],
		[42, 'Zordiak'],
		[42, 'Zordiak']
	]);

	const selectServer = (id: number) => {
		selectedServer = id;
		fetchState(selectedServer);
	};

	let timeout = 0;
	const refresh = () => {
		if (timeout <= 0) {
			plotmap = new Map();
			fetchState(selectedServer);
			timeout = 60;
			const int = setInterval(() => {
				timeout--;
				if (timeout <= 0) {
					clearInterval(int);
				}
			}, 1000);
		}
	};
</script>

<svelte:head>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
	<link href="https://fonts.googleapis.com/css2?family=Roboto&display=swap" rel="stylesheet" />
</svelte:head>

{#if selectedServer == 0}
	<div class="mapcontainer">
		<div class="servermap">
			{#each servermap.entries() as [id, servername]}
				<button on:click={() => selectServer(id)}>{servername}</button>
			{/each}
		</div>
	</div>
{:else}
	<div class="filters">
		<div>City</div>
		<div>Restriction</div>
		<div>Size</div>
		<div />

		<select bind:value={selectedDistrict}>
			{#each discrictmap as discrict}
				<option value={discrict[0]}>{discrict[1].name}</option>
			{/each}
		</select>

		<select bind:value={selectedRestriction}>
			<option value="all">All</option>
			<option value="nofc">no FC-only</option>
		</select>

		<select bind:value={selectedSize}>
			<option value={-1}>All</option>
			<option value={0}>{sizemap.get(0)}</option>
			<option value={1}>{sizemap.get(1)}</option>
			<option value={2}>{sizemap.get(2)}</option>
		</select>
		<div>
			<button style="width:100%" on:click={refresh} class:inactive={timeout > 0}
				>Aktualisieren{#if timeout > 0} ({timeout}s){/if}</button
			>
		</div>
	</div>

	<div class="housegrid">
		<div class="head">
			<div>Plot</div>
			<div class="sub">
				<div>Ward <span class="small">(Plot)</span></div>
				<div>Entries</div>
				<div>Size</div>
				<div>Restriction</div>
				<div>Phase</div>
			</div>
		</div>

		{#each plotmap.entries() as [id, openplot]}
			<div class="body">
				<div>{id + 1} / {id + 31}</div>
				<div class="sub">
					{#each openplot as subplot}
						<div>
							{subplot.ward_number + 1} <span class="small">({subplot.plot_number + 1})</span>
						</div>
						<div>{subplot.lotto_entries == null ? '?' : subplot.lotto_entries}</div>
						<div>{sizemap.get(subplot.size)}</div>
						<div>{restrictionmap.get(subplot.purchase_system)}</div>
						<div>{subplot.lotto_phase}</div>
					{/each}
				</div>
			</div>
		{:else}
			<div>Loading...</div>
		{/each}
	</div>
{/if}

<style lang="scss">
	$bg: #191919;
	$primary: #6c6c6c;

	$even: #454545;
	$odd: #222222;

	:global(body) {
		background-color: $bg;
		color: #8e8e8e;
		font-family: 'Roboto', sans-serif;
	}

	button,
	select {
		padding: 25px;
		background-color: $primary;
		color: #fff;
		border: none;

		&:hover {
			cursor: pointer;
			background-color: #939393;
		}
	}

	button.inactive {
		cursor: no-drop;
		opacity: 0.5;
	}
	.mapcontainer {
		display: grid;
		height: 100vh;
		width: 100vw;

		place-content: center;
		.servermap {
			display: grid;
			grid-template-columns: 1fr 1fr;
			gap: 15px;
		}
	}

	.filters {
		display: grid;
		gap: 5px;
		grid-template-columns: 1fr 1fr 1fr 200px;
	}

	.housegrid {
		.small {
			font-size: 70%;
		}
		margin-top: 50px;
		.body,
		.head {
			padding: 10px;
			display: grid;
			grid-template-columns: 100px 1fr;

			.sub {
				display: grid;
				grid-template-columns: 1fr 1fr 1fr 1fr 1fr;
			}
		}

		.head {
			font-weight: bold;
		}

		.body:nth-child(even) {
			background: $even;
		}
		.body:nth-child(odd) {
			background: $odd;
		}
	}
</style>
