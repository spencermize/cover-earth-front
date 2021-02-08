  
<template>
  <v-app>
    <v-main>
      <v-container
        class="fill-height"
        fluid
      >
        <v-row
          align="center"
          justify="center"
        >
          <v-col
            cols="12"
            sm="8"
            md="4"
          >
            <v-card class="elevation-12">
              <v-toolbar
                color="primary"
                dark
                flat
              >
                <v-toolbar-title>Login</v-toolbar-title>
                <v-spacer />
              </v-toolbar>
              <v-card-text>
				<v-row>
					<v-col sm="6">
						<a :href="env.VUE_APP_AUTH_URL + '/strava'"><v-img v-ripple class="elevation-1" src="/img/btn_strava_connectwith_light.svg"/></a>
					</v-col>
				</v-row>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<style lang="scss">

</style>
<script lang="ts">
	import Vue from 'vue';
	export default Vue.extend({
    name: 'Login',
    data: () => {
      return {
        env: process.env
      }
    }

	});
</script>