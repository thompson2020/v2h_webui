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
		state?: string;
		temp?: number;
		fan?: number;
		amps?: number;
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

	//Boost variables
	let amps_value = 16; // hardware max
	let soc_range_value = 90; // default
	let value = '';
	let wsInterval = 0;

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
<div class=" container mx-auto px-4 flex flex-wrap gap-2 md:grid-cols-3">
	<div class="w-80 grow md:grow-0 card p-10">
		<h2>Operational Mode: {value}</h2>
		<div class="h-4" />

		<!-- Off Button - Separate box at the top -->
		<div class="border border-surface-300 dark:border-surface-600 rounded-container-token p-4 bg-surface-100 dark:bg-surface-800 flex-col items-center">
			<div class="mt-1 text-sm font-semibold text-surface-500 dark:text-surface-400 mb-3">
				EV Power
			</div>

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

		<!--Smart Modes-->
		<div
			class="border border-surface-300 dark:border-surface-600 rounded-container-token p-4 bg-surface-100 dark:bg-surface-800 flex-col items-center"
		>
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

			<div class="mt-3">
				<div class="flex justify-between items-center mb-2">
					<div class="text-sm font-bold">SOC Range</div>
					<div class="text-xs">
						{soc_range_v2h[0]}% → {soc_range_v2h[1]}%
					</div>
				</div>

				<RangeSlider bind:values={soc_range_v2h} min={0} max={100} step={1} range />
			</div>

			<!-- Options -->
			<div class="mt-4 flex flex-col gap-3">

				<!-- Self-Use -->
				<div class="flex justify-between items-center text-sm">
					<span title="Discharge battery to off-set house use "> Self-Use </span>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={selfUse} class="sr-only peer" />
						<div
							class="
							w-9 h-5 bg-surface-300 rounded-full peer
							peer-checked:bg-primary-500
							after:content-['']
							after:absolute after:top-[2px] after:left-[2px]
							after:bg-white after:rounded-full
							after:h-4 after:w-4
							after:transition-all
							peer-checked:after:translate-x-4
						"
						/>
					</label>
				</div>

				<!-- Export Excess Solar -->
				<div class="flex justify-between items-center text-sm">
					<span title="Allow excess solar to be exported "> Export Excess Solar </span>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={exportExcessSolar} class="sr-only peer" />
						<div
							class="
							w-9 h-5 bg-surface-300 rounded-full peer
							peer-checked:bg-primary-500
							after:content-['']
							after:absolute after:top-[2px] after:left-[2px]
							after:bg-white after:rounded-full
							after:h-4 after:w-4
							after:transition-all
							peer-checked:after:translate-x-4
						"
						/>
					</label>
				</div>

				<!-- EV Drain Protection -->
				<div class="flex justify-between items-center text-sm">
					<span title="Don’t empty battery into EV charger"> EV Drain Protection </span>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={evDrainProtection} class="sr-only peer" />
						<div
							class="
							w-9 h-5 bg-surface-300 rounded-full peer
							peer-checked:bg-primary-500
							after:content-['']
							after:absolute after:top-[2px] after:left-[2px]
							after:bg-white after:rounded-full
							after:h-4 after:w-4
							after:transition-all
							peer-checked:after:translate-x-4
						"
						/>
					</label>
				</div>

				<!-- Time/Event-Based Controls -->
				<div class="mt-3 mb-0 text-xs font-semibold text-surface-500 dark:text-surface-400 mb-3">
					Time / Event Based Controls
				</div>

				<!-- Ready to Drive -->
				<div class="flex justify-between items-center text-sm">
					<span title="Specify when to vehicle to be charged by"> Ready to Drive </span>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={readyToDrive} class="sr-only peer" />
						<div
							class="
							w-9 h-5 bg-surface-300 rounded-full peer
							peer-checked:bg-primary-500
							after:content-['']
							after:absolute after:top-[2px] after:left-[2px]
							after:bg-white after:rounded-full
							after:h-4 after:w-4
							after:transition-all
							peer-checked:after:translate-x-4
						"
						/>
					</label>
				</div>

				<!-- Off-Peak Charging -->
				<div class="flex justify-between items-center text-sm">
					<span title="Charge when cheap rate is available"> Off-Peak Charging </span>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={OffPeakCharging} class="sr-only peer" />
						<div
							class="
							w-9 h-5 bg-surface-300 rounded-full peer
							peer-checked:bg-primary-500
							after:content-['']
							after:absolute after:top-[2px] after:left-[2px]
							after:bg-white after:rounded-full
							after:h-4 after:w-4
							after:transition-all
							peer-checked:after:translate-x-4
						"
						/>
					</label>
				</div>

				<!-- Smart Charge -->
				<div class="flex justify-between items-center text-sm">
					<span title="Charge when additional cheap slots are available"> Smart Charge </span>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={smartCharge} class="sr-only peer" />
						<div
							class="
							w-9 h-5 bg-surface-300 rounded-full peer
							peer-checked:bg-primary-500
							after:content-['']
							after:absolute after:top-[2px] after:left-[2px]
							after:bg-white after:rounded-full
							after:h-4 after:w-4
							after:transition-all
							peer-checked:after:translate-x-4
						"
						/>
					</label>
				</div>

				<!-- Smart Export -->
				<div class="flex justify-between items-center text-sm">
					<span title="Export when price is high"> Smart Export </span>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={smartExport} class="sr-only peer" />
						<div
							class="
							w-9 h-5 bg-surface-300 rounded-full peer
							peer-checked:bg-primary-500
							after:content-['']
							after:absolute after:top-[2px] after:left-[2px]
							after:bg-white after:rounded-full
							after:h-4 after:w-4
							after:transition-all
							peer-checked:after:translate-x-4
						"
						/>
					</label>
				</div>







			</div>
		</div>

		<div class="h-4" />

		<!-- Boost Section -->
		        <!-- Boost Section -->
        <div class="border border-surface-300 dark:border-surface-600 rounded-container-token p-4 bg-surface-100 dark:bg-surface-800">

            <div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-3">
                Charge
            </div>

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
				


            <!-- Keep your sliders and checkbox as they are -->
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

		<br />
	</div>
	<!-- </div> -->

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

	<div class="flex-auto card p-10 max-h-[60vh] overflow-y-auto space-y-4">
		<h2>Log</h2>
		<div class="table table-hover">
			<table id="dataTable" class="table-container">
				<thead>
					<tr>
						<th class="table-cell-fit">Localtime</th>
						<th>SoC</th>
						<th>State</th>
						<th>Temperature</th>
						<th>Fan duty</th>
						<th>Amps</th>
					</tr>
				</thead>
				<tbody>
					{#each $realTimeData as rtd}
						<tr>
							<td>{rtd.time}</td>
							<td>{rtd.soc}%</td>
							{#if typeof rtd.state === 'object'}
								<td>{Object.keys(rtd.state)[0]}</td>
							{:else}
								<td>{rtd.state}</td>
							{/if}
							<td>{rtd.temp}ºC</td>
							<td>{rtd.fan}%</td>
							<td>{rtd.amps}A</td>
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
