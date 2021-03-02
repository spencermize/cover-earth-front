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
							<v-icon>mdi-dots-square</v-icon>
						</v-btn>
					</template>
					<span>Map Visible Dots</span>
				</v-tooltip>	
				<v-tooltip bottom>
					<template v-slot:activator="{ on, attrs }">
						<v-btn 
							color="primary"
							icon
							v-on:click="mapPaths()" 
							v-on="on"
							v-bind="attrs"
						>				
							<v-icon>mdi-map-marker-path</v-icon>
						</v-btn>
					</template>
					<span>Show All Paths</span>
				</v-tooltip>	
				<v-tooltip bottom>
					<template v-slot:activator="{ on, attrs }">
						<v-btn 
							color="primary"
							icon
							v-on:click="drawGrid()" 
							v-on="on"
							v-bind="attrs"
						>				
							<v-icon>mdi-grid</v-icon>
						</v-btn>
					</template>
					<span>Draw Grid</span>
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
		<activity-list v-if="selectedTile && selectedTile.getProperties()" :hex="selectedTile" :initial-activities="selectedTile.getProperties().activityMeta"></activity-list>
	</div>
</template>

<script lang="ts">
	import Vue from 'vue';
	import Loading from "./Loading.vue";
	import ActivityList from '@/components/ActivityList.vue';

	import { ActivityMeta } from '@/typings';

	import * as geobuf from 'geobuf';
	import Pbf from 'pbf';

	import Map from 'ol/Map';
	import View from 'ol/View';
	import TileLayer from 'ol/layer/Tile';
	import {getBottomRight, getTopLeft} from 'ol/extent';
	import {defaults} from 'ol/interaction';
	import {fromLonLat, toLonLat} from 'ol/proj';
	import WebGLPointsLayer from 'ol/layer/WebGLPoints';
	import GeoJSON from 'ol/format/GeoJSON';
	import Vector from 'ol/source/Vector';
	import Geolocation from 'ol/Geolocation';
	import VectorLayer from 'ol/layer/Vector';
	import { SymbolType } from 'ol/style/LiteralStyle';
	import Stamen from "ol/source/Stamen";

	import * as turf from '@turf/turf';
	import Geometry from 'ol/geom/Geometry';
	import Feature from 'ol/Feature';
	import { FeatureCollection, Polygon, Properties } from '@turf/turf';
	import { Coordinate } from 'ol/coordinate';
	import Style from 'ol/style/Style';
	import Stroke from 'ol/style/Stroke';
	import Polyline from 'ol/format/Polyline';
	
	import colors from 'vuetify/lib/util/colors'
	import LineString from 'ol/geom/LineString';

	import * as geohash from 'ngeohash';
	import Fill from 'ol/style/Fill';


	export default Vue.extend({
		name: 'Main',
		components: { Loading, ActivityList },
		props: {
			user: String,
		},
		data()  { 
			return {
				map: {} as Map,
				message: "" as string,
				loading: false as boolean,
				hoverTile: new Feature() as Feature<Geometry>,
				selectedTile: null as Feature<Geometry>|null,
				visitedGrid: {} as Record<string, Feature>,
				grid: turf.featureCollection([]) as FeatureCollection<Polygon, Properties>
			}
		},
		methods: {
			sync: async function() {
				try {
					this.loading = true;
					const results = await fetch(`${process.env.VUE_APP_API_URL}/locations/sync/strava`, {credentials: 'include'});
					const json = await results.json();
					this.loading = false;
					if (!json.success) {
						throw 'Error syncing'
					}
					this.message = 'Successfully synced!';
				} catch (e) {
					this.message = e;
				}
			},

			drawGrid: async function() {
				this.loading = true;
				const response = await fetch(`${process.env.VUE_APP_API_URL}/locations/grid/${this.getBBoxString()}`, {credentials: 'include'});
				const decoded = geobuf.decode(new Pbf(await response.arrayBuffer()));
				const source = new Vector({
					features: new GeoJSON().readFeatures(decoded, { featureProjection: 'EPSG:3857' })
				});

				const vector = new VectorLayer({
					source,
				});
				this.map.addLayer(vector);
				this.loading = false;
			},

			mapVisible: async function() {	
				await this.loadPoints(`${process.env.VUE_APP_API_URL}/locations/strava/points/${this.getBBoxString()}`);
			},	

			mapPaths: async function() {
				this.loading = true;
				const response = await fetch(`${process.env.VUE_APP_API_URL}/locations/strava/activities/`, {credentials: 'include'});
				const activities = await response.json() as { id: string, meta: ActivityMeta, location?: [] }[];
				const features = [];

				for(const activity of activities) {
					if (activity.meta) {
						const geo = new Polyline().readFeature(activity.meta.map.summary_polyline, {
							featureProjection: 'EPSG:3857'
						});

						const lineString = geo.getGeometry() as LineString;
						for(let coordinate of lineString.getCoordinates()) {
							const coord = this.gridToGps(this.getNearestCenter(this.gpsToGrid(toLonLat(coordinate))));
							const hash = geohash.encode(coord[0], coord[1]);
							if (!this.visitedGrid[hash]) {
								this.visitedGrid[hash] = this.getClosestHex(coordinate);
								this.visitedGrid[hash].setProperties({
									activityMeta: {}
								})
							}

							if (!this.visitedGrid[hash].getProperties().activityMeta[activity.id.toString()]) {
								const meta = this.visitedGrid[hash].getProperties().activityMeta;
								meta[activity.id.toString()] = activity.meta;
								this.visitedGrid[hash].setProperties({
									activityMeta: meta
								})
							}
						}

						features.push(geo);
					}			
				}

				const source = new Vector({
					features
				});

				const vector = new VectorLayer({
					source,
					opacity: .5,
					style: new Style({
						stroke: new Stroke({
							width: 4,
							color: colors.pink.darken2
						})
					})
				});
				
				this.map.addLayer(vector);

				const hexSource = new Vector({
					features: Object.values(this.visitedGrid)
				});

				const hexVector = new VectorLayer({
					source: hexSource,
					opacity: .75,
					style: new Style({
						stroke: new Stroke({
							width: 2,
							color: colors.red.base
						}),
						fill: new Fill({
							color: colors.red.base,  // TODO: doesn't work?
						})
					})
				});

				this.map.addLayer(hexVector);
				this.loading = false;

			},

			getBBox: function() : [number, number, number, number] {
				const extents = this.map.getView().calculateExtent();
				const topLeft = toLonLat(getTopLeft(extents));
				const bottomRight = toLonLat(getBottomRight(extents));

				return [topLeft[0], topLeft[1], bottomRight[0], bottomRight[1]];
			},

			getBBoxString: function() {
				const extents = this.map.getView().calculateExtent();
				const topLeft = toLonLat(getTopLeft(extents));
				const bottomRight = toLonLat(getBottomRight(extents));
				return `${topLeft[0]},${topLeft[1]},${bottomRight[0]},${bottomRight[1]}`;
			},

			loadPoints: async function(location: string) {

				this.loading = true;
				console.log('downloading');
				const response = await fetch(location, {credentials: 'include'});

				console.log('decoding');
				const decoded = geobuf.decode(new Pbf(await response.arrayBuffer()));

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
				this.map.render();
				console.log('done');
				this.loading = false;
			},
			generateHex: function() {
				const mask = turf.bboxPolygon(this.getBBox());
				const size = 10;
				const southLimit = -55;
				const northLimit = 78;
				return turf.featureCollection([
					...turf.hexGrid([-180, southLimit, 0, northLimit], size, {
						mask
					}).features,
					...turf.hexGrid([0, southLimit, 180, northLimit], size, {
						mask
					}).features
				]);
			},

			generateSingleHex: function(center: Coordinate) {
				const size = .01;
				const coords = [];
				for(let i = 1; i < 7; i++) {
					const angleDeg = 60 * i - 30;
					const angleRad = Math.PI / 180 * angleDeg;
					coords.push([center[0] + size * Math.cos(angleRad), center[1] + size * Math.sin(angleRad)]);
				}
				coords.push(coords[0]);
				return turf.feature(turf.geometry("LineString", coords));
			},

			gpsToGrid(coord: Coordinate) {
				const sigDigits = 4;
				const x = Math.round(this.shift(turf.round(coord[0], sigDigits) + 180, 1 - sigDigits));
				const y = Math.round(this.shift(turf.round(coord[1], sigDigits) + 90, 1 - sigDigits));

				const retValue = [x, y];
				return retValue;
			},

			gridToGps(coord: Coordinate) {
				const sigDigits = 4;
				const x = this.shift(coord[0], sigDigits - 1) - 180;
				const y = this.shift(coord[1], sigDigits - 1) - 90;

				const retValue = [x, y];
				return retValue;
			},

			shift(num: number, sigDigits: number) {
				return num / Math.pow(10, sigDigits);
			},

			getNearestCenter(coord: Coordinate): Coordinate {
				const precisionX = 17.5; // TODO: generate these formulaicly ... ?!
				const precisionY = 15;

				coord[1] = Math.round(coord[1] / precisionY ) * precisionY;
				const offset = coord[1] % 2 === 0 ? precisionX / 2 : 0;
				coord[0] = (Math.round(coord[0] / precisionX ) * precisionX) + offset;
				return coord;
			},

			getClosestHex(coord: Coordinate) {
				const center = this.gridToGps(this.getNearestCenter(this.gpsToGrid(toLonLat(coord))));
				return new GeoJSON().readFeature(this.generateSingleHex(center), { featureProjection: 'EPSG:3857' });
			},

			getVisitedHex(coordinate: Coordinate) {
				const coord = this.gridToGps(this.getNearestCenter(this.gpsToGrid(toLonLat(coordinate))));
				const hash = geohash.encode(coord[0], coord[1]);

				return this.visitedGrid[hash] || null;
			}
		},
		mounted() {
			const view: View = new View({
					projection: 'EPSG:3857',
					zoom: 13,
					center: fromLonLat([-85.4812, 41.1668])
				});
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

			geolocation.on('change', () => {
				view.setCenter(geolocation.getPosition());
			});

			const hoverSource = new Vector();
			const hoverVector = new VectorLayer({
				source: hoverSource,
				style: new Style({
					stroke: new Stroke({
						width: 5,
						color: colors.pink.darken2
					})
				})
			});

			const selectedSource = new Vector();
			const selectedVector = new VectorLayer({
				source: selectedSource,
				style: new Style({
					stroke: new Stroke({
						color: 'red'
					})
				})
			});
			this.map.addLayer(hoverVector);
			this.map.addLayer(selectedVector);
			// this.grid = this.generateHex();
			this.map.on("pointermove", (e) => {
				hoverSource.clear();
				this.hoverTile = this.getClosestHex(e.coordinate);
				hoverSource.addFeature(this.hoverTile);
			});

			this.map.on("click", (e) => {
				this.selectedTile = this.getVisitedHex(e.coordinate) || this.getClosestHex(e.coordinate);
				selectedSource.clear();
				selectedSource.addFeature(this.selectedTile);
				console.log(this.selectedTile.getProperties().activityMeta);
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
		cursor: pointer;
	}
</style>