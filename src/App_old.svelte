<svelte:head>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.6.0/dist/leaflet.css"
   integrity="sha512-xwE/Az9zrjBIphAcBb3F6JVqxf46+CDLwfLMHloNu6KEQCAWi6HcDUbeOfBIptF7tcCzusKFjFw2yuvEpDL9wQ=="
   crossorigin=""/>
</svelte:head>

<script>
	import { onMount } from 'svelte';
	import { Container, Form, Input, Card, CardBody } from 'sveltestrap';
	import L from 'leaflet';
	import Hls from 'hls.js';

	const listRequestUrl = "https://data-seattlecitygis.opendata.arcgis.com/datasets/SeattleCityGIS::traffic-cameras.geojson";
	let openStreams = [];

	let searchResultMarker = L.circleMarker([0, 0], {
		radius: 8,
		fillColor: '#c14343',
		fillOpacity: 1,
		color: '#ffffff',
		weight: 1
	});
	searchResultMarker.bindTooltip("Click to dismiss");
	searchResultMarker.on('click', () => {
		searchResultMarker.removeFrom(map);
	});

	let map;
	let load = (async(container) => {
        map = L.map(container, {
            center: [47.608, -122.335],
            zoom: 11,
        });

        L.tileLayer("https://server.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}", {
            minZoom: 1,
            maxZoom: 19,
            className: "map-tiles",
            attribution:
                'Tiles &copy; Esri &mdash; Esri, DeLorme, NAVTEQ, TomTom, Intermap, iPC, USGS, FAO, NPS, NRCAN, GeoBase, Kadaster NL, Ordnance Survey, Esri Japan, METI, Esri China (Hong Kong), and the GIS User Community',
        }).addTo(map);

		const response = await fetch(listRequestUrl);
		const cameraList = await response.json();

		cameraList.features.forEach(element => {
			const url = element.properties.OWNERSHIP == "SDOT" ? `https://58cc2dce193dd.streamlock.net:443/live/${element.properties.NAME.replace('.jpg', '.stream')}/playlist.m3u8` : 'https://images.wsdot.wa.gov/nw/' + element.properties.NAME;
			let marker = L.circleMarker([element.geometry.coordinates[1], element.geometry.coordinates[0]], {
				radius: 5,
				fillColor: '#438cc1',
				fillOpacity: 1,
				color: '#ffffff',
				weight: 1
			});
			marker.bindTooltip(element.properties.LOCATION);
			marker.addTo(map);

			marker.on('click', () => {
				const index = openStreams.indexOf(url);

				if (index > -1) {
					//openStreams.splice(index, 1);
					openStreams = openStreams.filter(m => m !== url);
					marker.setStyle({fillColor: '#438cc1'});
				} else {
					openStreams.push(url);
					openStreams = openStreams;
					marker.setStyle({fillColor: '#c17a43'});
				}

				console.log(openStreams);
			});
		});

		openStreams = openStreams;
	});

	const searchChanged = async(e) => {
		const changeValue = (e.target).value;
		
		if (changeValue === "") return;

		const requestUrl = `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(changeValue)}+seattle&format=json&addressdetails=1`;
		const response = await fetch(requestUrl);
		const locationInfo = await response.json();

		if (locationInfo.length < 1 ||
			locationInfo[0].display_name.startsWith("Seattle, King County") ||
			locationInfo[0].address.state != "Washington") return;

		const latitude = locationInfo[0].lat;
		const longitude = locationInfo[0].lon;
		const amenity = locationInfo[0].address.amenity || locationInfo[0].address.tourism;

		map.setView([latitude, longitude], 18);
		searchResultMarker.setLatLng(new L.LatLng(latitude, longitude));
		searchResultMarker.bindTooltip(amenity ? amenity + " (click to dismiss)" : "Click to dismiss");
		searchResultMarker.addTo(map);
	};

	const loadVideo =() => {
		const videoStreams = document.getElementsByClassName("streamPlayer");

		Array.prototype.forEach.call(videoStreams, v => {
			var hls = new Hls();
			hls.loadSource(v.id);
			hls.attachMedia(v);
		});
	};

	onMount(async () => {
		var search = document.getElementById('search');

		search.addEventListener('submit', e => {
			e.preventDefault();
		});
	});

</script>

<div class="overflow-auto" style="height:60%;">
	<Container fluid>
		<Form inline id="search" class="px-4 py-2">
			<Input type="text" placeholder="Search location..." on:change={searchChanged} />
		</Form>
		<ul class="d-flex flex-wrap">
			{#each openStreams as stream}
			<Card class="w-25">
				<CardBody>
					<video autoplay controls id={stream} class="streamPlayer" style="width:100%;height:100%;object-fit:cover;" use:loadVideo><track kind="captions"></video>
				</CardBody>
			</Card>
			{/each}
		</ul>
	</Container>
</div>

<div id="map" use:load />

<style>
    :global(#map) {
        padding: 0;
        min-height: 40%;
        min-width: 100%;
        width: 100%;
        height: 40%;
        display: block;
        z-index: 1;
    }

	:global(.flex-grow) {
		height: 100%;
	}
</style>