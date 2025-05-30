<template>
  <div ref="card" class="card" :class="{ shadow, border }" :style="style">
    <div ref="wrapper" class="wrapper" :style="wrapperStyle">
      <slot />
    </div>
  </div>
</template>

<script>
export default {
  props: {
    shadow: { type: Boolean, default: true },
    border: { type: Boolean, default: false },
    wrapperStyle: { type: Object, default: () => {} },
    ignoreStyleMutations: { type: Boolean, default: false }, // whether to ignore content style mutations when deciding when to recalculate height
  },
  data() {
    return {
      height: 0,
      margin: 7,
      spanValue: 0,
      mutationObserver: null,
      debounceTimeout: null,
    };
  },
  computed: {
    style() {
      const { height, margin, spanValue } = this;
      return {
        height: `${height}px`,
        margin: `${margin}px`,
        gridRow: `span ${spanValue}`,
      };
    },
  },
  mounted() {
    this.setHeight();
    // for some reason, card doesn't open to full height on first load
    setTimeout(this.setHeight, 250);
    window.addEventListener('resize', this.debounceSetHeight);
    // The MutationObserver will detect when any children or descendants are added
    // and when any CSS is changed
    this.mutationObserver = new MutationObserver(this.setHeight);
    this.mutationObserver.observe(this.$refs.wrapper, {
      attributes: true,
      childList: true,
      subtree: true,
      attributeFilter: this.ignoreStyleMutations ? [] : ['style'],
    });
  },
  destroyed() {
    window.removeEventListener('resize', this.debounceSetHeight);
    if (this.mutationObserver) {
      this.mutationObserver.disconnect();
    }
  },
  methods: {
    // call setHeight() manually from parent component whenever the content (slot) height changes
    // and the change is undetectable by MutationObserver
    setHeight() {
      const { margin, $refs } = this;
      if ($refs && $refs.wrapper) {
        this.height = $refs.wrapper.offsetHeight;
        // Adjust the number of rows the card spans (necessary for the masonry layout to work)
        this.spanValue = Math.ceil((this.height + margin * 2) / 5); // 5 is row height
        this.$emit('height-change');
      }
    },
    debounceSetHeight() {
      clearTimeout(this.debounceTimeout);
      this.debounceTimeout = setTimeout(() => this.setHeight(), 250);
    },
  },
};
</script>

<style lang="sass" scoped>
@import '@/styles/style.sass'
.card
  background-color: var(--secondaryBackground) !important
  border-radius: 15px
  position: relative
  transition: height .3s
  overflow: hidden
  &.shadow
    +shadow-light
  &.border
    border: #999 1px solid

  .wrapper
    overflow: hidden
</style>
