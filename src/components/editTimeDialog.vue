<template>
  <v-row>
    <v-col cols="11" sm="5">
      <v-menu
        ref="menu"
        v-model="menuTime"
        :close-on-content-click="false"
        :nudge-right="40"
        :return-value.sync="time"
        transition="scale-transition"
        offset-y
        max-width="290px"
        min-width="290px"
      >
        <template v-slot:activator="{ on }">
          <v-text-field
            v-model="time"
            label="Escolha o horário"
            prepend-icon="mdi-access_time"
            readonly
            v-on="on"
          ></v-text-field>
        </template>
        <v-time-picker
          v-if="menuTime"
          v-model="time"
          format='24hr'
          scrollable
          :allowed-hours='allowedHours'
          :allowed-minutes='allowedMinutes'
          full-width
          @click:minute="$refs.menu.save(time)"
        ></v-time-picker>
      </v-menu>
    </v-col>
  </v-row>
</template>

<script>
  export default {
    data () {
      return {
        time: null,
        menuTime: false,
        allowedHours: [8,9,10,11,13,14,15,16,17,18],
        allowedMinutes: [0,30]
      }
    },
  }
</script>