<template>
  <div style="padding-bottom: 0;">
    <div class="ui fluid icon input input-box" :class="{ loading: loading, error: error, noresults: history.length == 0 }">
      <input id="addr_box" type="text" v-model="address" @keyup.enter="lookup" placeholder="IP address (e.g 1.2.3.4) or host (e.g google.com)" autofocus>
      <i class="circular search link icon" @click="lookup"></i>
    </div>

    <div v-if="error" class="ui negative message"><i class="icon warning sign"></i> {{ error }}</div>

    <div v-if="history.length > 0" class="result-box">
      <a v-if="stats.rate_remaining" data-position="top left" class="ui label" :data-tooltip="stats.rate_remaining + '/' + stats.rate_limit + ' calls remain; resets in ' + stats.rate_reset_seconds + ' sec'" data-inverted>remaining calls <div class="detail"><strong>{{ stats.rate_remaining.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",") }}</strong></div></a>
      <button class="ui right floated mini button" @click="clearHistory"><i class="ui remove circle icon"></i> Clear history</button>

      <transition-group name="list" appear>
        <div v-for="(item, index) in history" class="result" :index="index" :key="item.ip">
          <div class="fluid ui raised card">
            <div class="content">
              <span v-if="item.country_abbr" :data-tooltip="item.country + ' (' + item.country_abbr + ')'" data-inverted class="right floated"><i :class="[item.country_abbr.toLowerCase()]" class="flag"></i></span>

              <div class="header">
                <i class="circle check icon"></i> {{ item.query }}
                <span v-if="item.query != item.ip">
                  (<a data-tooltip="Click to copy" data-inverted="" @click="copyClipboard" :data-clipboard-text="item.ip">{{ item.ip }}</a>)
                </span>
              </div>

              <div class="meta">
                <span class="category" v-if="item.host || item.longitude != 0 && item.latitude != 0">
                  <span v-if="item.host">
                    [host: <a data-tooltip="Click to copy" data-inverted="" @click="copyClipboard" :data-clipboard-text="item.host">{{ item.host }}</a>]
                  </span>
                  <span v-if="item.longitude != 0 && item.latitude != 0" class="right floated">
                    [lat/long: <a :href="'https://www.google.com/maps/@'+item.latitude+','+item.longitude+',5z'" target="_blank">{{item.latitude.toFixed(4)}}, {{item.longitude.toFixed(4)}}</a>]
                  </span>
                </span>
              </div>

              <div class="description" v-if="item.longitude != 0 && item.latitude != 0">
                <v-map style="height: 150px" :options="mapOptions" :zoom=3 v-bind:center="[item.latitude, item.longitude]">
                  <v-tilelayer attribution="&copy; <a href='https://osm.org/copyright'>OpenStreetMap</a> contributors" url="//{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"></v-tilelayer>
                  <v-marker v-bind:lat-lng="[item.latitude, item.longitude]"></v-marker>
                </v-map>
              </div>
            </div>

            <div class="extra content">
              <div v-if="item.timezone" class="ui label"><i class="wait icon"></i> {{item.timezone}}</div>

              <a v-if="item.postal_code" data-tooltip="Postal code - Click to copy" data-inverted="" data-position="bottom center" @click="copyClipboard" :data-clipboard-text="item.postal_code">
                <div class="ui blue label"><i class="building icon"></i> {{item.postal_code}}</div>
              </a>

              <span class="ui right floated">
                <a v-if="item.summary" data-tooltip="Location - Click to copy" data-inverted="" data-position="bottom center" @click="copyClipboard" :data-clipboard-text="item.summary">
                  <div class="ui green label"><i class="map icon"></i> {{item.summary}}</div>
                </a>
              </span>
            </div>
          </div>
        </div>
      </transition-group>
    </div>
  </div>
</template>

<script>
import { utils } from '../utils'
import Vue2Leaflet from 'vue2-leaflet'

export default {
  name: "lookup",
  mixins: [utils],
  components: {
    'v-map': Vue2Leaflet.Map,
    'v-tilelayer' :Vue2Leaflet.TileLayer,
    'v-marker': Vue2Leaflet.Marker
  },
  data: function () {
    return {
      address: "",
      error: false,
      loading: false,
      mapOptions: { scrollWheelZoom: false },
      history: [],
    }
  },
  methods: {
    copyClipboard: (event) => {
      var clipboard = new Clipboard('.null');
      clipboard.onClick(event);
      toastr.success('Copied to clipboard', '', { timeOut: 1000 });
      clipboard.destroy();
    },
    lookup: function (lookupSelf) {
      let query = lookupSelf === true ? 'self' : this.address;
      if (query.length == 0 || this.loading) { return; }

      this.error = false;
      this.loading = true;
      this.$Progress.start();

      // Check to see if we've already looked it up, and it's in history.
      for (var i = 0; i < this.history.length; i++) {
        if (this.history[i].query == query) {
          let result = this.history[i];
          this.history.splice(i, 1);

          // And propagate that change to the URL, so if they copy/paste it,
          // it will pull up for others.
          this.$router.replace({ name: this.name, query: { q: query } });
          this.loading = false;
          this.$Progress.finish();
          this.address = "";
          this.addHistory(result);
          this.$autofocus();

          return;
        }
      }

      this.$lookup(query).then(data => {
        this.loading = false;

        // Propagate that change to the URL, so if they copy/paste it,
        // it will pull up for others.
        this.$router.replace({ name: this.name, query: { q: query } });

        this.$Progress.finish();
        this.address = "";
        this.addHistory(data);
        this.$autofocus();
      }, error => {
        this.loading = false;

        // Indicates that we shouldn't show anything.
        if (error == null) {
          this.$Progress.finish();
          return;
        }

        this.$Progress.fail();
        this.error = error;
      });
    },
    addHistory: function (result) {
        // Make sure we're only storing the last ~10 items.
        if (this.history.length > 10) {
          // this.history = this.history.split(0, 10)
          this.history.length = 10;
        }

        // Add the result to lookup history.
        this.history.unshift(result);

        // And save it to localstorage.
        this.$ls.set("history", JSON.stringify(this.history));
    },
    clearHistory: function () {
      this.history = [];
      this.$ls.set("history", JSON.stringify([]));
        this.$autofocus();
    }
  },
  mounted: function () {
    // On load, try looking in localstorage to see if they have any previous
    // results.
    var history = this.$ls.get("history", "");
    if (history.length > 0) {
      this.history = JSON.parse(history);
    } else {
      this.history = [];
    }

    // If they supplied a request via the URL, use that, otherwise if they
    // have no history, lookup their own IP.
    if (this.$route.query.q !== undefined) {
      this.address = this.$route.query.q;
      this.lookup();
    } else if (this.history.length == 0) {
      this.lookup(true);
    }

    this.$autofocus();
  }
}
</script>

<style scoped>
.input-box {
  transition: 0.15s padding ease-out, 0.15s margin ease-out, 0.15s border ease-out;
}

.input-box.noresults:not(.error) {
  margin-top: 100px;
  margin-bottom: 100px;
}

.input-box.noresults.error { margin-top: 100px; }
.input-box:not(.noresults):not(.error) input { margin-bottom: 20px; }

.result-box {
  border-top: 1px lightgray solid;
  padding-top: 20px;
  margin-left: -25px;
  margin-right: -25px;
  padding: 15px 15px 0 15px;
}

.result-box .result:first-child { margin-top: 40px; }
.result-box .result:not(:first-child) { margin-top: 15px; }

.list-enter, .list-leave-to { opacity: 0; }
.list-enter-active, .list-leave-active {
  animation-duration: .2s;
  animation-name: fadeInRight;
}

@keyframes fadeInRight {
   0% { opacity: 0; transform: translateX(200px); }
   100% { opacity: 1; transform: translateX(0); }
}
</style>
