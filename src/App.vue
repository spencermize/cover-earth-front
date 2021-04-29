<template>
  <v-app>
    <v-main>
		<router-view v-on:login-change="onLoginChange" v-if="!error" v-bind:login="login"></router-view>
		<v-alert 
			v-if="error"
			color="red"
			type="error"
			prominent
			elevation="15"
		>
			Sorry, something major has gone wrong. Good luck!
		</v-alert>
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
			error: false as boolean
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
