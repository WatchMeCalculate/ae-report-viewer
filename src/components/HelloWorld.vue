<template>
  <v-form v-if="showForm">
    <v-textarea
      name="fleetReport"
      v-model="fleetReport"
      :rules="fleetRules"
      label="fleet report"
      required
      multi-line
    ></v-textarea>
    <v-textarea
      name="baseReport"
      v-model="baseReport"
      :rules="baseRules"
      label="base report"
      required
      multi-line
    ></v-textarea>
    <v-btn class="mr-4" @click="submit">submit</v-btn>
  </v-form>
  <div v-else>
    <v-btn class="mr-4" @click="reset">Reset</v-btn>
    <v-card>
      <v-card-title>
        Bases
        <v-spacer></v-spacer>
        <!--<v-text-field
          v-model="search"
          label="Search"
          single-line
          hide-details
          ></v-text-field> -->
      </v-card-title>
      <v-data-table :headers="headers" :items="processedLocations" :items-per-page="30">
        <template v-slot:item.location="{ item }">
          <a :href="getLink(item.location)">{{ item.location }}</a>
        </template>
        <template v-slot:item.fleets="{ item }">
          {{ formatFleet(item) }}
        </template>
        <template v-slot:item.hasOwnerFleet="{ item }">
          {{ item.nonOwnerFleet() }}
        </template>
        <template v-slot:body.append>
          <tr>
            <td></td>
            <td></td>
            <td>
              <v-text-field v-model="names" type="text" label="other than"></v-text-field>
            </td>
            <td>
              <v-text-field
                v-model="fleetsize"
                type="number"
                label="fleet larger than"
              ></v-text-field>
              <!-- <v-text-field
                v-model="fleetplayer"
                type="text"
                label="fleet owner other than"
                ></v-text-field> -->
            </td>
            <td colspan="4"></td>
          </tr>
        </template>
      </v-data-table>
    </v-card>
  </div>
</template>

<script>
import { validationMixin } from 'vuelidate';
import { required } from 'vuelidate/lib/validators';

class Fleet {
  constructor(player, location, size, lastSeen) {
    this.player = player;
    this.location = location;
    this.size = parseInt(size, 10);
    this.lastSeen = lastSeen;
  }
}
class Location {
  constructor(location) {
    this.location = location;
    this.has_base = false;
    this.base_owner = null;
    this.fleets = [];
  }

  nonOwnerFleet() {
    return this.fleets.filter(f => this.base_owner !== f.player).length > 0;
  }
}

export default {
  mixins: [validationMixin],
  name: 'HelloWorld',

  validators: {
    fleetReport: { required },
    baseReport: { required },
  },

  data: () => ({
    search: '',
    fleetReport: '',
    baseReport: '',
    names: '',
    fleetsize: '',
    fleetplayer: '',
    showForm: true,
    fleetRules: [v => !!v || 'Fleets Required'],
    baseRules: [v => !!v || 'Bases Required'],
    processedFleets: [],
    processedBase: [],
    processedLocations: [],
  }),
  methods: {
    submit() {
      if (this.fleetReport && this.baseReport) {
        this.showForm = false;
        this.processedFleets = this.fleetReport
          .split('\n')
          .slice(4)
          .map(f => f.split('\t'))
          .filter(Boolean);
        this.processedBase = this.baseReport
          .split('\n')
          .slice(3)
          .map(f => f.split('\t'))
          .filter(Boolean);
        this.processData();
      }
    },
    getLink(location) {
      return `https://mystic.astroempires.com/map.aspx?loc=${location}`;
    },
    reset() {
      this.fleetReport = '';
      this.baseReport = '';
      this.processedFleets = [];
      this.processedBase = [];
      this.showForm = true;
    },
    processData() {
      const locations = {};
      /* eslint-disable-next-line */
      for (const base of this.processedBase) {
        const newLoc = new Location(base[2]);
        newLoc.has_base = true;
        /* eslint-disable-next-line */
        newLoc.base_owner = base[0];
        locations[newLoc.location] = newLoc;
      }
      /* eslint-disable-next-line */
      for (const f of this.processedFleets) {
        const fleet = new Fleet(f[0], f[1], f[2], f[3]);
        if (!(fleet.location in locations)) {
          const newLoc = new Location(fleet.location);
          locations[newLoc.location] = newLoc;
        }
        locations[fleet.location].fleets.push(fleet);
      }
      this.processedLocations = Object.values(locations);
    },
    formatFleet(loc) {
      const fleets = [];
      /* eslint-disable-next-line */
      for (const f of loc.fleets) {
        fleets.push({ player: f.player, size: f.size });
      }
      return fleets;
    },
  },
  computed: {
    headers() {
      return [
        { text: 'Location', value: 'location' },
        { text: 'Has Base', value: 'has_base' },
        {
          text: 'Base Owner',
          value: 'base_owner',
          filter: (value) => {
            if (!this.names || !value) return true;
            /* eslint-disable-next-line */
            for (const name of this.names.split(",")) {
              if (value.includes(name)) {
                return false;
              }
            }
            return true;
          },
        },
        {
          text: 'Fleets',
          value: 'fleets',
          filter: (value) => {
            let sizeB = false;
            let playerB = false;
            if (!value) return true;
            if (!this.fleetsize) {
              sizeB = true;
            } else {
              sizeB = value.filter(f => f.size >= this.fleetsize).length > 0;
            }
            if (!this.fleetplayer) {
              playerB = true;
            } else {
              /* eslint-disable-next-line */
              for (const name of this.fleetplayer.split(",")) {
                /* eslint-disable-next-line */
                playerB = playerB && value.filter(f => f.player.includes(name)).length > 0;
              }
            }
            return sizeB && playerB;
          },
        },

        { text: 'Has Non Owner Fleet', value: 'hasOwnerFleet' },
      ];
    },
  },
};
</script>
