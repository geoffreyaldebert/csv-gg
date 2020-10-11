<template>
  <main>
    <button v-on:click="flyto">Locate Me</button>
    <div id="map"></div>
  </main>
</template>

<script>
import mapboxgl from "mapbox-gl";
export default {
  name: "Map",
  props: ['lon','lat'],
  data() {
    // Set initial data, this.createMap() configures event listeners that update data based on user interaction
    return {
      center: [2.35, 48.853], // St. Paul
      zoom: 10.5
    };
  },
  mounted() {
    // create the map after the component is mounted
    this.createMap();
  },
  methods: {
    createMap(lat=null,lon=null) {

      if(lat == null){
        this.mycenter = this.center;
      }else{
        this.mycenter = [lon, lat]
      }

      // instantiate map.  this method runs once after the vue component is mounted to the dom
      this.map = new mapboxgl.Map({
        accessToken:
          "pk.eyJ1Ijoic2Fib3N0aXgiLCJhIjoiY2p6Y3BkdnJ4MDd2czNjbWdsYXB4MTJoNSJ9.U65hBBejoDAAJH5wrdLejg",
        container: "map",
        style: "mapbox://styles/mapbox/streets-v11",
        minzoom: 5,
        center: this.mycenter, // use initial data as default
        zoom: this.zoom
      });

      this.marker = new mapboxgl.Marker({
            draggable: true
        })
        .setLngLat(this.mycenter)
        .addTo(this.map);

        
        this.marker.on('dragend', () => {
            console.log(this.marker._lngLat);
            this.$emit('childToParent', [this.marker._lngLat.lat, this.marker._lngLat.lng])
        });

      // set mapbox event listeners to update Vue component data
      //this.map.on("move", () => {
        // set the vue instance's data.center to the results of the mapbox instance method for getting the center
      //  this.center = this.map.getCenter();
      //});
      //this.map.on("zoom", () => {
        // set the vue instance's data.zoom to the results of the mapbox instance method for getting the zoom
      //  this.zoom = this.map.getZoom();
      //});
    },
    flyto(){
      this.createMap(this.lat, this.lon)
    }
  }
};
</script>

<style scoped>
#map {
  height: 400px;
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
  border: 1px solid darkgrey;
}
.text-container {
  max-width: 500px;
  display: flex;
  flex-direction: column;
  text-align: left;
  align-items: flex-start;
  margin: 0 auto; /* center text container */
}
</style>