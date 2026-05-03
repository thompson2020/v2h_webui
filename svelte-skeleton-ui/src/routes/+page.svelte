<!-- 

	Todo:

		Change header/nav colour based on mode - https://www.skeleton.dev/docs/themes
		Make WS client a singleton or globally available? (WIP)
		Disable UI on WS disconnect (see static pages)
		Implement Event Table actions (Edit, Delete, Add Event & Update)
		Retreive current mode from State (table) and update Mode Selection
		Display WS bad acks {"ack": "err"}

 -->
<script lang="ts">
	import { RangeSlider } from '@skeletonlabs/skeleton';
	import { ListBoxItem, ListBox } from '@skeletonlabs/skeleton';
	import { tableMapperValues } from '@skeletonlabs/skeleton';
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
	import { operationalMode } from '$lib/stores';

	interface RangeSliderProps {
		value: number;
		min: number;
		max: number;
		step: number;
		ticked: boolean;
	}

	interface EventData {
		time: string;
		action: string;
	}

	interface ChargeOptions {
		amps?: number;
		eco?: boolean;
		soc_limit?: number;
	}

	interface EventData {
		time: string;
		action: string;
	}

	interface RealTimeData {
		time?: string;
		soc?: number;
		state?: string | object;
		requested_amps?: number;
		dc_kw?: number;
		volts?: number;
		temp?: number;
		amps?: number;
		fan?: number;
		meter_kw?: number;
		phase_w?: number | null;
		smart_charge?: boolean;
		ev_drain_protection?: boolean;
	}

	let socket: WebSocket;
	let time = '';

	//Auto/Load Balancing + Time Based + Event Based(Charge)e.g Additional Slots + Event Based Discharge (High export price)
	let soc_range_v2h = [31, 90];
	let selfUse = true;
	let exportExcessSolar = false;
	let evDrainProtection = false;

	let readyToDrive = false;
	let OffPeakCharging = true;
	let smartCharge = true;
	let smartExport = false;
	let smartExportLimit = 2500;
	let showSmartExportOptions = false;
	let showReadyToDriveOptions = false;
	let readyToDriveTime = '08:00';
	let readyToDriveDays = [false, false, false, false, false, false, false]; // M T W T F S S
	let showOffPeakOptions = false;
	let showSmartChargeOptions = false;
	let offPeakStart = '00:30';
	let offPeakEnd = '04:30';
	let v2hMaxAmps = 16;

	//Boost variables
	let amps_value = 16; // hardware max
	let soc_range_value = 90; // default
	let value = '';
	let wsInterval = 0;
	$: operationalMode.set(value);

	let eventData = writable<EventData[]>([]);
	let realTimeData = writable<RealTimeData[]>([]);
	let sourceData = [
		{ time: '00:01:59', action: 'Idle' },
		{ time: '00:01:59', action: 'Idle' }
	]; // Placeholder data whilst testing
	const tableSimple = {
		head: ['Time', 'Action'],
		body: tableMapperValues(sourceData, ['time', 'action']),
		meta: tableMapperValues(sourceData, ['name', 'action'])
	};

	function submitBoostCharge(event: Event) {
		event.preventDefault();
		const ampsValue = Number(
			(document.getElementById('range-slider-amps') as HTMLInputElement)?.value
		);
		const socRangeValue = Number(
			(document.getElementById('range-slider-boost-soc') as HTMLInputElement)?.value
		);

		const chargePayload = {
			cmd: {
				SetMode: {
					Charge: {
						// maybe use ChargeOptions interface here?
						amps: ampsValue, // can be null
						eco: false ,  // Boost always sends eco = false
						soc_limit: socRangeValue
					}
				}
			}
		};

		console.log('Testing charge payload' + JSON.stringify(chargePayload));
		// Send the payload as a JSON string
		// {"cmd":{"SetMode":{"Charge":{"amps":90,"eco":false,"soc_limit":16}}}}
		socket.send(JSON.stringify(chargePayload));
	}

	function submitChargeOnSolar(event: Event) {
		event.preventDefault();
		const ampsValue = Number(
			(document.getElementById('range-slider-amps') as HTMLInputElement)?.value
		);
		const socRangeValue = Number(
			(document.getElementById('range-slider-boost-soc') as HTMLInputElement)?.value
		);

		const chargePayload = {
			cmd: {
				SetMode: {
					Charge: {
						amps: ampsValue,
						eco: true, // Always true for Charge on Solar
						soc_limit: socRangeValue
					}
				}
			}
		};

		socket.send(JSON.stringify(chargePayload));
	}

	function radioModeChange(event: Event) {
		let mode = value; // global fudge
		let chargePayload = null;
		if (mode === 'Discharge') {
			const discharge_params: ChargeOptions = { soc_limit: soc_range_value, amps: amps_value };
			chargePayload = { cmd: { SetMode: { Discharge: discharge_params } } };
		} else {
			chargePayload = {
				cmd: {
					SetMode: mode
				}
			};
		}
		console.log(JSON.stringify(event));
		console.log('Testing mode payload' + JSON.stringify(chargePayload));
		// Send the payload as a JSON string
		// {"cmd": {"SetMode": "V2h"}}
		// {"cmd": {"SetMode": "Idle"}}
		// {"cmd": {"SetMode": {"Discharge" : {amps: 10, soc_limit: 30, eco: null}}
		socket.send(JSON.stringify(chargePayload));
	}
	if (typeof WebSocket !== 'undefined') {
		// Make this a singleton?
		socket = new WebSocket('ws://192.168.10.101:5555');
		socket.addEventListener('open', () => {
			console.log('Opened');
			const message = JSON.stringify({ cmd: 'GetData' });

			// Send the message
			socket.send(message);
		});

		socket.addEventListener('message', (event: MessageEvent) => {
			const message = JSON.parse(event.data);
			console.log(JSON.stringify(message));
			if (message.Events) {
				eventData.set(message.Events);
			}
			if (message.Data) {
				message.Data.time = new Date().toLocaleTimeString();
				realTimeData.update((items) => {
					items.unshift(message.Data); // push into [0]
					if (items.length > 80) {
						items.pop();
					}
					return items;
				});
			}
			if (message.Mode) {
				value = message.Mode; // Update displayed mode from charger
				console.log('Mode received from charger:', value);
			}
		});
	}

	onMount(() => {
		console.log('on mount');
		if (socket) {
			async function fetchData() {
				try {
					let message = JSON.stringify({ cmd: 'GetData' });
					console.log('periodic: ' + message);
					// Send the message
					socket.send(message);

					message = JSON.stringify({ cmd: 'GetEvents' });
					console.log('periodic: ' + message);
					// Send the message
					socket.send(message);
				} catch (error) {
					console.error('WebSocket send error:', error);
				}
			}

			const interval = setInterval(fetchData, 3000);
			fetchData(); // Fetch data immediately when the component mounts

			return () => {
				console.log('onMount returned');
				clearInterval(interval);
			};
		}
	});
</script>

<br />
<div class="container mx-auto px-4 flex flex-wrap gap-2">

	<!-- EV Power Card -->
	<div class="card p-4">
		<div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-3">EV Power</div>
		<button
			class="btn variant-filled {value === 'Idle' ? 'variant-filled-primary' : ''}"
			on:click={() => {
				value = 'Idle';
				radioModeChange(new Event('click'));
			}}
		>
			Off
		</button>
	</div>

	<!--Smart Self-Powered Card-->
	<div class="card p-4">
		<div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-3">
			Smart Self-Powered
		</div>

		<button
			class="btn variant-filled {value === 'V2h' ? 'variant-filled-primary' : ''}"
			on:click={() => {
				value = 'V2h';
				radioModeChange(new Event('click'));
			}}
		>
			On
		</button>

		<!-- SOC Range: two sliders for min and max -->
		<div class="mt-3 flex flex-col gap-1">
			<RangeSlider
				name="soc-min"
				bind:value={soc_range_v2h[0]}
				min={0}
				max={soc_range_v2h[1]}
				step={5}
				ticked
			>
				<div class="flex justify-between items-center">
					<div class="text-xs">Min SoC</div>
					<div class="text-xs">{soc_range_v2h[0]}%</div>
				</div>
			</RangeSlider>
			<RangeSlider
				name="soc-max"
				bind:value={soc_range_v2h[1]}
				min={soc_range_v2h[0]}
				max={100}
				step={5}
				ticked
			>
				<div class="flex justify-between items-center">
					<div class="text-xs">Max SoC</div>
					<div class="text-xs">{soc_range_v2h[1]}%</div>
				</div>
			</RangeSlider>
		</div>

		<!-- Max Amps -->
		<div class="mt-2">
			<RangeSlider name="v2h-amps" bind:value={v2hMaxAmps} min={1} max={16} step={1} ticked>
				<div class="flex justify-between items-center">
					<div class="text-xs">Max Amps</div>
					<div class="text-xs">{v2hMaxAmps}A</div>
				</div>
			</RangeSlider>
		</div>

		<!-- Options -->
		<div class="mt-4 flex flex-col gap-3">

			<div class="flex justify-between items-center text-sm">
				<span title="Discharge battery to offset house use"> Self-Use </span>
				<label class="relative inline-flex items-center cursor-pointer">
					<input type="checkbox" bind:checked={selfUse} class="sr-only peer" />
					<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
				</label>
			</div>

			<div class="flex justify-between items-center text-sm">
				<span title="Allow excess solar to be exported"> Export Excess Solar </span>
				<label class="relative inline-flex items-center cursor-pointer">
					<input type="checkbox" bind:checked={exportExcessSolar} class="sr-only peer" />
					<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
				</label>
			</div>

			<div class="flex justify-between items-center text-sm">
				<span title="Don't empty battery into EV charger"> EV Drain Protection </span>
				<label class="relative inline-flex items-center cursor-pointer">
					<input type="checkbox" bind:checked={evDrainProtection} class="sr-only peer" />
					<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
				</label>
			</div>

			<!-- Time/Event-Based Controls -->
			<div class="mt-1 text-xs font-semibold text-surface-500 dark:text-surface-400">
				Time / Event Based Controls
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors"
						on:click={() => (showReadyToDriveOptions = !showReadyToDriveOptions)}>
						<span class="text-xs w-3 text-center">{showReadyToDriveOptions ? '▼' : '▶'}</span>
						<span>Ready to Drive</span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={readyToDrive} class="sr-only peer" />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showReadyToDriveOptions}
					<div class="mt-2 ml-4 flex flex-col gap-2 text-surface-600 dark:text-surface-300">
						<div class="flex justify-between items-center">
							<span>Ready by</span>
							<input type="time" bind:value={readyToDriveTime} class="input w-28 text-sm py-0.5 px-1" />
						</div>
						<div class="flex items-center gap-2">
							{#each ['M','T','W','T','F','S','S'] as day, i}
								<label class="flex flex-col items-center cursor-pointer">
									<span class="text-xs mb-0.5">{day}</span>
									<input type="checkbox" bind:checked={readyToDriveDays[i]} class="w-4 h-4 accent-primary-500" />
								</label>
							{/each}
						</div>
					</div>
				{/if}
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors"
						on:click={() => (showOffPeakOptions = !showOffPeakOptions)}>
						<span class="text-xs w-3 text-center">{showOffPeakOptions ? '▼' : '▶'}</span>
						<span>Off-Peak Charging</span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={OffPeakCharging} class="sr-only peer" />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showOffPeakOptions}
					<div class="mt-2 ml-4 flex flex-col gap-2 text-surface-600 dark:text-surface-300">
						<div class="flex justify-between items-center">
							<span>Start</span>
							<input type="time" bind:value={offPeakStart} class="input w-28 text-sm py-0.5 px-1" />
						</div>
						<div class="flex justify-between items-center">
							<span>End</span>
							<input type="time" bind:value={offPeakEnd} class="input w-28 text-sm py-0.5 px-1" />
						</div>
					</div>
				{/if}
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors"
						on:click={() => (showSmartChargeOptions = !showSmartChargeOptions)}>
						<span class="text-xs w-3 text-center">{showSmartChargeOptions ? '▼' : '▶'}</span>
						<span>Smart Charge</span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={smartCharge} class="sr-only peer" />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showSmartChargeOptions}
					<div class="mt-1 ml-4 text-xs text-surface-400">
						uses Charge SoC/Amps
					</div>
				{/if}
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors"
						on:click={() => (showSmartExportOptions = !showSmartExportOptions)}>
						<span class="text-xs w-3 text-center">{showSmartExportOptions ? '▼' : '▶'}</span>
						<span>Smart Export</span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={smartExport} class="sr-only peer" />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showSmartExportOptions}
					<div class="mt-2 ml-4 flex justify-between items-center text-surface-600 dark:text-surface-300">
						<span>Export Limit</span>
						<div class="flex items-center gap-1">
							<input type="number" bind:value={smartExportLimit}
								min="0" max="10000" step="100"
								class="input w-20 text-right text-sm py-0.5 px-1" />
							<span class="text-xs">W</span>
						</div>
					</div>
				{/if}
			</div>

		</div>
	</div>

	<!-- Charge Card -->
	<div class="card p-4">
		<div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-3">Charge</div>

		<button
			class="btn variant-filled flex justify-between items-center mb-4"
			on:click={submitBoostCharge}>
			Boost
		</button>

		<button
			class="btn variant-filled flex justify-between items-center mb-4"
			on:click={submitChargeOnSolar}>
			Charge On Solar
		</button>

		<RangeSlider
			name="soc"
			id="range-slider-boost-soc"
			bind:value={soc_range_value}
			min={10}
			max={100}
			step={5}
			ticked
		>
			<div class="flex justify-between items-center">
				<div class="font-bold">SoC</div>
				<div class="text-xs">{soc_range_value} / 100</div>
			</div>
		</RangeSlider>

		<RangeSlider
			name="amps"
			id="range-slider-amps"
			bind:value={amps_value}
			max={16}
			step={1}
			ticked
		>
			<div class="flex justify-between items-center">
				<div class="font-bold">Amps</div>
				<div class="text-xs">{amps_value} / 16</div>
			</div>
		</RangeSlider>
	</div>

	<!-- Operational Mode Card -->
	<div class="card p-4">
		<div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-3">Operational Mode</div>
		<div class="text-2xl font-bold">{value || '—'}</div>
	</div>

	<div class="flex-auto card p-10 max-h-[60vh] overflow-y-auto space-y-4">
		<h2>Event Table</h2>
		<table id="eventsTable" class="table table-hover">
			<thead>
				<tr>
					<th class="">Time</th>
					<th class="">Action</th>
					<th class="table-cell-fit">Edit</th>
					<th class="table-cell-fit">Delete</th>
				</tr>
			</thead>
			<tbody>
				{#each $eventData as event}
					<tr>
						<td>{event.time}</td>
						<td>{event.action}</td>
						<td><button type="button" class="btn btn-sm">📝</button></td>
						<td><button type="button" class="btn btn-sm">❌</button></td>
					</tr>
				{/each}
			</tbody>
		</table>
		<div class="grid container-fluid">
			<button id="addRowButton">Add Event</button>
			<button id="updateButton">Update</button>
		</div>
	</div>

</div>

<div class="container mx-auto px-4 mt-2">
	<div class="card p-4" style="max-height: 60vh; overflow-y: auto;">
		<div style="overflow-x: auto;">
			<table id="dataTable" style="white-space: nowrap; border-collapse: collapse; width: 100%;">
				<thead>
					<tr style="border-bottom: 2px solid var(--color-surface-400);">
						<th class="p-2 text-left" rowspan="2">Time</th>
						<th class="p-2 text-left text-primary-500" colspan="3" style="border-left: 1px solid var(--color-surface-400);">CHAdeMO</th>
						<th class="p-2 text-left text-secondary-500" colspan="5" style="border-left: 1px solid var(--color-surface-400);">Pre</th>
						<th class="p-2 text-left text-tertiary-500" colspan="2" style="border-left: 1px solid var(--color-surface-400);">Meter</th>
						<th class="p-2 text-left text-success-500" colspan="2" style="border-left: 1px solid var(--color-surface-400);">Supervisory</th>
					</tr>
					<tr style="border-bottom: 1px solid var(--color-surface-300);">
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">SoC</th>
						<th class="p-2 text-left text-xs">State</th>
						<th class="p-2 text-left text-xs">Req A</th>
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">DC kW</th>
						<th class="p-2 text-left text-xs">Volts</th>
						<th class="p-2 text-left text-xs">Temp</th>
						<th class="p-2 text-left text-xs">Amps</th>
						<th class="p-2 text-left text-xs">Fan</th>
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">Total W</th>
						<th class="p-2 text-left text-xs">Phase W</th>
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">Smart</th>
						<th class="p-2 text-left text-xs">Drain Prot</th>
					</tr>
				</thead>
				<tbody>
					{#each $realTimeData as rtd}
						<tr>
							<td class="p-2">{rtd.time}</td>
							<td class="p-2">{Math.round(rtd.soc ?? 0)}%</td>
							<td class="p-2">{typeof rtd.state === 'object' && rtd.state !== null ? Object.keys(rtd.state)[0] : rtd.state}</td>
							<td class="p-2">{(rtd.requested_amps ?? 0).toFixed(1)}A</td>
							<td class="p-2">{(rtd.dc_kw ?? 0).toFixed(2)}</td>
							<td class="p-2">{(rtd.volts ?? 0).toFixed(1)}V</td>
							<td class="p-2">{(rtd.temp ?? 0).toFixed(1)}ºC</td>
							<td class="p-2">{(rtd.amps ?? 0).toFixed(1)}A</td>
							<td class="p-2">{Math.round(rtd.fan ?? 0)}%</td>
							<td class="p-2">{Math.round(rtd.meter_kw ?? 0)}</td>
							<td class="p-2">{rtd.phase_w != null ? Math.round(rtd.phase_w) + 'W' : '—'}</td>
							<td class="p-2">{rtd.smart_charge ? '✓' : '—'}</td>
							<td class="p-2">{rtd.ev_drain_protection ? '✓' : '—'}</td>
						</tr>
					{/each}
				</tbody>
			</table>
		</div>
	</div>
</div>

<!-- </div> -->

<!-- </div> -->

<style lang="postcss">
	figure {
		@apply flex relative flex-col;
	}
	figure svg,
	.img-bg {
		@apply w-64 h-64 md:w-80 md:h-80;
	}
	.img-bg {
		@apply absolute z-[-1] rounded-full blur-[50px] transition-all;
		animation: pulse 5s cubic-bezier(0, 0, 0, 0.5) infinite, glow 5s linear infinite;
	}
	@keyframes glow {
		0% {
			@apply bg-primary-400/50;
		}
		33% {
			@apply bg-secondary-400/50;
		}
		66% {
			@apply bg-tertiary-400/50;
		}
		100% {
			@apply bg-primary-400/50;
		}
	}
	@keyframes pulse {
		50% {
			transform: scale(1.5);
		}
	}
</style>
