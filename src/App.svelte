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

    import "leaflet.markercluster/dist/leaflet.markercluster.js";
    import "leaflet.markercluster/dist/MarkerCluster.css";
    import "leaflet.markercluster/dist/MarkerCluster.Default.css";

	import Hls from 'hls.js';

	const datasetRequestUrl = "https://the-sink.github.io/seattle-traffic-cams/data/dataset.geojson";
	const sdotMapApiUrl = "https://the-sink.github.io/seattle-traffic-cams/data/sdot.json";

	let cameras = {};

	let openStreams = [];
	let cameraMarkers = {};
	let openUnitIds = [];

	let mapOpen = true;
	let testsSkipped = false;
	let fullyLoaded = false;

	let warningMessage = "";

	// Set up search result marker
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

	// Returns the associated HTML for building a camera stream card template
	// (cursed)
	function makeTemplateHTML(url, name){
		let streamHeader = `<div class="d-flex justify-content-between px-2 py-1 user-select-none stream-header"><div>${name}</div><div><button type="button" class="btn btn-outline-danger btn-sm p-0 px-1" style="font-weight:900;">X</div></div>`

		if (url.endsWith('jpg')) {
			return `<div class="card video-card mb-2" style="width: 25%;">${streamHeader}<img data-id="${url}" class="image-stream img-fluid"></div>`;
		} else {
			return `<div class="card video-card mb-2" style="width: 25%;">${streamHeader}<video muted autoplay controls data-id="${url}"></video></div>`;
		}
	}

	function makeMarkerClusterIcon(n) {
		return new L.DivIcon({
			className: "circle-group",
			iconSize: new L.Point(15, 15)
		});
	}

	// Bulk of the code for loading the map & initializing a list of available cameras
	let map;
	let load = (async(container) => {
		// Set up map base layer

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

		let markers = L.markerClusterGroup({maxClusterRadius: 1, iconCreateFunction: makeMarkerClusterIcon, spiderLegPolylineOptions: {weight: 1, color: '#eee', opacity: 1}});
		map.addLayer(markers);

		// Collect list of cameras

		const datasetResponse = await fetch(datasetRequestUrl);
		const sdotResponse = await fetch(sdotMapApiUrl);

		const datasetCameraList = await datasetResponse.json();
		const sdotCameraList = await sdotResponse.json();

		// Check SDOT first...
		for (const group of sdotCameraList.Features) {
			let coordinates = {lat: group.PointCoordinate[0], lon: group.PointCoordinate[1]};

			for (const element of group.Cameras) {
				let camera = {};

				camera.id = element.Id;
				camera.location = element.Description;
				camera.url = element.ImageUrl;
				camera.coordinates = coordinates;
				camera.source = element.Type.toUpperCase();

				cameras[element.Id] = camera;
			}
		}

		/// ...Then check the Seattle Open Data source as it contains better location data, and override SDOT data if available
		for (const element of datasetCameraList.features) {
			let camera = {};
			
			camera.id = element.properties.UNITID;
			camera.location = element.properties.LOCATION;
			camera.url = element.properties.NAME;
			camera.coordinates = {lat: element.geometry.coordinates[1], lon: element.geometry.coordinates[0]};
			camera.source = element.properties.OWNERSHIP;

			if (element.properties.UNITID in cameras && cameras[element.properties.UNITID].url == camera.url) {
				cameras[element.properties.UNITID] = camera;
			}
		}

		// Iterate over each camera and add them to the map

		for (const [_, camera] of Object.entries(cameras)) {
			const url = (camera.source == "SDOT" ? `https://61e0c5d388c2e.streamlock.net:443/live/${camera.url.replace('.jpg', '.stream')}/playlist.m3u8` : 'https://images.wsdot.wa.gov/nw/' + camera.url).replace(/(\r\n|\n|\r)/gm, "");
			
			// For SDOT streams only, this will try to make sure the feed actually exists.
			// The force-cache mode may present issues if a camera is only temporarily offline, as the cached result will be used even if stale. Better solution needed at some point.
			const working = testsSkipped || camera.source == "WSDOT" || (await fetch(url, {cache: "force-cache"})).status == 200;
			if (working) {

				// Create marker for this camera
				let marker = L.circleMarker([camera.coordinates.lat, camera.coordinates.lon], {
					radius: 5,
					fillColor: '#438cc1',
					fillOpacity: 1,
					color: '#eaeaea',
					weight: 1
				});
				marker.bindTooltip(camera.location);
				markers.addLayer(marker);
				cameraMarkers[camera.id] = marker;

				// Add or remove camera stream from page on click
				marker.on('click', () => {
					if (!fullyLoaded) return;

					const index = openStreams.indexOf(url);

					if (index > -1) {
						openStreams = openStreams.filter(m => m !== url);
						openUnitIds = openUnitIds.filter(m => m !== camera.id);
						marker.setStyle({fillColor: '#438cc1'});
						marker.setStyle({color: '#eaeaea'});
						marker.setStyle({weight: 1});

						var existing = document.querySelector(`[data-id="${url}"]`).parentElement;
						existing.remove();
					} else {
						openUnitIds.push(camera.id);

						openStreams.push(url);
						openStreams = openStreams;
						marker.setStyle({fillColor: '#c17a43'});
						marker.setStyle({color: '#ffffff'});
						marker.setStyle({weight: 2});

						addStream(url, camera);
					};

					if (fullyLoaded) window.location.hash = "#" + openUnitIds.join();
				});
			} else {
				console.log("Camera ", camera.id, " not working, ignoring...");
			};
		};

		document.getElementById('skipTests').style.setProperty("visibility", "hidden");

		map.invalidateSize(); // Sometimes neccesary for the map to load properly for some reason?
		openStreams = openStreams;
		fullyLoaded = true;

		// Open streams that are listed in the URL hash immediately
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

	// Conduct a search using nominatim for locations given by the user in the search box
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

		// Focus map to search result, move search result marker to location and show it
		map.setView([latitude, longitude], 18);
		searchResultMarker.setLatLng(new L.LatLng(latitude, longitude));
		searchResultMarker.bindTooltip(amenity ? amenity + " (click to dismiss)" : "Click to dismiss");
		searchResultMarker.addTo(map);
	};

	// If user presses the skip button during camera testing, skip the tests
	const skipTestsPressed = async() => {
		testsSkipped = true;
	};

	// Uses the given url to create a new HLS stream (or image) and place it in the video grid container
	function addStream(url, camera){
		var container = document.getElementById('video-container');
		var template = document.createElement('template');
		template.innerHTML = makeTemplateHTML(url, camera.location).trim();

		var clone = template.content.cloneNode(true);
		container.appendChild(clone);
		var video = document.querySelector(`[data-id="${url}"]`);
		var main = video.parentElement;
		var header = main.querySelector('.stream-header');
		var closeButton = header.querySelectorAll('div')[1].querySelector('button');

		if (url.endsWith("jpg")) {
			updateImageStream(video);
		} else {
			var hls = new Hls();
			hls.loadSource(url);
			hls.attachMedia(video);
		};

		video.parentElement.style.setProperty("width", document.getElementById('widthSlider').value + "%");
		header.style.setProperty('display', 'none', 'important');

		main.addEventListener('mouseenter', function() {
			header.style.setProperty('display', 'flex', 'important');
		});

		main.addEventListener('mouseleave', function() {
			header.style.setProperty('display', 'none', 'important');
		});

		closeButton.addEventListener('click', function() {
			openStreams = openStreams.filter(m => m !== url);
			openUnitIds = openUnitIds.filter(m => m !== camera.id);
			cameraMarkers[camera.id].setStyle({fillColor: '#438cc1'});
			cameraMarkers[camera.id].setStyle({color: '#eaeaea'});
			cameraMarkers[camera.id].setStyle({weight: 1});

			window.location.hash = "#" + openUnitIds.join();
			video.parentElement.remove();
		});
	}

	// Updates an existing image camera (WSDOT) by changing the query string to include the current timestamp
	function updateImageStream(video){
		video.src = video.dataset.id + "?" + new Date().getTime();
	}

	// Show/hide map window
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

	// Clear all currently open streams
	function clearAllButton(){
		openStreams = [];
		openUnitIds = [];

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

	// Adjust the width of all video streams (based on percentage of window size from 0 to 100)
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

	// Play all video streams and adjust their currentTime to 10 seconds less than their duration (SDOT streams update every 10 seconds)
	function playAllButton(){
		for (let video of document.querySelectorAll("video")) {
			video.play();
			video.currentTime = video.duration - 10;
		};
	}

	// Initial startup tasks (ran before the map and camera list are fully loaded)
	onMount(async () => {
		var search = document.getElementById('search');
		var slider = document.getElementById('widthSlider');

		search.addEventListener('submit', e => {
			e.preventDefault();
		});

		// Load existing map visibility and video width parameters from the query string, if available
		var url = new URLSearchParams(window.location.search);

		var mapOpen = url.get('map');
		if (mapOpen === 'false') {
			testsSkipped = true;
			warningMessage = "Camera stream testing was skipped because the site launched with the map hidden. Some video streams that are offline/broken may be visible on the map! To automatically filter out broken video streams (to an extent), ensure the map is visible and reload the page. Depending on your internet connection, it may take a while.";
			mapButton();
		};

		var videoWidth = url.get('videoWidth');
		if (videoWidth != null) {
			slider.value = videoWidth;
			widthUpdate();
		};
	});

	// Update all camera images (WSDOT) every 5 seconds
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
		{#if warningMessage.length > 0}
			<button type="button" class="btn btn-danger ms-2 my-2" style="height: 38px;font-size:1.5em;" id="warning" on:click={()=>{warningMessage = ""}}><Fa icon={faWarning} /></button>
			<Tooltip target="warning" bottom>{warningMessage}</Tooltip>
		{/if}
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

	:global(.circle-group) {
		background-color: #2f5899;
		border: 1px solid #eaeaea;
		border-radius:50%;
		-moz-border-radius:50%;
	}

	:global(.flex-grow) {
		height: 100%;
	}

	:global(video) {
		width: 100% !important;
		height: 100% !important;
	}

	:global(.video-card) {
		background: black !important;
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

	:global(.stream-header) {
		position:absolute;
		left:0;
		right:0;
		background: rgba(0,0,0,0.8);
		z-index: 99;
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