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

	interface OperatorSettings {
		v2h_soc_min:          number;
		v2h_soc_max:          number;
		v2h_max_amps:         number;
		self_use:             boolean;
		export_excess_solar:  boolean;
		ev_drain_protection:  boolean;
		ready_to_drive:            boolean;
		ready_to_drive_end_time:   string;
		ready_to_drive_start_time: string;
		ready_to_drive_soc:        number;
		ready_to_drive_days:       boolean[];
		off_peak_charging:    boolean;
		off_peak_start:       string;
		off_peak_end:         string;
		smart_charge:         boolean;
		smart_export:                boolean;
		smart_export_limit_w:        number;
		smart_export_excess_solar:   boolean;
		charge_soc_limit:            number;
		charge_amps:          number;
		charge_eco:           boolean;
	}

	interface RealTimeData {
		time?: string;
		soc?: number;
		state?: string | object;
		requested_amps?: number;
		// PRE charger fields
		pre_state?: string;
		dc_volts_setpoint?: number;
		dc_amps_setpoint?: number;
		dc_output_volts?: number;
		dc_output_amps?: number;
		dc_w?: number;
		dc_bus_volts?: number;
		ac_volts?: number;
		ac_amps?: number;
		ac_w?: number;
		pre_temp?: number;
		pre_fan?: number;
		pre_enabled?: boolean;
		pre_status_ok?: boolean;
		pre_status?: [number, number];
		// Meter
		meter_kw?: number;
		phase_w?: number | null;
		// Supervisory
		smart_charge_request?: boolean;
		smart_charge_active?: boolean;
		ev_drain_protection_request?: boolean;
		ev_drain_protection_active?: boolean;
		smart_export_request?: boolean;
		smart_export_active?: boolean;
		smart_export_excess_solar_request?: boolean;
		smart_export_excess_solar_active?: boolean;
		ready_to_drive_active?: boolean;
		off_peak_charging_active?: boolean;
		settings?: OperatorSettings;
	}

	let socket: WebSocket;
	let time = '';

	//Auto/Load Balancing + Time Based + Event Based(Charge)e.g Additional Slots + Event Based Discharge (High export price)
	let v2h_soc_min = 31;
	let v2h_soc_max = 90;
	let selfUse = true;
	let exportExcessSolar = false;
	let evDrainProtection = false;

	let readyToDrive = false;
	let readyToDriveSoc = 90;
	let readyToDriveEndTime = '08:00';
	let readyToDriveStartTime = '--:--';
	let OffPeakCharging = true;
	let smartCharge = true;
	let smartExport = false;
	let smartExportLimit = 2500;
	let showSmartExportOptions = false;
	let smartExportExcessSolar = false;
	let showSmartExportExcessSolarOptions = false;
	let showReadyToDriveOptions = false;
	let readyToDriveDays = [false, false, false, false, false, false, false]; // M T W T F S S
	let showOffPeakOptions = false;
	let showSmartChargeOptions = false;
	let showEvDrainProtectionOptions = false;
	let offPeakStart = '00:30';
	let offPeakEnd = '04:30';
	let v2hMaxAmps = 16;

	//Charge variables
	let amps_value = 16; // hardware max
	let soc_range_value = 90; // default
	let chargeEco = false;
	let wsInterval = 0;

	// Snapshot state — source of truth, populated from Data messages
	let snapshotMode = '';
	let snapshotSoc = 0;
	let snapshotDcKw = 0;
	let snapshotMeterW = 0;
	let snapshotSmartChargeRequest = false;
	let snapshotSmartChargeActive = false;
	let snapshotEvDrainProtectionRequest = false;
	let snapshotEvDrainProtectionActive = false;
	let snapshotSmartExportRequest = false;
	let snapshotSmartExportActive = false;
	let snapshotSmartExportExcessSolarRequest = false;
	let snapshotSmartExportExcessSolarActive = false;
	let snapshotReadyToDriveRequest = false;
	let snapshotReadyToDriveActive = false;
	let snapshotOffPeakChargingRequest = false;
	let snapshotOffPeakChargingActive = false;
	function ledClass(request: boolean, active: boolean): string {
		if (active)  return 'inline-block w-2 h-2 rounded-full bg-success-500 ml-1.5 flex-shrink-0';
		if (request) return 'inline-block w-2 h-2 rounded-full ring-1 ring-success-500 ml-1.5 flex-shrink-0';
		return 'inline-block w-2 h-2 rounded-full bg-surface-400 ml-1.5 flex-shrink-0 opacity-30';
	}

	$: operationalMode.set(snapshotMode);

	const DC_BAR_MAX_W = 5600; // 14A * 400V
	$: dcBarPct   = Math.min(Math.abs(snapshotDcKw) / DC_BAR_MAX_W * 50, 50);
	$: dcBarStyle = snapshotDcKw >= 0
		? `bottom: 50%; height: ${dcBarPct}%`
		: `top: 50%; height: ${dcBarPct}%`;
	$: dcBarColor = snapshotDcKw > 50 ? 'bg-emerald-500' : snapshotDcKw < -50 ? 'bg-blue-400' : 'bg-surface-400';

	$: gridBarPct   = Math.min(Math.abs(snapshotMeterW) / DC_BAR_MAX_W * 50, 50);
	$: gridBarStyle = snapshotMeterW >= 0
		? `bottom: 50%; height: ${gridBarPct}%`
		: `top: 50%; height: ${gridBarPct}%`;
	$: gridBarColor = snapshotMeterW > 50 ? 'bg-amber-500' : snapshotMeterW < -50 ? 'bg-emerald-500' : 'bg-surface-400';

	$: modePillClass =
		snapshotMode === 'V2h'    ? 'border-2 border-blue-500 text-blue-500' :
		snapshotMode === 'Charge' ? 'border-2 border-emerald-500 text-emerald-500' :
		snapshotMode === 'Idle'   ? 'bg-black text-white border-2 border-white' :
		'border-2 border-surface-400 text-surface-400';

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

	function sendSettings() {
		if (!socket || socket.readyState !== WebSocket.OPEN) return;
		socket.send(JSON.stringify({
			cmd: {
				SetSettings: {
					v2h_soc_min:          v2h_soc_min,
					v2h_soc_max:          v2h_soc_max,
					v2h_max_amps:         v2hMaxAmps,
					self_use:             selfUse,
					export_excess_solar:  exportExcessSolar,
					ev_drain_protection:  evDrainProtection,
					ready_to_drive:            readyToDrive,
					ready_to_drive_end_time:   readyToDriveEndTime,
					ready_to_drive_start_time: readyToDriveStartTime,
					ready_to_drive_soc:        readyToDriveSoc,
					ready_to_drive_days:       readyToDriveDays,
					off_peak_charging:    OffPeakCharging,
					off_peak_start:       offPeakStart,
					off_peak_end:         offPeakEnd,
					smart_charge:         smartCharge,
					smart_export:                smartExport,
					smart_export_limit_w:        smartExportLimit,
					smart_export_excess_solar:   smartExportExcessSolar,
					charge_soc_limit:            soc_range_value,
					charge_amps:          amps_value,
					charge_eco:           chargeEco,
				}
			}
		}));
	}

	function applySettings(s: OperatorSettings) {
		v2h_soc_min        = s.v2h_soc_min;
		v2h_soc_max        = s.v2h_soc_max;
		v2hMaxAmps         = s.v2h_max_amps;
		selfUse            = s.self_use;
		exportExcessSolar  = s.export_excess_solar;
		evDrainProtection  = s.ev_drain_protection;
		readyToDrive          = s.ready_to_drive;
		readyToDriveEndTime   = s.ready_to_drive_end_time;
		readyToDriveStartTime = s.ready_to_drive_start_time;
		readyToDriveSoc       = s.ready_to_drive_soc;
		readyToDriveDays      = s.ready_to_drive_days;
		OffPeakCharging    = s.off_peak_charging;
		offPeakStart       = s.off_peak_start;
		offPeakEnd         = s.off_peak_end;
		smartCharge        = s.smart_charge;
		smartExport             = s.smart_export;
		smartExportLimit        = s.smart_export_limit_w;
		smartExportExcessSolar  = s.smart_export_excess_solar;
		soc_range_value         = s.charge_soc_limit;
		amps_value         = s.charge_amps;
		chargeEco          = s.charge_eco;
	}

	function submitCharge(event: Event) {
		event.preventDefault();
		socket.send(JSON.stringify({
			cmd: { SetMode: { Charge: { amps: amps_value, eco: chargeEco, soc_limit: soc_range_value } } }
		}));
	}

	function sendModeChange(mode: string) {
		if (!socket || socket.readyState !== WebSocket.OPEN) return;
		let payload;
		if (mode === 'Discharge') {
			payload = { cmd: { SetMode: { Discharge: { soc_limit: soc_range_value, amps: amps_value } } } };
		} else {
			payload = { cmd: { SetMode: mode } };
		}
		socket.send(JSON.stringify(payload));
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
					items.unshift(message.Data);
					if (items.length > 80) items.pop();
					return items;
				});
				const rawState = message.Data.state;
				snapshotMode = typeof rawState === 'object' && rawState !== null
					? Object.keys(rawState)[0]
					: (rawState ?? '');
				snapshotSoc = message.Data.soc ?? 0;
				snapshotDcKw = message.Data.dc_w ?? 0;
				snapshotMeterW = message.Data.meter_kw ?? 0;
				snapshotSmartChargeRequest = message.Data.smart_charge_request ?? false;
				snapshotSmartChargeActive = message.Data.smart_charge_active ?? false;
				snapshotEvDrainProtectionRequest = message.Data.ev_drain_protection_request ?? false;
				snapshotEvDrainProtectionActive = message.Data.ev_drain_protection_active ?? false;
				snapshotSmartExportRequest = message.Data.smart_export_request ?? false;
				snapshotSmartExportActive = message.Data.smart_export_active ?? false;
				snapshotSmartExportExcessSolarRequest = message.Data.smart_export_excess_solar_request ?? false;
				snapshotSmartExportExcessSolarActive = message.Data.smart_export_excess_solar_active ?? false;
				snapshotReadyToDriveRequest = message.Data.ready_to_drive_request ?? false;
				snapshotReadyToDriveActive = message.Data.ready_to_drive_active ?? false;
				snapshotOffPeakChargingRequest = message.Data.off_peak_charging_request ?? false;
				snapshotOffPeakChargingActive = message.Data.off_peak_charging_active ?? false;
				if (message.Data.settings) applySettings(message.Data.settings);
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
		<button
			class="btn variant-filled {snapshotMode === 'Idle' ? 'variant-filled-primary' : ''}"
			on:click={() => sendModeChange('Idle')}
		>
			Idle
		</button>
	</div>

	<!--Smart Self-Powered Card-->
	<div class="card p-4">
		<button
			class="btn variant-filled {snapshotMode === 'V2h' ? 'variant-filled-primary' : ''}"
			on:click={() => sendModeChange('V2h')}
		>
			V2H
		</button>

			<!-- SOC Min -->
			<div class="mt-2">
				<RangeSlider name="soc-min" bind:value={v2h_soc_min} on:change={sendSettings} min={0} max={100} step={5} ticked>
					<div class="flex justify-between items-center">
						<div class="text-xs">SoC Min</div>
						<div class="text-xs">{v2h_soc_min}%</div>
					</div>
				</RangeSlider>
			</div>
			<!-- SOC Max -->
			<div class="mt-2">
				<RangeSlider name="soc-max" bind:value={v2h_soc_max} on:change={sendSettings} min={0} max={100} step={5} ticked>
					<div class="flex justify-between items-center">
						<div class="text-xs">SoC Max</div>
						<div class="text-xs">{v2h_soc_max}%</div>
					</div>
				</RangeSlider>
			</div>


		<!-- Max Amps -->
		<div class="mt-2">
			<RangeSlider name="v2h-amps" bind:value={v2hMaxAmps} on:change={sendSettings} min={1} max={16} step={1} ticked>
				<div class="flex justify-between items-center">
					<div class="text-xs">Max Amps</div>
					<div class="text-xs">{v2hMaxAmps}A</div>
				</div>
			</RangeSlider>
		</div>

		<!-- Options -->
		<div class="mt-4 flex flex-col gap-3">

			<div class="flex justify-between items-center text-sm">
				<span class="flex items-center gap-1" title="Discharge battery to offset house use">
					Self-Use
					<span class={ledClass(false, snapshotDcKw < -50)}></span>
				</span>
				<label class="relative inline-flex items-center cursor-pointer">
					<input type="checkbox" bind:checked={selfUse} class="sr-only peer" on:change={sendSettings} />
					<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
				</label>
			</div>

			<div class="flex justify-between items-center text-sm">
				<span class="flex items-center gap-1" title="Allow excess solar to be exported">
					Export Excess Solar
					<span class={ledClass(false, snapshotMeterW < -50)}></span>
				</span>
				<label class="relative inline-flex items-center cursor-pointer">
					<input type="checkbox" bind:checked={exportExcessSolar} class="sr-only peer" on:change={sendSettings} />
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
						class="flex items-center gap-1 hover:text-primary-500 transition-colors mr-2"
						on:click={() => (showReadyToDriveOptions = !showReadyToDriveOptions)}>
						<span class="text-xs w-3 text-center">{showReadyToDriveOptions ? '▼' : '▶'}</span>
						<span>Ready to Drive</span>
						<span class={ledClass(snapshotReadyToDriveRequest, snapshotReadyToDriveActive)}></span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={readyToDrive} class="sr-only peer" on:change={sendSettings} />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showReadyToDriveOptions}
					<div class="mt-2 ml-4 flex flex-col gap-2 text-surface-600 dark:text-surface-300">
						<div class="mt-1">
							<RangeSlider name="rtd-soc" bind:value={readyToDriveSoc} on:change={sendSettings} min={10} max={100} step={5} ticked>
								<div class="flex justify-between items-center">
									<div class="text-xs">Target SoC</div>
									<div class="text-xs">{readyToDriveSoc}%</div>
								</div>
							</RangeSlider>
						</div>
						<div class="flex justify-between items-center">
							<span>Start charge</span>
							<input type="time" value={readyToDriveStartTime} disabled class="input w-28 text-sm py-0.5 px-1 opacity-50 cursor-not-allowed" />
						</div>
						<div class="flex justify-between items-center">
							<span>Ready by</span>
							<input type="time" bind:value={readyToDriveEndTime} on:change={sendSettings} class="input w-28 text-sm py-0.5 px-1" />
						</div>
						<div class="flex items-center gap-2">
							{#each ['M','T','W','T','F','S','S'] as day, i}
								<label class="flex flex-col items-center cursor-pointer">
									<span class="text-xs mb-0.5">{day}</span>
									<input type="checkbox" bind:checked={readyToDriveDays[i]} class="w-4 h-4 accent-primary-500" on:change={sendSettings} />
								</label>
							{/each}
						</div>
					</div>
				{/if}
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors mr-2"
						on:click={() => (showOffPeakOptions = !showOffPeakOptions)}>
						<span class="text-xs w-3 text-center">{showOffPeakOptions ? '▼' : '▶'}</span>
						<span>Off-Peak Charging</span>
						<span class={ledClass(snapshotOffPeakChargingRequest, snapshotOffPeakChargingActive)}></span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={OffPeakCharging} class="sr-only peer" on:change={sendSettings} />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showOffPeakOptions}
					<div class="mt-2 ml-4 flex flex-col gap-2 text-surface-600 dark:text-surface-300">
						<div class="flex justify-between items-center">
							<span>Start</span>
							<input type="time" bind:value={offPeakStart} on:change={sendSettings} class="input w-28 text-sm py-0.5 px-1" />
						</div>
						<div class="flex justify-between items-center">
							<span>End</span>
							<input type="time" bind:value={offPeakEnd} on:change={sendSettings} class="input w-28 text-sm py-0.5 px-1" />
						</div>
					</div>
				{/if}
			</div>

			<!-- Smart (MQTT-driven) Controls -->
			<div class="mt-1 text-xs font-semibold text-surface-500 dark:text-surface-400">
				Smart
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors mr-2"
						on:click={() => (showSmartExportOptions = !showSmartExportOptions)}>
						<span class="text-xs w-3 text-center">{showSmartExportOptions ? '▼' : '▶'}</span>
						<span>Smart Export</span>
						<span class={ledClass(snapshotSmartExportRequest, snapshotSmartExportActive)}></span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={smartExport} class="sr-only peer" on:change={sendSettings} />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showSmartExportOptions}
					<div class="mt-1 ml-4 text-xs text-surface-400">Export when prices are high</div>
					<div class="mt-2 ml-4 flex justify-between items-center text-surface-600 dark:text-surface-300">
						<span>Export Limit</span>
						<div class="flex items-center gap-1">
							<input type="number" bind:value={smartExportLimit} on:change={sendSettings}
								min="0" max="10000" step="100"
								class="input w-20 text-right text-sm py-0.5 px-1" />
							<span class="text-xs">W</span>
						</div>
					</div>
				{/if}
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors mr-2"
						on:click={() => (showSmartExportExcessSolarOptions = !showSmartExportExcessSolarOptions)}>
						<span class="text-xs w-3 text-center">{showSmartExportExcessSolarOptions ? '▼' : '▶'}</span>
						<span>Smart Export Excess Solar</span>
						<span class={ledClass(snapshotSmartExportExcessSolarRequest, snapshotSmartExportExcessSolarActive)}></span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={smartExportExcessSolar} class="sr-only peer" on:change={sendSettings} />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showSmartExportExcessSolarOptions}
					<div class="mt-1 ml-4 text-xs text-surface-400">
						Exports when solar export rate &gt; overnight import rate
					</div>
				{/if}
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors mr-2"
						on:click={() => (showSmartChargeOptions = !showSmartChargeOptions)}>
						<span class="text-xs w-3 text-center">{showSmartChargeOptions ? '▼' : '▶'}</span>
						<span>Smart Charge</span>
						<span class={ledClass(snapshotSmartChargeRequest, snapshotSmartChargeActive)}></span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={smartCharge} class="sr-only peer" on:change={sendSettings} />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showSmartChargeOptions}
					<div class="mt-1 ml-4 text-xs text-surface-400">Charge during cheap import slots</div>
					<div class="mt-0.5 ml-4 text-xs text-surface-400">uses Charge SoC/Amps</div>
				{/if}
			</div>

			<div class="text-sm">
				<div class="flex items-center justify-between">
					<button type="button"
						class="flex items-center gap-1 hover:text-primary-500 transition-colors mr-2"
						on:click={() => (showEvDrainProtectionOptions = !showEvDrainProtectionOptions)}>
						<span class="text-xs w-3 text-center">{showEvDrainProtectionOptions ? '▼' : '▶'}</span>
						<span>EV Drain Protection</span>
						<span class={ledClass(snapshotEvDrainProtectionRequest, snapshotEvDrainProtectionActive)}></span>
					</button>
					<label class="relative inline-flex items-center cursor-pointer">
						<input type="checkbox" bind:checked={evDrainProtection} class="sr-only peer" on:change={sendSettings} />
						<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
					</label>
				</div>
				{#if showEvDrainProtectionOptions}
					<div class="mt-1 ml-4 text-xs text-surface-400">
						Prevents discharge into EV charging load
					</div>
				{/if}
			</div>

		</div>
	</div>

	<!-- Charge Card -->
	<div class="card p-4">
		<button
			class="btn variant-filled flex justify-between items-center mb-4 {snapshotMode === 'Charge' ? 'variant-filled-primary' : ''}"
			on:click={submitCharge}>
			Charge
		</button>

		<RangeSlider
			name="soc"
			id="range-slider-boost-soc"
			bind:value={soc_range_value}
			on:change={sendSettings}
			min={10}
			max={100}
			step={5}
			ticked
		>
			<div class="flex justify-between items-center">
				<div class="text-xs">SoC</div>
				<div class="text-xs">{soc_range_value} / 100</div>
			</div>
		</RangeSlider>

		<RangeSlider
			name="amps"
			id="range-slider-amps"
			bind:value={amps_value}
			on:change={sendSettings}
			max={16}
			step={1}
			ticked
		>
			<div class="flex justify-between items-center">
				<div class="text-xs">Amps</div>
				<div class="text-xs">{amps_value} / 16</div>
			</div>
		</RangeSlider>

		<div class="mt-3 flex justify-between items-center text-sm">
			<span title="Charge using solar power only">Eco (Use Solar Only)</span>
			<label class="relative inline-flex items-center cursor-pointer">
				<input type="checkbox" bind:checked={chargeEco} class="sr-only peer" on:change={sendSettings} />
				<div class="w-9 h-5 bg-surface-300 rounded-full peer peer-checked:bg-primary-500 after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:after:translate-x-4" />
			</label>
		</div>
	</div>

	<!-- Operational Mode Card -->
	<div class="card p-4 text-center">
		<div class="text-2xl font-bold mb-4 inline-block rounded-lg px-4 py-1 {modePillClass}">{snapshotMode || '—'}</div>
		<div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-1">Battery</div>
		<div class="text-2xl font-bold mb-4">{snapshotSoc.toFixed(1)}%</div>
		<div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-1">{snapshotDcKw > 50 ? 'Charging' : snapshotDcKw < -50 ? 'Discharging' : 'Power'}</div>
		<div class="flex flex-col items-center mb-2">
			<div class="relative w-6 h-28 bg-surface-300 rounded-full overflow-hidden">
				<div class="absolute left-0 w-full rounded-full {dcBarColor}" style="{dcBarStyle}"></div>
				<div class="absolute left-0 w-full top-1/2 h-px bg-surface-500"></div>
			</div>
		</div>
		<div class="text-2xl font-bold">{Math.abs(Math.round(snapshotDcKw))} W</div>
		<div class="text-sm font-semibold text-surface-500 dark:text-surface-400 mb-1 mt-3">{snapshotMeterW > 50 ? 'Grid Import' : snapshotMeterW < -50 ? 'Grid Export' : 'Grid'}</div>
		<div class="flex flex-col items-center mb-2">
			<div class="relative w-6 h-28 bg-surface-300 rounded-full overflow-hidden">
				<div class="absolute left-0 w-full rounded-full {gridBarColor}" style="{gridBarStyle}"></div>
				<div class="absolute left-0 w-full top-1/2 h-px bg-surface-500"></div>
			</div>
		</div>
		<div class="text-2xl font-bold">{Math.abs(Math.round(snapshotMeterW))} W</div>
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
						<th class="p-2 text-left text-secondary-500" colspan="12" style="border-left: 1px solid var(--color-surface-400);">Pre-Charger</th>
						<th class="p-2 text-left text-tertiary-500" colspan="2" style="border-left: 1px solid var(--color-surface-400);">Meter</th>
						<th class="p-2 text-left text-success-500" colspan="4" style="border-left: 1px solid var(--color-surface-400);">Supervisory</th>
					</tr>
					<tr style="border-bottom: 1px solid var(--color-surface-300);">
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">SoC</th>
						<th class="p-2 text-left text-xs">State</th>
						<th class="p-2 text-left text-xs">Req A</th>
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">PRE State</th>
						<th class="p-2 text-left text-xs">DC V SP</th>
						<th class="p-2 text-left text-xs">DC A SP</th>
						<th class="p-2 text-left text-xs">DC Out V</th>
						<th class="p-2 text-left text-xs">DC Out A</th>
						<th class="p-2 text-left text-xs">DC W</th>
						<th class="p-2 text-left text-xs">DC Bus V</th>
						<th class="p-2 text-left text-xs">AC V</th>
						<th class="p-2 text-left text-xs">AC A</th>
						<th class="p-2 text-left text-xs">AC W</th>
						<th class="p-2 text-left text-xs">Temp</th>
						<th class="p-2 text-left text-xs">Fan</th>
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">Total W</th>
						<th class="p-2 text-left text-xs">Phase W</th>
						<th class="p-2 text-left text-xs" style="border-left: 1px solid var(--color-surface-400);">SE Req</th>
						<th class="p-2 text-left text-xs">SEES Req</th>
						<th class="p-2 text-left text-xs">SC Req</th>
						<th class="p-2 text-left text-xs">EDP Req</th>
					</tr>
				</thead>
				<tbody>
					{#each $realTimeData as rtd}
						<tr>
							<td class="p-2">{rtd.time}</td>
							<td class="p-2">{Math.round(rtd.soc ?? 0)}%</td>
							<td class="p-2">{typeof rtd.state === 'object' && rtd.state !== null ? Object.keys(rtd.state)[0] : rtd.state}</td>
							<td class="p-2">{(rtd.requested_amps ?? 0).toFixed(1)}A</td>
							<td class="p-2">{rtd.pre_state ?? '—'}</td>
							<td class="p-2">{(rtd.dc_volts_setpoint ?? 0).toFixed(1)}V</td>
							<td class="p-2">{(rtd.dc_amps_setpoint ?? 0).toFixed(1)}A</td>
							<td class="p-2">{(rtd.dc_output_volts ?? 0).toFixed(1)}V</td>
							<td class="p-2">{(rtd.dc_output_amps ?? 0).toFixed(1)}A</td>
							<td class="p-2">{Math.round(rtd.dc_w ?? 0)}W</td>
							<td class="p-2">{(rtd.dc_bus_volts ?? 0).toFixed(1)}V</td>
							<td class="p-2">{(rtd.ac_volts ?? 0).toFixed(1)}V</td>
							<td class="p-2">{(rtd.ac_amps ?? 0).toFixed(1)}A</td>
							<td class="p-2">{Math.round(rtd.ac_w ?? 0)}W</td>
							<td class="p-2">{(rtd.pre_temp ?? 0).toFixed(1)}ºC</td>
							<td class="p-2">{Math.round(rtd.pre_fan ?? 0)}%</td>
							<td class="p-2">{Math.round(rtd.meter_kw ?? 0)}</td>
							<td class="p-2">{rtd.phase_w != null ? Math.round(rtd.phase_w) + 'W' : '—'}</td>
							<td class="p-2">{rtd.smart_export_request ? '✓' : '—'}</td>
							<td class="p-2">{rtd.smart_export_excess_solar_request ? '✓' : '—'}</td>
							<td class="p-2">{rtd.smart_charge_request ? '✓' : '—'}</td>
							<td class="p-2">{rtd.ev_drain_protection_request ? '✓' : '—'}</td>
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
