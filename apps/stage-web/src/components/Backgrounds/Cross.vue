<template>
  <div class="cross-background-container">
    <slot />
  </div>
</template>

<style lang="css" scoped>
/*
CSS Background Patterns by MagicPattern
https://www.magicpattern.design/tools/css-backgrounds
*/
.cross-background-container {
  background-color: #ffffff;
  --cross-color: oklch(0.85 calc(var(--theme-colors-chroma) * 0.3) var(--theme-colors-hue) / 0.2);
  background:
    radial-gradient(circle, transparent 20%, #ffffff 20%, #ffffff 80%, transparent 80%, transparent),
    radial-gradient(circle, transparent 20%, #ffffff 20%, #ffffff 80%, transparent 80%, transparent) 25px 25px,
    linear-gradient(var(--cross-color) 2px, transparent 2px) 0 -1px,
    linear-gradient(90deg, var(--cross-color) 2px, #ffffff 2px) -1px 0;
  background-size:
    50px 50px,
    50px 50px,
    25px 25px,
    25px 25px;
}

.dark .cross-background-container {
  background-color: #121212;
  --cross-color: oklch(0.2 calc(var(--theme-colors-chroma) * 0.3) var(--theme-colors-hue));
  background:
    radial-gradient(circle, transparent 20%, #121212 20%, #121212 80%, transparent 80%, transparent),
    radial-gradient(circle, transparent 20%, #121212 20%, #121212 80%, transparent 80%, transparent) 25px 25px,
    linear-gradient(var(--cross-color) 2px, transparent 2px) 0 -1px,
    linear-gradient(90deg, var(--cross-color) 2px, #121212 2px) -1px 0;
  background-size:
    50px 50px,
    50px 50px,
    25px 25px,
    25px 25px;
}
</style>
