<script>
	import { onMount } from 'svelte';
	import Dialog, { Title, Content, Actions } from '@smui/dialog';
	import SegmentedButton, { Segment, Label } from '@smui/segmented-button';
	import Button, { Label as ButtonLabel } from '@smui/button';
	import IconButton, { Icon } from '@smui/icon-button';
	import pumas from '$lib/data/ipums_puma_2010.json';
	import data from '$lib/data/data.json';
	import * as d3 from 'd3';

	const varMap = {
		Black: 'p_black',
		Hispanic: 'p_hisp',
		Asian: 'p_asian',
		Native: 'p_native',
		Poverty: 'p_pov',
		'Educational Attainment': 'mdn_edu',
		Unemployment: 'p_unemp',
		Disabled: 'p_disabled',
		Veterans: 'p_vet',
		'Limited English': 'p_lim_eng'
	};

	const scaledVarMap = {
		p_black: 'black_t',
		p_hisp: 'hisp_t',
		p_pov: 'pov_t',
		mdn_edu: 'edu_t',
		p_unemp: 'unemp_t',
		p_disabled: 'disabled_t',
		p_vet: 'vet_t',
		p_lim_eng: 'lim_eng_t',
		p_asian: 'asian_t',
		p_native: 'native_t'
	};

	const eduMap = {
		'17': 'Regular high school diploma',
		'16': 'GED or alternative credential',
		'18': 'Some college, but less than 1 year',
		'19': '1 or more years of college credit, no degree',
		'20': "Associate's degree",
		'21': "Bachelor's degree",
		'22': "Master's degree"
	};

	let L;

	let choices = [
		'Black',
		'Hispanic',
		'Asian',
		'Native',
		'Poverty',
		'Educational Attainment',
		'Unemployment',
		'Disabled',
		'Veterans',
		'Limited English'
	];

	let selected = [];
	let innerHeight;
	let layerGroup;
	let jsonLayer;
	let open = false;

	$: selectedVars = selected.map((s) => varMap[s]);

	$: if (selected) {
		redrawMap();
	}

	function addAverageKey(keys) {
		data.forEach((obj) => {
			let sum = 0;
			let count = 0;

			for (const prop in obj) {
				if (keys.includes(prop)) {
					sum += obj[prop];
					count++;
				}
			}

			const average = sum / count;
			obj.average = average;
		});
	}

	function redrawMap() {
		const keys = Object.keys(data[1])
			.filter((x) => selectedVars.includes(x))
			.map((k) => scaledVarMap[k]);

		addAverageKey(keys);

		if (layerGroup) {
			layerGroup.removeLayer(jsonLayer);

			jsonLayer = L.geoJSON(pumas, {
				style: function (feature) {
					if (selected.length > 0) {
						const puma = data.filter(
							(obj) => obj.ST == feature.properties.STATEFIP && obj.PUMA == feature.properties.PUMA
						);
						return {
							fillColor: puma.length > 0 ? getColor(puma[0].average) : 'white',
							weight: 0.5,
							opacity: 1,
							color: 'black',
							fillOpacity: puma.length > 0 ? 0.6 : 0
						};
					} else {
						return {
							weight: 0.0,
							fillOpacity: 0
						};
					}
				},
				onEachFeature: function (feature, layer) {
					const puma = data.filter(
						(obj) => obj.ST == feature.properties.STATEFIP && obj.PUMA == feature.properties.PUMA
					);

					let popupText = [`<strong>${feature.properties.Name}</strong>`, '<br />'];

					for (let i = 0; i < selected.length; i++) {
						if (puma.length > 0) {
							const selectedVar = varMap[selected[i]];
							if (selectedVar == 'mdn_edu') {
								popupText.push(
									`Median ${selected[i]}: <strong> ${eduMap[puma[0][`${selectedVar}`]]}</strong>`
								);
							} else {
								popupText.push(`${selected[i]}: <strong> ${puma[0][`${selectedVar}`]}%</strong>`);
							}

							popupText.push('<br />');
						}
					}

					puma.length > 0 &&
						popupText.push(`Index Value: <strong>${Math.round(puma[0].average * 100)}</strong>`);
					popupText.length > 0 && layer.bindPopup(popupText.join(''));
				}
			});

			layerGroup.addLayer(jsonLayer);
		}
	}

	function getColor(val) {
		return d3.scaleQuantize(d3.extent(data.map((obj) => obj.average)), d3.schemeReds[9])(val);
	}

	function resizeMap() {
		const map = document.getElementById('map-container');
		map.style.height = `${innerHeight - 100}px`;
	}

	onMount(async () => {
		L = await import('leaflet');

		const mapContainer = document.getElementById('map-container');
		const map = L.map(mapContainer).setView([39.8283, -98.5795], 5);

		map.attributionControl.setPrefix('')

		L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
			attribution: false,
			maxZoom: 18
		}).addTo(map);

		layerGroup = new L.LayerGroup();
		layerGroup.addTo(map);

		jsonLayer = L.geoJSON(pumas, {
			style: {
				weight: 0.5,
				opacity: 1,
				color: 'black',
				fillOpacity: 0.0
			}
		});
	});

	function convertToCSV(headers) {
		const csvRows = [];
		csvRows.push(headers.join(','));
		for (const row of data) {
			const values = headers.map((header) => {
				if (header == 'State') {
					const puma = pumas.features.find(
						(obj) =>
							obj.properties.STATEFIP == row[headers[2]] && obj.properties.PUMA == row[headers[3]]
					);
					return puma.properties.State;
				} else if (header == 'Name') {
					const puma = pumas.features.find(
						(obj) =>
							obj.properties.STATEFIP == row[headers[2]] && obj.properties.PUMA == row[headers[3]]
					);
					return puma.properties.Name.replace(/,/g, ' |');
				} else if (header == 'Index Value') {
					return Math.round(row['average'] * 100);
				} else {
					return row[header];
				}
			});
			csvRows.push(values.join(','));
		}

		return csvRows.join('\n');
	}

	function downloadCSV() {
		const keys = ['State', 'Name', 'ST', 'PUMA']
			.concat(Object.keys(data[1]).filter((x) => selectedVars.includes(x)))
			.concat(['Index Value']);

		const csvData = convertToCSV(keys);
		const blob = new Blob([csvData], { type: 'text/csv' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'data.csv';
		a.click();
		URL.revokeObjectURL(url);
	}
</script>

<svelte:window bind:innerHeight on:resize={resizeMap} />

<div class="centered">
	<SegmentedButton segments={choices} let:segment bind:selected>
		<Segment {segment}>
			<Label style="font-size: 12px;">{segment}</Label>
		</Segment>
	</SegmentedButton>
	<IconButton style="margin-left: 8px;" on:click={() => (open = true)}>
		<Icon class="material-icons">help_outline</Icon>
	</IconButton>
	<IconButton on:click={downloadCSV} disabled={selected.length == 0}>
		<Icon class="material-icons">download</Icon>
	</IconButton>
</div>

<div id="map-container" style="height: {innerHeight - 100}px; width: 100%;" />
<Dialog
	bind:open
	aria-labelledby="simple-title"
	aria-describedby="simple-content"
	style="z-index: 9999 !important"
	fullscreen
>
	<Title id="simple-title" style="padding: 24px;">About</Title>
	<Content id="simple-content">
		<p>
			This application allows you to explore a variety of workforce characteristics that have been
			estimated from the 2021 American Community Survey 5-Year Public Use Microdata Sample. You can
			select a single variable or a combination of variables. If selecting multiple variables, the
			color scale is determined by the average of the (normalized) values of each selected variable.
		</p>
		<p>A brief explanation of each of the variables is below:</p>

		<ul>
			<li><strong>Black:</strong> The percentage of workers that are Black</li>
			<li><strong>Hispanic:</strong> The percentage of workers that are Hispanic</li>
			<li><strong>Asian:</strong> The percentage of workers that are Asian</li>
			<li>
				<strong>Native:</strong> The percentage of workers that are either American Indian, Alaska Native,
				Native Hawaiian, or Pacific Islander
			</li>
			<li>
				<strong>Poverty:</strong> The percentage of workers whose income-to-poverty ratio is less than
				1
			</li>
			<li>
				<strong>Educational Attainment:</strong> The median educational attainment for workers
			</li>
			<li><strong>Unemployment:</strong> The civilian unemployment rate</li>
			<li><strong>Disabled:</strong> The percentage of workers that have a disability</li>
			<li><strong>Veterans:</strong> The percentage of workers that have served in the military</li>
			<li>
				<strong>Limited English:</strong> The percentage of workers that do not speak English well, or
				at all
			</li>
		</ul>
	</Content>
	<Actions>
		<Button>
			<ButtonLabel>Close</ButtonLabel>
		</Button>
	</Actions>
</Dialog>

<style>
	@import 'leaflet/dist/leaflet.css';
	.centered {
		display: flex;
		justify-content: center;
		align-items: center;
		margin-bottom: 8px;
	}
</style>
