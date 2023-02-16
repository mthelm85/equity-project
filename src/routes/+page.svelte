<script>
	import { onMount } from 'svelte';
	import Dialog, { Title, Content, Actions } from '@smui/dialog';
	import SegmentedButton, { Segment, Label } from '@smui/segmented-button';
	import Button, { Label as ButtonLabel } from '@smui/button';
	import IconButton, { Icon } from '@smui/icon-button';
	import Textfield from '@smui/textfield';
	import HelperText from '@smui/textfield/helper-text';
	import pumas from '$lib/data/ipums_puma_2010.json';
	import data from '$lib/data/data.json';
	import * as d3 from 'd3';

	const varMap = {
		'Black Workers': 'p_black',
		'Hispanic Workers': 'p_hisp',
		'Poverty': 'p_pov',
		'Educational Attainment': 'mdn_edu',
		'Unemployment': 'p_unemp',
		'Disabled Workers': 'p_disabled'
	};

	const scaledVarMap = {
		p_black: 'black_t',
		p_hisp: 'hisp_t',
		p_pov: 'pov_t',
		mdn_edu: 'edu_t',
		p_unemp: 'unemp_t',
		p_disabled: 'disabled_t'
	};

	let L;
	let choices = ['Black Workers', 'Hispanic Workers', 'Poverty', 'Educational Attainment', 'Unemployment', 'Disabled Workers'];
	let selected = [];
	let innerHeight;
	let layerGroup;
	let jsonLayer;
	let open = false;

	let weightBlack = '';
	let weightPoverty = '';
	let weightEduAttnmnt = '';
	let weightUnemployment = '';

	$: selectedVars = selected.map((s) => varMap[s]);

	$: if (selectedVars) {
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

		if (layerGroup && jsonLayer) {
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
							opacity: 1,
							color: 'black',
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
							popupText.push(
								`${selected[i]}: <strong> ${puma[0][`${varMap[selected[i]]}`]}</strong>`
							);
							popupText.push('<br />');
						}
					}

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

		L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
			attribution:
				'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors',
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
</script>

<svelte:window bind:innerHeight on:resize={resizeMap} />

<div class="centered">
	<SegmentedButton segments={choices} let:segment bind:selected>
		<Segment {segment}>
			<Label>{segment}</Label>
		</Segment>
	</SegmentedButton>
	<IconButton style="margin-left: 8px;" on:click={() => (open = true)}>
		<Icon class="material-icons">help_outline</Icon>
	</IconButton>
</div>

<!-- <div class="centered">
	<Textfield
		variant="outlined"
		bind:value={weightBlack}
		type="number"
		input$step="0.1"
		style="width: 153.06px; height: 36px;"
	/>
	<Textfield
		variant="outlined"
		bind:value={weightPoverty}
		
		style="width: 94.73px; height: 36px;"
	/>
	<Textfield
		variant="outlined"
		bind:value={weightEduAttnmnt}
		
		style="width: 235.28px; height: 36px;"
	/>
	<Textfield
		variant="outlined"
		bind:value={weightUnemployment}
		
		style="width: 152.64px; height: 36px;"
	/>
</div> -->

<div id="map-container" style="height: {innerHeight - 100}px; width: 100%;" />
{#if layerGroup}
<div class="caption">Source: 2021 American Community Survey 5-Year Public Use Microdata Sample</div>
{/if}
<Dialog
  bind:open
  aria-labelledby="simple-title"
  aria-describedby="simple-content"
  style="z-index: 999 !important"
  fullscreen
>
  <!-- Title cannot contain leading whitespace due to mdc-typography-baseline-top() -->
  <Title id="simple-title" style="padding: 24px;">About This App</Title>
  <Content id="simple-content">
	<p>
		This application allows you to explore a variety of workforce characteristics that have been estimated from the
		2021 American Community Survey 5-Year Public Use Microdata Sample.
	</p>
	<p>A brief explanation of each of the variables is below:</p>

	<ul>
		<li><strong>Black Workers:</strong> The percentage of workers that are Black</li>
		<li><strong>Hispanic Workers:</strong> The percentage of workers that are Hispanic</li>
		<li><strong>Poverty:</strong> The percentage of workers whose income-to-poverty ratio is less than 1</li>
		<li>
			<strong>Educational Attainment:</strong> The median educational attainment for workers:
			<ul>
				<li><strong>16:</strong> Regular high school diploma</li>
				<li><strong>17:</strong> GED or alternative credential</li>
				<li><strong>18:</strong> Some college, but less than 1 year</li>
				<li><strong>19:</strong> 1 or more years of college credit, no degree</li>
				<li><strong>20:</strong> Associate's degree</li>
				<li><strong>21:</strong> Bachelor's degree</li>
				<li><strong>22:</strong> Master's degree</li>
			</ul>
		</li>
		<li><strong>Unemployment:</strong> The civilian unemployment rate</li>
		<li><strong>Disabled Workers:</strong> The percentage of workers that have a disability</li>
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
	.caption {
		font-family: 'Roboto Mono';
		font-size: 14px;
		font-weight: 100;
	}
	.centered {
		display: flex;
		justify-content: center;
		align-items: center;
		margin-bottom: 8px;
	}
</style>
