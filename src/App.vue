<template>
    <div class="container">
        <div class="videos">
            <video class="video" :src-object.prop.camel="mediaStream" autoplay />
            <video class="video" :src-object.prop.camel="mediaStream" autoplay />
        </div>
    </div>
</template>

<script setup>
import { onMounted, ref } from 'vue';

const mediaStream = ref('');

onMounted(async () => {
    try {
        mediaStream.value = await navigator.mediaDevices.getDisplayMedia({ video: true });
        console.log(mediaStream.value);
    } catch (error) {
        alert('Please give audio');
    }
})


</script>

<style lang="scss">


* {
    padding: 0;
    margin: 0;
    box-sizing: border-box;
    font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
}

body {
    background: #011741;
}



.container {
    width: 1200px;
    margin: 0px auto;
    padding: 0 15px;
}

.videos {
    margin-top: 100px;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 50px;
}

.video {
    max-width: 100%;
    border: 3px solid #ffffff;
    border-radius: 10px;
    overflow: hidden;
}
</style>