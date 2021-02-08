<template>
  <v-app>
    <v-main>
		<router-view v-on:login-change="onLoginChange" v-bind:login="login"></router-view>
    </v-main>
  </v-app>
</template>

<script lang="ts">
import Vue from "vue";

export default Vue.extend({
	name: "App",
	data: () => {
		return {
			login: false as boolean,
		}
	},
	methods: {
		onLoginChange: async function(login: boolean) {
			const request = await fetch(`${process.env.VUE_APP_AUTH_URL}/logout`);
			const json = await request.json();
			if ( json.success ) {
				this.$router.replace('/login');
			}
		},
		
	}
});
</script>
