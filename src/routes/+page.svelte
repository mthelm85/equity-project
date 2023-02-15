<script>
	import LayoutGrid, { Cell } from '@smui/layout-grid';
	import { onMount } from 'svelte';
	import L, { popup } from 'leaflet';
	import SegmentedButton, { Segment, Label } from '@smui/segmented-button';
	import pumas from '$lib/data/ipums_puma_2010.json';
	import data from '$lib/data/data.json';
	import * as d3 from 'd3';

	let choices = ['Black Workers', 'Poverty', 'Educational Attainment', 'Unemployment'];
	let selected = [];
	let innerHeight;
	let layerGroup;
	let jsonLayer;

	const varMap = {
		'Black Workers': 'p_black',
		Poverty: 'p_pov',
		'Educational Attainment': 'mdn_edu',
		Unemployment: 'p_unemp'
	};

	const scaledVarMap = {
		p_black: 'black_t',
		p_pov: 'pov_t',
		mdn_edu: 'edu_t',
		p_unemp: 'unemp_t'
	};

	$: selectedVars = selected.map((s) => varMap[s]);

	function resizeMap() {
		const map = document.getElementById('map-container');
		map.style.height = `${innerHeight}px`;
	}

	function getColor(val) {
		return d3.scaleQuantize(
			d3.extent(data.map((obj) => obj.average)),
			d3.schemeReds[9]
		)(val);
	}

	function addAverageKey(keys) {
		data.forEach(obj => {
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

	$: if (selectedVars) {
		redrawMap();
	}

	function redrawMap() {
        const keys = Object.keys(data[1]).filter((x) => selectedVars.includes(x)).map(k => scaledVarMap[k]);
		addAverageKey(keys);
		if (layerGroup && jsonLayer) {
			layerGroup.removeLayer(jsonLayer);
			jsonLayer = L.geoJSON(pumas, {
				style: function (feature) {
					if (selected.length > 0) {
                        const puma = data.filter(
							(obj) =>
								obj.ST == feature.properties.STATEFIP && obj.PUMA == feature.properties.PUMA
						);
						return {
							fillColor: puma.length > 0 ? getColor(puma[0].average) : 'white',
							weight: 0.5,
							opacity: 1,
							color: 'black',
							fillOpacity: puma.length > 0 ? 0.7 : 0
						};
					} else {
						return {
							weight: 0.5,
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

	onMount(() => {
		resizeMap();
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
<LayoutGrid>
	<Cell>
		<div class="centered">
			<SegmentedButton segments={choices} let:segment bind:selected>
				<Segment {segment}>
					<Label>{segment}</Label>
				</Segment>
			</SegmentedButton>
		</div>
	</Cell>
	<Cell>
		<div id="map-container" style="height: {0.9 * innerHeight}px;" />
        <div class="caption">Source: 2021 American Community Survey 5-Year Public Use Microdata Sample</div>
	</Cell>
</LayoutGrid>

<style>
	@import 'leaflet/dist/leaflet.css';
    .caption {
        font-family: 'Roboto Mono';
        font-size: 14px;
        font-weight: 100;
    }
	.centered {
		height: 60px;
		display: flex;
		justify-content: center;
		align-items: center;
	}
</style>
