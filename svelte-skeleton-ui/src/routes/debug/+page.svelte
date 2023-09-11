<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';

	interface X100 {
		minimum_charge_current: number;
		minimum_battery_voltage: number;
		maximum_battery_voltage: number;
		constant_of_charging_rate_indication: number;
	}

	interface X101 {
		max_charging_time_10s_bit: number;
		max_charging_time_1min_bit: number;
		estimated_charging_time: number;
		rated_battery_capacity: number;
	}

	interface X102Faults {
		fault_battery_voltage_deviation: boolean;
		fault_high_battery_temperature: boolean;
		fault_battery_current_deviation: boolean;
		fault_battery_undervoltage: boolean;
		fault_battery_overvoltage: boolean;
	}

	interface X102Status {
		status_discharge_compatible: boolean;
		status_normal_stop_request: boolean;
		status_vehicle: boolean;
		status_charging_system: boolean;
		status_vehicle_shifter_position: boolean;
		status_vehicle_charging: boolean;
	}

	interface X102 {
		control_protocol_number_ev: number;
		target_battery_voltage: number;
		charging_current_request: number;
		faults: X102Faults;
		status: X102Status;
		state_of_charge: number;
	}

	interface X108 {
		available_output_current: number;
		avaible_output_voltage: number;
		welding_detection: number;
		threshold_voltage: number;
	}

	interface X109Status {
		status_charger_stop_control: boolean;
		fault_charging_system_malfunction: boolean;
		fault_battery_incompatibility: boolean;
		status_vehicle_connector_lock: boolean;
		fault_station_malfunction: boolean;
		status_station: boolean;
	}

	interface X109 {
		status: X109Status;
		control_protocol_number_qc: number;
		output_voltage: number;
		output_current: number;
		discharge_compatitiblity: boolean;
		remaining_charging_time_10s_bit: number;
		remaining_charging_time_1min_bit: number;
	}

	interface X200 {
		maximum_discharge_current: number;
		minimum_discharge_voltage: number;
		minimum_battery_discharge_level: number;
		max_remaining_capacity_for_charging: number;
	}

	interface X208 {
		discharge_current: number;
		input_voltage: number;
		input_current: number;
		lower_threshold_voltage: number;
	}

	interface X209 {
		sequence: number;
		remaing_discharge_time: number;
	}

	// interface ChargeParameters {
	// 	amps: number | null;
	// 	eco: boolean | null;
	// 	soc_limit: number | null;
	// }

	// interface Charge {
	// 	ChargeParameters?;
	// }

	interface WebSocketData {
		// pins?: Pins;
		x100: X100;
		x101: X101;
		x102: X102;
		x108: X108;
		x109: X109;
		x200: X200;
		x208: X208;
		x209: X209;
		state: any;
		amps: number;
	}

	// {"Debug":{"x100":{"minimum_charge_current":0,"minimum_battery_voltage":0,"maximum_battery_voltage":0,"constant_of_charging_rate_indication":0},"x101":{"max_charging_time_10s_bit":0,"max_charging_time_1min_bit":0,"estimated_charging_time":0,"rated_battery_capacity":0},"x102":{"control_protocol_number_ev":0,"target_battery_voltage":0,"charging_current_request":0,"faults":{"fault_battery_voltage_deviation":false,"fault_high_battery_temperature":false,"fault_battery_current_deviation":false,"fault_battery_undervoltage":false,"fault_battery_overvoltage":false},"status":{"status_discharge_compatible":false,"status_normal_stop_request":false,"status_vehicle":false,"status_charging_system":false,"status_vehicle_shifter_position":false,"status_vehicle_charging":false},"state_of_charge":0},"x108":{"available_output_current":16,"avaible_output_voltage":500,"welding_detection":1,"threshold_voltage":435},"x109":{"status":{"status_charger_stop_control":true,"fault_charging_system_malfunction":false,"fault_battery_incompatibility":false,"status_vehicle_connector_lock":false,"fault_station_malfunction":false,"status_station":false},"control_protocol_number_qc":2,"output_voltage":0,"output_current":0,"discharge_compatitiblity":true,"remaining_charging_time_10s_bit":255,"remaining_charging_time_1min_bit":255},"x200":{"maximum_discharge_current":0,"minimum_discharge_voltage":0,"minimum_battery_discharge_level":0,"max_remaining_capacity_for_charging":0},"x208":{"discharge_current":0,"input_voltage":500,"input_current":16,"lower_threshold_voltage":250},"x209":{"sequence":2,"remaing_discharge_time":0},"state":"Uninitalised","amps":0}} (+page.svelte, line 1649)

	let socket: WebSocket;
	let payloads = writable<WebSocketData[]>([]);
	if (typeof WebSocket !== 'undefined') {
		// Make this a singleton?
		socket = new WebSocket('ws://10.0.1.177:5555');
		socket.addEventListener('open', () => {
			console.log('Opened');
			const message = JSON.stringify({ cmd: 'GetDebug' });

			// Send the message
			socket.send(message);
		});

		socket.addEventListener('message', (event: MessageEvent) => {
			const message = JSON.parse(event.data);
			console.log(JSON.stringify(message));
			if (message.Debug) {
				payloads.set([message.Debug]);
			}
		});
	}
	onMount(() => {
		console.log('on mount');
		if (socket) {
			async function fetchData() {
				try {
					let message = JSON.stringify({ cmd: 'GetDebug' });
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

<main class="container-fluid mx-auto px-4">
	<br />
	{#each $payloads as payload}
		<section>
			<h2>State</h2>
			<hr />
			<br />
			<ul class="indent-5">
				<li>Raw: {payload.state}</li>
			</ul>
			<br />
		</section>
		<h2>EV Data</h2>
		<hr />
		<br />
		<div class="  mx-auto px-4 flex flex-wrap gap-5 md:grid-cols-2">
			<section>
				<h2>X100</h2>
				<ul>
					<li>
						Minimum Charge Current: {payload.x100.minimum_charge_current}
					</li>
					<li>
						Minimum Battery Voltage: {payload.x100.minimum_battery_voltage}
					</li>
					<li>
						Maximum Battery Voltage: {payload.x100.maximum_battery_voltage}
					</li>
					<li>
						Constant of Charging Rate Indication: {payload.x100
							.constant_of_charging_rate_indication}
					</li>
				</ul>
			</section>
			<section>
				<h2>X101</h2>
				<ul>
					<li>
						Max Charging Time (10s Bit): {payload.x101.max_charging_time_10s_bit}
					</li>
					<li>
						Max Charging Time (1min Bit): {payload.x101.max_charging_time_1min_bit}
					</li>
					<li>
						Estimated Charging Time: {payload.x101.estimated_charging_time}
					</li>
					<li>
						Rated Battery Capacity: {payload.x101.rated_battery_capacity}
					</li>
				</ul>
			</section>
			<section>
				<h2>X102</h2>
				<ul>
					<li>
						Control Protocol Number EV: {payload.x102.control_protocol_number_ev}
					</li>
					<li>
						Target Battery Voltage: {payload.x102.target_battery_voltage}
					</li>
					<li>
						Charging Current Request: {payload.x102.charging_current_request}
					</li>
					<li>State of Charge: {payload.x102.state_of_charge}</li>
					<li>Faults:</li>
					<ul class="indent-5">
						<li>
							Fault Battery Voltage Deviation: {payload.x102.faults.fault_battery_voltage_deviation}
						</li>
						<li>
							Fault High Battery Temperature: {payload.x102.faults.fault_high_battery_temperature}
						</li>
						<li>
							Fault Battery Current Deviation: {payload.x102.faults.fault_battery_current_deviation}
						</li>
						<li>
							Fault Battery Undervoltage: {payload.x102.faults.fault_battery_undervoltage}
						</li>
						<li>
							Fault Battery Overvoltage: {payload.x102.faults.fault_battery_overvoltage}
						</li>
					</ul>
					<li>Status:</li>
					<ul class="indent-5">
						<li>
							Status Discharge Compatible: {payload.x102.status.status_discharge_compatible}
						</li>
						<li>
							Status Normal Stop Request: {payload.x102.status.status_normal_stop_request}
						</li>
						<li>Status Vehicle: {payload.x102.status.status_vehicle}</li>
						<li>
							Status Charging System: {payload.x102.status.status_charging_system}
						</li>
						<li>
							Status Vehicle Shifter Position: {payload.x102.status.status_vehicle_shifter_position}
						</li>
						<li>
							Status Vehicle Charging: {payload.x102.status.status_vehicle_charging}
						</li>
					</ul>
				</ul>
			</section>

			<section>
				<h2>X200</h2>
				<ul>
					<li>
						Maximum Discharge Current: {payload.x200.maximum_discharge_current}
					</li>
					<li>
						Minimum Discharge Voltage: {payload.x200.minimum_discharge_voltage}
					</li>
					<li>
						Minimum Battery Discharge Level: {payload.x200.minimum_battery_discharge_level}
					</li>
					<li>
						Max Remaining Capacity for Charging: {payload.x200.max_remaining_capacity_for_charging}
					</li>
				</ul>
			</section>
		</div>
		<br />
		<h2>EVSE Data</h2>
		<hr />
		<br />
		<div class="  mx-auto px-4 flex flex-wrap gap-5 md:grid-cols-2">
			<section>
				<h2>X108</h2>
				<ul>
					<li>
						Available Output Current: {payload.x108.available_output_current}
					</li>
					<li>
						Available Output Voltage: {payload.x108.avaible_output_voltage}
					</li>
					<li>Welding Detection: {payload.x108.welding_detection}</li>
					<li>Threshold Voltage: {payload.x108.threshold_voltage}</li>
				</ul>
			</section>
			<section>
				<h2>X109</h2>
				<ul>
					<li>
						Control Protocol Number QC: {payload.x109.control_protocol_number_qc}
					</li>
					<li>Output Voltage: {payload.x109.output_voltage}</li>
					<li>Output Current: {payload.x109.output_current}</li>
					<li>
						Discharge Compatibility: {payload.x109.discharge_compatitiblity}
					</li>
					<li>
						Remaining Charging Time (10s Bit): {payload.x109.remaining_charging_time_10s_bit}
					</li>
					<li>
						Remaining Charging Time (1min Bit): {payload.x109.remaining_charging_time_1min_bit}
					</li>
					<li>Status:</li>
					<ul class="indent-5">
						<li>
							Charger Stop Control: {payload.x109.status.status_charger_stop_control}
						</li>
						<li>
							Fault Charging System Malfunction: {payload.x109.status
								.fault_charging_system_malfunction}
						</li>
						<li>
							Fault Battery Incompatibility: {payload.x109.status.fault_battery_incompatibility}
						</li>
						<li>
							Vehicle Connector Lock: {payload.x109.status.status_vehicle_connector_lock}
						</li>
						<li>
							Fault Station Malfunction: {payload.x109.status.fault_station_malfunction}
						</li>
						<li>Status Station: {payload.x109.status.status_station}</li>
					</ul>
				</ul>
			</section>
			<section>
				<h2>X208</h2>
				<ul>
					<li>Discharge Current: {payload.x208.discharge_current}</li>
					<li>Input Voltage: {payload.x208.input_voltage}</li>
					<li>Input Current: {payload.x208.input_current}</li>
					<li>
						Lower Threshold Voltage: {payload.x208.lower_threshold_voltage}
					</li>
				</ul>
			</section>
			<section>
				<h2>X209</h2>
				<ul>
					<li>Sequence: {payload.x209.sequence}</li>
					<li>
						Remaining Discharge Time: {payload.x209.remaing_discharge_time}
					</li>
				</ul>
			</section>
		</div>
		<br />
	{/each}
</main>
