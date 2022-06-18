<svelte:head>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.6.0/dist/leaflet.css"
   integrity="sha512-xwE/Az9zrjBIphAcBb3F6JVqxf46+CDLwfLMHloNu6KEQCAWi6HcDUbeOfBIptF7tcCzusKFjFw2yuvEpDL9wQ=="
   crossorigin=""/>
   <link rel="stylesheet" href="https://unpkg.com/leaflet-geosearch@3.0.0/dist/geosearch.css"/>
</svelte:head>

<script>
	import { onMount } from 'svelte';
	import { Container, Form, Input } from 'sveltestrap';
	import Fa from 'svelte-fa';
	import {faMapLocation, faSearch} from '@fortawesome/free-solid-svg-icons';
	import L from 'leaflet';
	import Hls from 'hls.js';

	const listRequestUrl = "https://data-seattlecitygis.opendata.arcgis.com/datasets/SeattleCityGIS::traffic-cameras.geojson";
	let openStreams = [];

	let mapOpen = true;

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

	function makeTemplateHTML(url){
		if (url.endsWith('jpg')) {
			return `<div class="card video-card" style="width: 25%;"><div class="card-body"><img data-id="${url}" class="imageStream img-fluid"></div></div>`;
		} else {
			return `<div class="card video-card" style="width: 25%;"><div class="card-body"><video autoplay controls data-id="${url}" style="width:100%;height:100%;object-fit:cover;"></video></div></div>`;
		}
	}

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

					var existing = document.querySelector(`[data-id="${url}"]`).parentElement.parentElement;
					existing.remove();
				} else {
					openStreams.push(url);
					openStreams = openStreams;
					marker.setStyle({fillColor: '#c17a43'});

					addStream(url);
				}

				console.log(openStreams);
			});
		});

		map.invalidateSize();
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

	function addStream(url){
		var container = document.getElementById('video-container');
		var template = document.createElement('template');
		template.innerHTML = makeTemplateHTML(url).trim();

		var clone = template.content.cloneNode(true);
		var cloned_node = container.appendChild(clone);
		var video = document.querySelector(`[data-id="${url}"]`);

		if (url.endsWith("jpg")) {
			updateImageStream(video);
		} else {
			var hls = new Hls();
			hls.loadSource(url);
			hls.attachMedia(video);
		}
	}

	function updateImageStream(video){
		video.src = video.dataset.id + "?" + new Date().getTime();
	}

	function mapButton(){
		let button = document.getElementById('mapButton');
		let mapElement = document.getElementById('map');
		let contents = document.getElementById('contents');
		mapOpen = !mapOpen;

		button.className = "btn btn-" + (mapOpen ? "primary" : "secondary");
		if (mapOpen){
			mapElement.style.setProperty("height", "70%", "important");
			contents.style.setProperty("height", "23%", "important");
			map.invalidateSize();
		} else {
			mapElement.style.setProperty("height", "0%", "important");
			contents.style.setProperty("height", "93%", "important");
		}
	}

	function widthUpdate(){
		var slider = document.getElementById('widthSlider');
		var videos = document.getElementsByClassName('video-card');

		Array.from(videos).forEach(function(video){
			video.style.setProperty("width", slider.value + "%");
		});
	}

	onMount(async () => {
		var search = document.getElementById('search');

		search.addEventListener('submit', e => {
			e.preventDefault();
		});
	});

	setInterval(function(){
		var imageStreams = document.getElementsByClassName('imageStream');
		Array.from(imageStreams).forEach(updateImageStream);
	}, 5000);

</script>

<div class="d-flex justify-content-end p-2" style="height: 7%;">
	<input type="range" min="5" max="100" value="25" id="widthSlider" class="ms-2 me-4" on:input={widthUpdate}>
	<button type="button" class="btn btn-primary" id="mapButton" on:click={mapButton}><Fa icon={faMapLocation} /></button>
</div>
<div id="contents" class="overflow-auto">
	<Container fluid>
		<ul id="video-container" class="d-flex flex-wrap"></ul>
	</Container>
</div>

<div id="map" class="shadow" use:load>
	<div class="d-flex px-4 py-1" style="position: absolute;z-index: 999">
		<Form inline id="search" class="px-4 py-2">
			<Input type="text" placeholder="Search location..." on:change={searchChanged} />
		</Form>
	</div>
</div>

<style>
    :global(#map) {
        padding: 0;
		height: 70% !important;
        min-width: 100%;
        width: 100%;
        height: 40%;
        display: block;
        z-index: 1;
    }

	:global(#contents) {
		height: 23%;
	}

	:global(.flex-grow) {
		height: 100%;
	}
</style>