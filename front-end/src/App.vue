

<script>
  import MapSimulation from "@/components/Map";
  export default {
    name: 'App',
    data() {
      return {
        cssProps: {
          //backgroundImage: `url(${require('@/assets/bg.jpeg')})`,
        }
      }
    },
    components: {
      MapSimulation
    },
    created() {
      //document.body.style.backgroundImage = `url(${require('@/assets/Logo-RATP.png')})`;
      document.body.style.backgroundSize = 'cover';
    }
  }
</script>
<template>
<!--  <img alt="Vue logo" src="./assets/Logo-RATP.png" height="90">-->
  <div class="body-container">
    <MapSimulation/>
  </div>
</template>
<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 1vh;
}

.body-container {
  padding: 1vh;
  width: inherit;
  display: flex;
  flex-direction: row;
  flex-grow: 2;
  gap: 1vw;
  justify-content: space-evenly;
  ;
}

</style>
