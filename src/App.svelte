<svelte:head>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.6.0/dist/leaflet.css"
   integrity="sha512-xwE/Az9zrjBIphAcBb3F6JVqxf46+CDLwfLMHloNu6KEQCAWi6HcDUbeOfBIptF7tcCzusKFjFw2yuvEpDL9wQ=="
   crossorigin=""/>
   <link rel="stylesheet" href="https://unpkg.com/leaflet-geosearch@3.0.0/dist/geosearch.css"/>

   <title>Seattle Traffic Camera Viewer</title>

   <link
   	rel="stylesheet"
   	href="https://cdn.jsdelivr.net/npm/bootswatch@5.1.0/dist/darkly/bootstrap.min.css"
	/>
</svelte:head>

<script>
	import { onMount } from 'svelte';
	import { Container, Form, Input, Tooltip, Badge } from 'sveltestrap';
	import Fa from 'svelte-fa';
	import {faMapLocation, faVideoSlash, faPlay, faWarning} from '@fortawesome/free-solid-svg-icons';
	import L from 'leaflet';
	import Hls from 'hls.js';

	const listRequestUrl = "https://data-seattlecitygis.opendata.arcgis.com/datasets/SeattleCityGIS::traffic-cameras.geojson";
	let openStreams = [];
	let cameraMarkers = {};
	let openUnitIds = [];

	let mapOpen = true;
	let testsSkipped = false;
	let fullyLoaded = false;

	let searchResultMarker = L.circleMarker([0, 0], {
		radius: 8,
		fillColor: '#c14343',
		fillOpacity: 1,
		color: '#ffffff',
		weight: 2
	});
	searchResultMarker.bindTooltip("Click to dismiss");
	searchResultMarker.on('click', () => {
		searchResultMarker.removeFrom(map);
	});

	function makeTemplateHTML(url){
		if (url.endsWith('jpg')) {
			return `<div class="card video-card" style="width: 25%;"><img data-id="${url}" class="image-stream img-fluid"></div>`;
		} else {
			return `<div class="card video-card" style="width: 25%;"><video autoplay controls data-id="${url}" style="width:100%;height:100%;object-fit:cover;"></video></div>`;
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

		for (const element of cameraList.features) {
			const url = (element.properties.OWNERSHIP == "SDOT" ? `https://58cc2dce193dd.streamlock.net:443/live/${element.properties.NAME.replace('.jpg', '.stream')}/playlist.m3u8` : 'https://images.wsdot.wa.gov/nw/' + element.properties.NAME).replace(/(\r\n|\n|\r)/gm, "");
			const working = testsSkipped || element.properties.OWNERSHIP == "WSDOT" || (await fetch(url, {cache: "force-cache"})).status == 200;
			if (working) {
				let marker = L.circleMarker([element.geometry.coordinates[1], element.geometry.coordinates[0]], {
					radius: 5,
					fillColor: '#438cc1',
					fillOpacity: 1,
					color: '#eaeaea',
					weight: 1
				});
				marker.bindTooltip(element.properties.LOCATION);
				marker.addTo(map);
				cameraMarkers[element.properties.UNITID] = marker;

				marker.on('click', () => {
					if (!fullyLoaded) return;

					const index = openStreams.indexOf(url);

					if (index > -1) {
						openStreams = openStreams.filter(m => m !== url);
						openUnitIds = openUnitIds.filter(m => m !== element.properties.UNITID);
						marker.setStyle({fillColor: '#438cc1'});
						marker.setStyle({color: '#eaeaea'});
						marker.setStyle({weight: 1});

						var existing = document.querySelector(`[data-id="${url}"]`).parentElement;
						existing.remove();
					} else {
						openUnitIds.push(element.properties.UNITID);

						openStreams.push(url);
						openStreams = openStreams;
						marker.setStyle({fillColor: '#c17a43'});
						marker.setStyle({color: '#ffffff'});
						marker.setStyle({weight: 2});

						addStream(url);
					};

					if (fullyLoaded) window.location.hash = "#" + openUnitIds.join();
				});
			};
		};

		document.getElementById('skipTests').style.setProperty("visibility", "hidden");

		map.invalidateSize();
		openStreams = openStreams;
		fullyLoaded = true;

		for (const id of window.location.hash.replace("#", "").split(",")) {
			if (id == '') continue;

			if (id in cameraMarkers) {
				cameraMarkers[id].fire('click');
				cameraMarkers[id].fire('mouseout');
			} else {
				console.warn(`Camera ID ${id} is not available!`);
			}
		};
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

	const skipTestsPressed = async() => {
		testsSkipped = true;
	};

	function addStream(url){
		var container = document.getElementById('video-container');
		var template = document.createElement('template');
		template.innerHTML = makeTemplateHTML(url).trim();

		var clone = template.content.cloneNode(true);
		container.appendChild(clone);
		var video = document.querySelector(`[data-id="${url}"]`);

		if (url.endsWith("jpg")) {
			updateImageStream(video);
		} else {
			var hls = new Hls();
			hls.loadSource(url);
			hls.attachMedia(video);
		};

		video.parentElement.style.setProperty("width", document.getElementById('widthSlider').value + "%");
	}

	function updateImageStream(video){
		video.src = video.dataset.id + "?" + new Date().getTime();
	}

	function mapButton(){
		let button = document.getElementById('mapButton');
		let mapElement = document.getElementById('map');
		let contents = document.getElementById('contents');
		mapOpen = !mapOpen;

		button.className = "btn me-2 btn-" + (mapOpen ? "primary" : "secondary");
		if (mapOpen){
			mapElement.style.setProperty("height", "70%", "important");
			contents.style.setProperty("height", "calc(30% - 60px)", "important");
			map.invalidateSize();
		} else {
			mapElement.style.setProperty("height", "0%", "important");
			contents.style.setProperty("height", "calc(100% - 60px)", "important");
		};

		var url = new URL(window.location.href);
		if (mapOpen) {
			url.searchParams.delete('map');
		} else {
			url.searchParams.set('map', false);
		};
		window.history.replaceState(null, null, url.toString());
	}

	function clearAllButton(){
		openStreams = [];

		var videos = document.getElementsByClassName('video-card');

		Array.from(videos).forEach(function(video){
			video.remove();
		});

		for (const [_, marker] of Object.entries(cameraMarkers)) {
			marker.setStyle({fillColor: '#438cc1'});
			marker.setStyle({color: '#eaeaea'});
			marker.setStyle({weight: 1});
		};

		window.location.hash = "";
	}

	function widthUpdate(){
		var slider = document.getElementById('widthSlider');
		var videos = document.getElementsByClassName('video-card');

		Array.from(videos).forEach(function(video){
			video.style.setProperty("width", slider.value + "%");
		});
	
		var url = new URL(window.location.href);
		url.searchParams.set('videoWidth', slider.value);
		window.history.replaceState(null, null, url.toString());
	}

	function playAllButton(){
		for (let video of document.querySelectorAll("video")) {
			video.play();
			video.currentTime = video.duration - 10;
		};
	}

	onMount(async () => {
		var search = document.getElementById('search');
		var slider = document.getElementById('widthSlider');

		search.addEventListener('submit', e => {
			e.preventDefault();
		});

		var url = new URLSearchParams(window.location.search);

		var mapOpen = url.get('map');
		if (mapOpen === 'false') {
			testsSkipped = true;
			document.getElementById('invalidCameraWarning').style.display = 'unset';
			mapButton();
		};

		var videoWidth = url.get('videoWidth');
		if (videoWidth != null) {
			slider.value = videoWidth;
			widthUpdate();
		};
	});

	setInterval(function(){
		var imageStreams = document.getElementsByClassName('image-stream');
		Array.from(imageStreams).forEach(updateImageStream);
	}, 5000);

</script>

<div class="d-flex justify-content-end p-2" style="height: 60px;">
	<input type="range" min="5" max="100" value="25" id="widthSlider" class="ms-2 me-2" on:input={widthUpdate}>
	<Tooltip target="widthSlider" bottom>Video Width</Tooltip>
	<button type="button" class="btn btn-primary me-2" id="mapButton" on:click={mapButton}><Fa icon={faMapLocation} /></button>
	<Tooltip target="mapButton" bottom>Show/Hide Map</Tooltip>
	<button type="button" class="btn btn-outline-danger me-2" id="clearAllButton" on:click={clearAllButton}><Fa icon={faVideoSlash} /></button>
	<Tooltip target="clearAllButton" bottom>Clear All Streams</Tooltip>
	<button type="button" class="btn btn-outline-success" id="playAllButton" on:click={playAllButton}><Fa icon={faPlay} /></button>
	<Tooltip target="playAllButton" bottom>Play All/Move to Current</Tooltip>
</div>
<div id="contents" class="overflow-auto">
	<Container fluid class="p-0">
		<ul id="video-container" class="d-flex flex-wrap p-0"></ul>
	</Container>
</div>

<div id="map" class="shadow" use:load>
	<div class="d-flex px-4 py-1" style="position: absolute;z-index: 999">
		<Form inline id="search" class="ps-4 py-2">
			<Input type="text" placeholder="Search location..." on:change={searchChanged} />
			<button class="btn btn-primary btn-sm button-fadeout my-2" id="skipTests" type="button" on:click={skipTestsPressed}>
				<span class="spinner-border spinner-border-sm text-nowrap" role="status" aria-hidden="true"></span> <b>Click if loading is taking absurdly long (broken SDOT streams may appear on map)</b>
			</button>
		</Form>
		<Badge class="bg-danger ms-2 my-2" style="height: 38px;font-size:1.5em;display:none;" id="invalidCameraWarning"><Fa icon={faWarning} /></Badge>
		<Tooltip target="invalidCameraWarning" bottom>Camera stream testing was skipped because the site launched with the map hidden. Some video streams that are offline/broken may be visible on the map! To automatically filter out broken video streams (to an extent), ensure the map is visible and reload the page. Depending on your internet connection, it may take a while.</Tooltip>
	</div>
</div>

<style>
    :global(#map) {
        padding: 0;
		height: calc(70%) !important;
        min-width: 100%;
        width: 100%;
        height: 40%;
        display: block;
        z-index: 1;
    }

	:global(#contents) {
		height: calc(30% - 60px);
	}

	:global(.flex-grow) {
		height: 100%;
	}

	:global(video) {
		object-fit: cover;
	}

	:global(.form-control) {
		color: rgb(232, 230, 227) !important;
	}

	:global(.leaflet-control-attribution) {
		background-color: rgba(24, 26, 27, 0.7) !important;
		color: rgb(200, 195, 188) !important;
	}

	:global(.leaflet-bar a) {
		background-color: rgba(24, 26, 27) !important;
		color: rgb(232, 230, 227) !important;
	}

	:root {
		--map-tiles-filter: brightness(0.6) invert(1) contrast(3) hue-rotate(200deg) saturate(0.3) brightness(0.7);
	}

	@media (prefers-color-scheme: dark) {
		:global(.map-tiles) {
			filter:var(--map-tiles-filter, none);
		}
	}

	:global(.leaflet-interactive) {
		-webkit-animation: fade-in 1s;
		animation: fade-in 1s;
	}

	@keyframes fade-in {
		from { opacity: 0; }
		to   { opacity: 1; }
	}

	@-webkit-keyframes fade-in {
		from { opacity: 0; }
		to   { opacity: 1; }
	}
</style>