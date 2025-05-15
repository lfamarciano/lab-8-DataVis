<script>
    import mapboxgl from "mapbox-gl";
    import * as d3 from "d3";
    import { onMount } from "svelte";

    mapboxgl.accessToken = "pk.eyJ1IjoibGZhbWFyY2lhbm8iLCJhIjoiY21hcGk0Zm5nMGcwODJsbzhuZjgwYnh2OCJ9.lGgX7SSTrs-bvlA5xpLbJg";

    let map;

    let departures = new Map();
    let arrivals = new Map();

    let stations = [];
    let trips = [];

    async function initMap() {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09415, 42.36027],
            zoom: 12,
            style: "mapbox://styles/mapbox/streets-v12",
        });

        await new Promise(resolve => map.on("load", resolve));

        // Fonte de Boston
        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        // bike lanes Boston
        map.addLayer({
            id: "boston_bike_lanes",
            type: "line",
            source: "boston_route",
            paint: {
                "line-color": "rgba(255, 0, 0, 0.5)", // vermelho
                "line-width": 3,
                "line-opacity": 0.9,
            },
        });

        // Fonte de Cambridge
        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        // bike lanes Cambridge
        map.addLayer({
            id: "cambridge_bike_lanes",
            type: "line",
            source: "cambridge_route",
            paint: {
                "line-color": "rgba(0, 0, 255, 0.5)",  // azul
                "line-width": 3,
                "line-opacity": 0.9,
            },
        });
    }

    async function loadStationData() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            const data = await d3.csv(csvUrl);

            stations = data.map(station => ({
                id: station.Number,
                name: station.NAME,
                Lat: +station.Lat,
                Long: +station.Long,
            }));

            // força atualização visual após carregar estações
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    async function loadStationDemand() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            const data = await d3.csv(csvUrl);
            
            trips = data.map(trip => {
                return {
                    id: trip.ride_id,
                    name: trip.NAME,
                    started_at: new Date(trip.started_at),
                    ended_at: new Date(trip.ended_at),
                    start_station_id: trip.start_station_id,
                    end_station_id: trip.end_station_id
                };
            });
            
            // console.log('Trips loaded:', trips.length);
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    let mapViewChanged = 0;
    $: map?.on("move", evt => mapViewChanged++);
    
    function updateTraffic() {
        if (!stations.length || !trips.length) return;

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.id;
            station.arrivals = arrivals.get(id) || 0;
            station.departures = departures.get(id) || 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    }

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic) || 1]) 
        .range([0, 25]);

    onMount(async () => {
        await initMap();
        await loadStationData();
        await loadStationDemand();

        updateTraffic();

        map.on("move", () => {
            // dispara reatividade forçando redraw dos círculos
            stations = stations.map(s => s);
        });
    });

    function getCoords(station) {
        if (!map) return { cx: 0, cy: 0 };
        let point = new mapboxgl.LngLat(station.Long, station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    let timeFilter = -1;
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});
</script>

<svelte:head>
  <title>Luís Felipe Marciano</title>
</svelte:head>

<header>
    <h1>BikeWatch</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter !== -1}
            <time style="display: block">
                {timeFilterLabel}
            </time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
</header>

<div id="map" style="position: relative; width: 100%; height: 800px;">
    {#key mapViewChanged}
        <svg 
        style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none;">
        {#each stations as station (station.id)}
        <circle 
            style="fill-opacity: 0.6; stroke: white;" 
            fill="darkgreen"
            cx={getCoords(station).cx} 
            cy={getCoords(station).cy} 
            r={radiusScale(station.totalTraffic)} 
        />
        {/each}
        </svg>
    {/key}
</div>