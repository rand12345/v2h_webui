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

	function submitCustomCharge(event: Event) {
		event.preventDefault();
		const ampsValue = Number(
			(document.getElementById('range-slider-amps') as HTMLInputElement)?.value
		);
		const socRangeValue = Number(
			(document.getElementById('range-slider-soc') as HTMLInputElement)?.value
		);
		const ecoCheckbox = (document.getElementById('eco') as HTMLInputElement)?.checked;

		const chargePayload = {
			cmd: {
				SetMode: {
					Charge: {
						// maybe use ChargeOptions interface here?
						amps: ampsValue, // can be null
						eco: ecoCheckbox,
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
		socket = new WebSocket('ws://10.0.1.177:5555');
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
				// update mode buttons etc
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
		<h2>Mode Selection {value}</h2>
		<ListBox id="modeRadio" rounded="rounded-container-token" display="flex-row">
			<ListBoxItem on:change={radioModeChange} bind:group={value} name="justify" value="Idle"
				>Idle</ListBoxItem
			>
			<ListBoxItem on:change={radioModeChange} bind:group={value} name="justify" value="V2h"
				>Load matching</ListBoxItem
			>
			<ListBoxItem on:change={radioModeChange} bind:group={value} name="justify" value="Discharge"
				>Discharge vehicle</ListBoxItem
			>
		</ListBox>
		<br />
		<div>
			<h2>Manual Charge Parameters</h2>
			<form id="chargeForm">
				<div class="">
					<RangeSlider
						name="soc"
						id="range-slider-soc"
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

					<label for="eco" title="Permits charge to vehicle from exported energy only"
						>Solar Economy:
						<input type="checkbox" id="eco" name="eco" />
					</label>
				</div>
				<br />
				<button on:click={submitCustomCharge} class="btn variant-filled" type="submit"
					>Charge</button
				>
			</form>
		</div>
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
