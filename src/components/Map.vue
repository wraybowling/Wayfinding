<template>
  <div class="map-module">
    <div id="map" style='width: 100%; height: 100%;'></div>
    <nav>
      <div @touchstart="setLevel(3)" :class="{'viewing': floorViewing === 3, 'you-are-here': floorStanding === 3}">3</div>
      <div @touchstart="setLevel(2)" :class="{'viewing': floorViewing === 2, 'you-are-here': floorStanding === 2}">2</div>
      <div @touchstart="setLevel(1)" :class="{'viewing': floorViewing === 1, 'you-are-here': floorStanding === 1}">1</div>
      <div @touchstart="setLevel(0)" :class="{'viewing': floorViewing === 0, 'you-are-here': floorStanding === 0}">G</div>
    </nav>
    <div class="overlay">You are on Level {{floorStanding}}</div>
  </div>
</template>

<script>
  import { mapState } from 'vuex'

  mapboxgl.accessToken = 'pk.eyJ1IjoicmljaGxhbmRsaWJyYXJ5IiwiYSI6ImNpZnh5cDZ1NTRwbGN1dW0wcmhjMWZ3MHYifQ.gqkQbN_FAgHPRVsL6Vi-lA'
  var map;

  export default {
    name: 'map-module',
    computed: mapState ({
      // defaultFloor: state => state.defaults.floor,
      floorStanding: 'floorStanding',
      floorViewing: 'floorViewing',
      you: 'you',
      room_id: 'room_id',
      zoom: 'zoom',
      bearing: 'bearing',
      center: 'center',
      defaultMode: 'defaultMode'
    }),
    watch: {
      room_id() {
        this.renderFloor()
      },
      floorViewing() {
        this.renderFloor()
      },
      defaultMode(e) {
        if(e){
          map.flyTo({
            duration:30000,
            zoom: this.zoom,
            center: this.center,
            bearing: this.bearing,
            
          })
        }
      }
    },
    mounted: function() {
      map = new mapboxgl.Map({
          container: this.$el.querySelector('#map'),
          style: 'mapbox://styles/mapbox/light-v9',
          zoom: this.zoom,
          center: this.center,
          bearing: this.bearing,
          pitch:45
      });

      map.on('load', function () {

        map.addSource('indoorLabels',{
          type: 'geojson',
          data: '/static/data/geojson/labels.geojson'
        });

        map.addSource('indoorPolygons',{
          type: 'geojson',
          data: '/static/data/geojson/polygons.geojson'
        });

        map.addLayer({
          'id': 'areas',
          'minzoom': 17,
          'type': 'fill',
          'source': 'indoorPolygons',
          'paint': {
            'fill-color': 'rgb(171, 240, 81)'
          },
          filter: [
            'all',
            ['==','show_on_init',1],
          ]
        });

        map.addLayer({
          'id': 'walls',
          'minzoom': 16,
          'type': 'fill-extrusion',
          'source': 'indoorPolygons',
          'paint': {
            'fill-extrusion-color': 'rgb(210,210,210)',
            'fill-extrusion-height': 0.5,
          },
          filter: [
            'all',
            ['==','level_1',1],
            [
              'any',
              ['==','type','wall'],
              ['==','staff',1]
            ]
          ]
        });

        map.addLayer({
          'id': 'windows',
          'minzoom': 17,
          'type': 'fill-extrusion',
          'source': 'indoorPolygons',
          'paint': {
            'fill-extrusion-color': 'rgb(200,245,220)',
            'fill-extrusion-height': 1,
            'fill-extrusion-opacity': 0.6
          },
          filter: [
            'all',
            ['==','level_1',1],
            ['==','type','window']
          ]
        });

        [
          'stairs',
          'escelator-up',
          'escelator-down',
          'printer',
          'computer',
          'elevator',
          'emergency-exit',
          'exit'
        ].map( icon => 
          map.loadImage(`/static/img/${icon}.png`, function(error, image) {
            if (error) throw error;
            map.addImage(icon, image);
          })  
        )

        map.addLayer({
          'id': 'labels',
          'type': 'symbol',
          'source': 'indoorLabels',
          'minzoom': 18,
          'layout': {
            'text-optional': true,
            'text-field': `{label}`,
            // "text-font": ["Open Sans Semibold", "Arial Unicode MS Bold"],
            'text-size': 12,
            // "text-offset": [0, 0.1],
            'text-anchor': 'top',
            
            // 'icon-optional': false,
            'icon-ignore-placement': true,
            'icon-size': 0.75,
            'icon-anchor': 'bottom',
            'icon-image': '{symbol}',

            // 'visibility': ['case'
            //   ['>', ['get','priority'], 18], 'visible',
            //   'none'
            // ]
          },
          'paint': {
            'text-color': 'black',
            'text-halo-color': 'white',
            'text-halo-width': 2,

            'icon-halo-color': 'white',
            'icon-halo-width': 2

            // 'text-opacity': [
            //   'interpolate', ['linear'], 'zoom',
            //   ['>', ['zoom'], ['to-number',['get','priority']]],
            //   1.0
            // ]

            // 'text-opacity': [
            //   'interpolate', ['linear'], 'zoom',
            //   ['>', ['zoom'], ['to-number',['get','priority']]],
            //   1.0
            // ]
          },
          filter: [ 'all',
            ['==','level_1',1],
            ['!=','staff',1]
            // ['>','priority', 7]
          ]
        });

        var mkr = document.createElement('div');
        mkr.className = 'marker you-are-here'
        var youAreHere = new mapboxgl.Marker(mkr)
                            .setLngLat({lng: -81.0374490827052, lat: 34.00425027521804})
                            .addTo(map);
      });

      map.on('touchend', () => {
        this.$store.dispatch('used')
      });

      map.on('click', function(e){
        console.log(e);
      })
    },
    methods: {
      setLevel: function(levelNumber) {
        this.$store.commit('userSet',{ floorViewing: levelNumber })
      },
      renderFloor: function() {

        // console.log('set the level filter to',levelNumber)
        map.setFilter('areas',
        [
          'all',
          ['==',`level_${this.floorViewing}`,1],
          [
            'any',
            ['!=','type','wall'],
            ['!=','type','window'],
            ['!=','type','staff']
          ],
          ['==','room_id',this.room_id]
        ]);

        map.setFilter('walls',
        [
          'all',
          ['==',`level_${this.floorViewing}`,1],
          [
            'any',
            ['==','type','wall'],
            ['==','staff',1]
          ]
        ]);

        map.setFilter('windows',
        [
          'all',
          ['==',`level_${this.floorViewing}`,1],
          ['==','type','window'],
        ]);

        map.setFilter('labels',
        [
          'all',
          ['==',`level_${this.floorViewing}`,1],
          ['!=','staff',1]
        ]);
      }
    }
  }
</script>

<style lang="scss">
  $you: rgb(189, 15, 73);
  $viewing: rgb(171, 240, 81);

  .map-module{
    position:relative;

    nav {
      position: absolute;
      bottom:20%;
      right:5vw;
      background-color:white;
      border-radius: 1em;
      display:flex;
      flex-direction: column;
      text-align:center;
      overflow:hidden;
    }
    nav > *{
      padding:0.5em;
    }
    .viewing{
      background-color: $viewing;
    }
    .you-are-here{
      background-color: $you;
      font-weight: 800;
      color: white;
    }

    .marker{
      background-image: url('/static/img/you.svg');
      background-size:cover;
      width:2rem;
      height:2rem;
      border-radius: 50%;
      cursor: pointer;
    }

    .overlay{
      position:absolute;
      bottom:0;
      background-color: $you;
      z-index:2000;
      width:96vw;
      font-size:2rem;
      color:white;
      padding:0.5em 2vw;
      font-weight:800;
      text-align:center;
    }
  }

  $room:rgb(194, 202, 186);
  $room_focus: rgb(47, 140, 216);
</style>
