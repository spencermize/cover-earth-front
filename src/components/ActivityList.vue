<template>
		<v-dialog
			v-model="activities"
			width="500"
			scrollable
			>
			<v-card>
                <v-card-title class="headline grey lighten-2">
                    Hex ID: {{ hex.id }}
                </v-card-title>
				<v-list>
					<v-list-item v-for="activity in sortedActivities" :key="activity.id">
                        <v-list-item-icon>
                            <v-icon v-text="getActivityIcon(activity)"></v-icon>
                        </v-list-item-icon>
						<v-list-item-content>
                            <v-list-item-title v-text="activity.name"></v-list-item-title>
                            <v-list-item-subtitle v-text="getDistance(activity) + ' ' + units"></v-list-item-subtitle>
                        </v-list-item-content>
                        <v-list-item-action>
                            <v-list-item-action-text v-text="getDate(activity)"></v-list-item-action-text>
                        </v-list-item-action>
					</v-list-item>
				</v-list>
				
			</v-card>
		</v-dialog>
</template>

<script lang="ts">
	import { ActivityMeta, ActivityType } from '@/typings/strava';
    import Vue from 'vue';
    import format from 'date-fns/format';
    import * as turf from '@turf/turf';

    enum DistanceUnit {
        Miles = 'miles',
        Meters = 'meters',
        Millimeters = 'millimeters',
        Centimeters = 'centimeters',
        Kilometers = 'kilometers',
        Acres = 'acres',
        NauticalMiles = 'nauticalmiles',
        Inches = 'inches',
        Yards = 'yards',
        Feet = 'feet',
        Radians = 'radians',
        Degrees = 'degrees',
        Hectares = 'hectares'
    }
    export default Vue.extend({
            name: 'ActivityList',
            props: {
                initialActivities: Object,
                hex: Object
            },
            data: function() {
                return {
                    activities: this.initialActivities as ActivityMeta || {},
                    units: DistanceUnit.Miles
                }
            },
            methods: {
                getActivityIcon(activity: ActivityMeta) {
                    switch (activity.type) {
                        case ActivityType.Run :
                            return 'mdi-run';
                        case ActivityType.Ride :
                            return 'mdi-bike';
                        case ActivityType.Hike : 
                            return 'mdi-pine-tree';
                        case ActivityType.Walk :
                            return 'mdi-walk';
                    }
                },
                getDate(activity: ActivityMeta) {
                    return format(new Date(activity.start_date_local), 'MM/dd/yyyy');
                },
                getDistance(activity: ActivityMeta) {
                    return activity.distance ? turf.convertLength(activity.distance, 'meters', this.units).toFixed(2) : 0;
                }
            },

            computed: {
                sortedActivities() {
                    const sortable: ActivityMeta[] = Object.values(this.activities);

                    return sortable.sort((a, b) => {
                        return new Date(a.start_date_local) > new Date(b.start_date_local) ? -1 : 1;
                    });
                }
            }
    });
</script>