
<script>
import * as couleurs from "@/assets/couleurs.json"
  export default {
    name: "Box-options",
    props: {
      case: String
    },
    computed: {
      c: function() {
        return '#'+couleurs[this.case].split('x')[1]
      }

    },
    mounted() {
    }
  }
</script>
<template>
  <span class="box" :style="{'background-color':c}"></span>
</template>

<style>
  .box {
    float: left;
    height: 30px;
    width: 30px;
    clear: both;
    border-radius: 50%;
    margin-right: 1vw;
    background-color: v-bind(c);
  }
</style>