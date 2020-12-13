<template>
  <Withdraw class="app" />
</template>

<script>
export default {
  mounted: function() {
    this.$vs.setTheme('dark')
  }
}
</script>

<style>
.app {
  width: 100%;
  height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
}

[vs-theme=dark] {
  --vs-background: 30, 32, 35;
  --vs-text: 255, 255, 255;
  --vs-gray-1: 24, 25, 28;
  --vs-gray-2: 20, 20, 23;
  --vs-gray-3: 15, 16, 19;
  --vs-gray-4: 10, 11, 14;
  --vs-shadow-opacity: .3;
  --vs-dark: 0, 0, 0;
  --vs-background-opacity: .6;
}

[vs-theme=dark] {
  background-color: rgba(var(--vs-background), 1);
  color: rgba(var(--vs-text), 1);
}
</style>