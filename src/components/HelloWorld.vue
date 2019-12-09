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
      <v-data-table :headers="headers" :items="processedLocations" :items-per-page="30" dense>
        <template v-slot:item.location="{ item }">
          <a :href="getLink(item.location)">{{ item.location }}</a>
        </template>
        <template v-slot:item.fleets="{ item }">
          {{ formatFleet(item) }}
        </template>
        <template v-slot:body.prepend>
          <tr>
            <td>
              <v-text-field
                v-model="location_search"
                type="text"
                label="Match Location"
              ></v-text-field>
            </td>
            <td>
              <v-text-field v-model="has_base_search" type="text" label="has base"></v-text-field>
            </td>
            <td>
              <v-text-field
                v-model="base_owner_search"
                type="text"
                label="other than"
              ></v-text-field>
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
            <td>
              <v-text-field
                v-model="non_owner_fleet_search"
                type="text"
                label="non owner fleet"
              ></v-text-field>
            </td>
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
    this.size = parseInt(size.replace(',', ''), 10);
    this.lastSeen = lastSeen;
  }
}
class Location {
  constructor(location) {
    this.location = location;
    this.has_base = false;
    this.has_non_owner_fleet = false;
    this.base_owner = null;
    this.fleets = [];
  }

  nonOwnerFleet() {
    this.has_non_owner_fleet = this.fleets.filter(f => this.base_owner !== f.player).length > 0;
  }
}

export default {
  mixins: [validationMixin],
  name: 'HelloWorld',

  validators: {
    fleetReport: { required },
    baseReport: { required },
  },
  created() {
    this.fleetReport = localStorage.getItem('fleet_report');
    this.baseReport = localStorage.getItem('base_report');
    if (this.fleetReport && this.baseReport) {
      this.submit();
    }
  },

  data: () => ({
    search: '',
    fleetReport: '',
    baseReport: '',
    base_owner_search: '',
    location_search: '',
    has_base_search: '',
    non_owner_fleet_search: '',
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
        localStorage.setItem('fleet_report', this.fleetReport);
        localStorage.setItem('base_report', this.baseReport);
        this.processedFleets = this.fleetReport
          .split('\n')
          .slice(4)
          .map(f => f.split('\t'))
          .filter(f => f[0] !== '');
        this.processedBase = this.baseReport
          .split('\n')
          .slice(3)
          .map(f => f.split('\t'))
          .filter(f => f[0] !== '');
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
      localStorage.removeItem('fleet_report');
      localStorage.removeItem('base_report');
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
        locations[fleet.location].nonOwnerFleet();
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
        {
          text: 'Location',
          value: 'location',
          filter: (value) => {
            if (!this.location_search || !value) return true;
            /* eslint-disable-next-line */
            if (value.includes(this.location_search)) {
              return true;
            }
            return false;
          },
        },
        {
          text: 'Has Base',
          value: 'has_base',
          filter: (value) => {
            if (!this.has_base_search) return true;
            /* eslint-disable-next-line */
            if (value === (this.has_base_search.toLowerCase() === "true")) {
              return true;
            }
            return false;
          },
        },
        {
          text: 'Base Owner',
          value: 'base_owner',
          filter: (value) => {
            if (!this.base_owner_search || !value) return true;
            /* eslint-disable-next-line */
            for (const name of this.base_owner_search.split(",")) {
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

        {
          text: 'Has Non Owner Fleet',
          value: 'has_non_owner_fleet',
          filter: (value) => {
            if (!this.non_owner_fleet_search) return true;
            /* eslint-disable-next-line */
            if (value == (this.non_owner_fleet_search === "true")) {
              return true;
            }
            return false;
          },
        },
      ];
    },
  },
};
</script>
