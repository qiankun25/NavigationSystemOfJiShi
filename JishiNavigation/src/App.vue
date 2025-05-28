<template>
    <div class="app">
        <FloorMap />
    </div>
</template>

<script setup>
import FloorMap from './components/FloorMap.vue'
</script>

<style>
body {
    margin: 0;
    padding: 0;
    overflow: hidden;
}

.app {
    width: 100vw;
    height: 100vh;
    background: #f0f0f0;
}
</style>