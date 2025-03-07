<template>
  <div class="container bg-white shadow rounded px-3 py-3 my-3">
    <Top class="mt-3 mb-4 mx-2"></Top>
    <Problems v-bind="problems"></Problems>
    <div class="row">
      <div class="col-12 col-lg-6">
        <Map @move="loadSegments" @ready="onMapReady" ref="map"></Map>
        <img src="/img/powered-by-strava.svg" alt="Powered by Strava" height="30" class="float-end" />
      </div>
      <div class="col-12 col-lg-6">
        <Results :segments="segmentsArray" :bounds="bounds" v-if="bounds && segmentsArray.length > 0"></Results>
        <div v-if="problems.notConnected" class="pt-5 text-center">
          <h4>
            <span class="text-muted">Not connected</span>
          </h4>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
  import { defineComponent } from 'vue';
  import Map from './components/Map.vue';
  import Results from './components/Results.vue';
  import { Bounds, Segment, SegmentDetails } from './types';
  import Cookies from 'js-cookie';
  import axios from 'axios';
  import Problems from '@/components/Problems.vue';
  import Top from '@/components/Top.vue';

  let timeout = 0;

  const api = axios.create({
    baseURL: 'https://www.strava.com/api/v3/',
    headers: {
      Authorization: 'Bearer ' + Cookies.get('access_token'),
    },
  });

  export default defineComponent({
    name: 'App',
    components: {
      Top,
      Problems,
      Map,
      Results,
    },
    data: () => ({
      segments: {} as { [key: string]: Segment },
      bounds: undefined as Bounds | undefined,
      problems: {
        limitReached: false,
        notConnected: false,
      },
    }),
    created() {
      api.interceptors.response.use(
        (res) => {
          this.problems.notConnected = false;
          this.problems.limitReached = false;
          return res;
        },
        (error) => {
          this.problems.notConnected = error.response.status === 401;
          this.problems.limitReached = error.response.status === 429;
          if (error.response.status === 401) {
            Cookies.remove('access_token');
          }

          if (this.problems.limitReached) {
            const secondsLeft = 60 * (15 - (new Date().getMinutes() % 15)) - new Date().getSeconds() + 15;
            clearTimeout(timeout);
            timeout = setTimeout(() => {
              this.problems.limitReached = false;
            }, secondsLeft * 1000);
          }

          return Promise.reject(error);
        }
      );
    },
    methods: {
      async loadSegments(bounds: Bounds) {
        this.bounds = bounds;

        if (this.problems.limitReached) {
          return;
        }

        const payload: Segment[] = await api
          .get('/segments/explore', {
            params: {
              bounds: bounds.join(','),
              activity_type: 'running',
            },
          })
          .then((res) => res.data.segments);

        await Promise.all(
          payload.map(async (segment) => {
            if (!this.segments[segment.id]) {
              this.segments[segment.id] = segment;
              (this.$refs.map as any).drawSegment(segment);

              const key = `segment_${segment.id}`;
              const cached = localStorage.getItem(key);
              if (!cached) {
                await api
                  .get('/segments/' + segment.id)
                  .then((res) => res.data)
                  .then((res: SegmentDetails) => {
                    this.segments[segment.id].details = res;
                    localStorage.setItem(key, JSON.stringify(this.segments[segment.id]));
                  });
              } else {
                this.segments[segment.id].details = JSON.parse(cached).details;
              }
            }
          })
        );
      },
      onMapReady() {
        this.segments = Object.keys(localStorage)
          .filter((k) => k.indexOf('segment_') === 0)
          .reduce((p: any, c: string) => {
            const key = c.substr('segment_'.length);
            p[key] = JSON.parse(localStorage.getItem(c) as string);
            (this.$refs.map as any).drawSegment(p[key]);
            return p;
          }, {});
      },
    },
    computed: {
      segmentsArray(): Segment[] {
        return Object.values(this.segments);
      },
    },
  });
</script>

<style lang="scss">
  @import './assets/bootstrap';
  @import '../node_modules/leaflet/dist/leaflet.css';
  @import '../node_modules/leaflet.locatecontrol/src/L.Control.Locate';
  @import url('https://fonts.googleapis.com/css2?family=Fira+Sans&family=Merriweather:wght@900&display=swap');

  .title {
    font-family: 'Merriweather', serif;
  }
</style>
