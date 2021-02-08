<template>
	<div class="wrapper">
		<link rel="stylesheet" href="https://openlayers.org/en/latest/css/ol.css" type="text/css">
		<v-app-bar
			dense
			color="blue-grey lighten-4"
		>
			<v-spacer></v-spacer>
				<v-tooltip bottom>
					<template v-slot:activator="{ on, attrs }">
						<v-btn 
							color="primary"
							icon
							v-on:click="sync()"
							v-on="on"
							v-bind="attrs"
						>	
							<v-icon>mdi-database-refresh</v-icon>
						</v-btn>
					</template>
					<span>Sync Database</span>
				</v-tooltip>
				<v-tooltip bottom>
					<template v-slot:activator="{ on, attrs }">
						<v-btn 
							color="primary"
							icon
							v-on:click="mapVisible()" 
							v-on="on"
							v-bind="attrs"
						>				
							<v-icon>mdi-card-search-outline</v-icon>
						</v-btn>
					</template>
					<span>Map Visible Dots</span>
				</v-tooltip>						
				<v-tooltip bottom>
					<template v-slot:activator="{ on, attrs }">
						<v-btn 
							color="primary"
							icon
							v-on:click="$emit('login-change', false)" 
							v-on="on"
							v-bind="attrs"
						>					
							<v-icon>mdi-logout-variant</v-icon>
						</v-btn>
					</template>
					<span>Logout</span>
				</v-tooltip>						
		</v-app-bar>
		<div id="map"></div>	
		<div ref="popup" v-show="false">Hi there!</div>
		<v-snackbar v-if="message.length">{{ message }}</v-snackbar>		
		<loading v-if="loading"></loading>
	</div>
</template>

<script lang="ts">
	import Vue from 'vue';
	import Loading from "./Loading.vue";

	import * as geobuf from 'geobuf';
	import Pbf from 'pbf';

	// Try out OpenLayers
	import Map from 'ol/Map';
	import View from 'ol/View';
	import TileLayer from 'ol/layer/Tile';
	import {getBottomLeft, getBottomRight, getTopLeft, getTopRight} from 'ol/extent';
	import {defaults} from 'ol/interaction';
	import {toLonLat} from 'ol/proj';
	import WebGLPointsLayer from 'ol/layer/WebGLPoints';
	import GeoJSON from 'ol/format/GeoJSON';
	import Vector from 'ol/source/Vector';
	import Geolocation from 'ol/Geolocation';
	import VectorLayer from 'ol/layer/Vector';
	import { SymbolType } from 'ol/style/LiteralStyle';
	import Stamen from "ol/source/Stamen";
	
	export default Vue.extend({
		name: 'Main',
		components: { Loading },
		props: {
			user: String,
		},
		data()  { 
			return {
				map: {} as Map,
				message: "" as string,
				loading: false as boolean
			}
		},
		methods: {
			sync: async function() {
				try {
					this.loading = true;
					const results = await fetch(`${process.env.VUE_APP_API_URL}/locations/sync/strava`, {credentials: 'include'});
					await results.json();
					this.loading = false;
					this.message = 'Successfully synced!';
				} catch (e) {
					this.message = 'Error syncing';
				}
			},

			mapVisible: async function() {	
				const extents = this.map.getView().calculateExtent();
				const topLeft = toLonLat(getTopLeft(extents));
				const bottomRight = toLonLat(getBottomRight(extents));
				await this.loadPoints(`${process.env.VUE_APP_API_URL}/locations/strava/points/${topLeft[0]},${topLeft[1]},${bottomRight[0]},${bottomRight[1]}`);
			},	

			loadPoints: async function(location: string) {

				this.loading = true;
				console.log('downloading');
				const response = await fetch(location, {credentials: 'include'});

				console.log('decoding');
				const decoded = geobuf.decode(new Pbf(await response.arrayBuffer()));
				console.log(decoded);
				console.log('creating geojson')
				const vectorSource = new Vector({
					features: new GeoJSON().readFeatures(decoded, { featureProjection: 'EPSG:3857' })
				});

				console.log('creating points layer')
				const webgl = new WebGLPointsLayer({
					source: vectorSource,
					style: {
						symbol: {
							symbolType: SymbolType.CIRCLE,
							size: 5,
							color: '#006688'
						}
					}

				});
				this.map.addLayer(webgl);

				console.log('rendering');
				this.map.renderSync();
				console.log('done');
				this.loading = false;
			}
		},
		mounted() {
			const view: View = new View({
					projection: 'EPSG:3857',
					center: [0, 0],
					zoom: 13
				})
			this.map = new Map({
				target: 'map',
				interactions: defaults({dragPan: true, mouseWheelZoom: true}),				
				layers: [
					new TileLayer({
						source: new Stamen({
							layer: "terrain"
						})
					})
				],
				view
			});	

			const geolocation = new Geolocation({
				trackingOptions: {
					enableHighAccuracy: true
				},
				projection: view.getProjection(),
				tracking: true
			});

			geolocation.once('change', () => {
				view.setCenter(geolocation.getPosition());
			});
		}
	})
</script>

<style lang="scss">
	#map {
		height: 100vh;
		width: 100vw;
		z-index: 1;
	}
	.wrapper {
		height: 100vh;
		width: 100vw;		
		overflow: hidden;
	}
</style>