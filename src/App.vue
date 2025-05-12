<script setup lang="ts">
import JobDashboard from './components/JobDashboard.vue';
import ControlPanel from './components/ControlPanel.vue';
</script>

<template>
  <div class="app-layout"> 
    <ControlPanel />
    <JobDashboard />
  </div>
</template>

<style>
/* Apply global resets */
*,
*::before,
*::after {
  box-sizing: border-box;
}

body {
  margin: 0; /* Remove default body margin */
  font-family: sans-serif; /* Optional: Set a base font */
}

/* Layout for the two panels */
.app-layout {
  display: flex;
  height: 100vh; /* Full viewport height */
  width: 100vw; /* Full viewport width */
  overflow: hidden; /* Prevent outer scrollbars */
}

/* Let JobDashboard grow to fill space */
.job-dashboard {
  flex-grow: 1; /* Let this component grow */
  /* width: calc(100vw - 300px); Remove explicit width */
  overflow-y: auto; /* Allow dashboard itself to scroll */
  height: 100vh; /* Ensure it fills height */
}

/* Fix Control Panel width */
.control-panel {
  flex-shrink: 0; /* Prevent shrinking */
  width: 300px; /* Set a fixed width */
  /* Remove max/min width if fixed width is desired, or keep for flexibility */
  /* max-width: 300px; */
  /* min-width: 280px; */
  height: 100vh; /* Ensure it fills height */
  overflow-y: auto; /* Allow panel itself to scroll if content overflows */
}

/* Remove old logo styles if they are still present */
.logo {
  display: none;
}
</style>
