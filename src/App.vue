<template>
  <router-view />
</template>

<script>
import "./assets/tailwind.css";

export default {
  name: "App",
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

#mhm {
  float: left;
  margin-left: 100px;
  margin-top: -2px;
}
#nav {
  text-align: right;
  margin-right: 50px;
}

#nav a {
  font-weight: bold;
  color: lightslategray;
}

#nav a.router-link-exact-active {
  color: black;
}
</style>
