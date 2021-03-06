<template>
  <div :class="flyoutClass" :style="flyoutStyle" class="flyout">
    <div :style="sidebarStyle" class="flyout-sidebar">
      <div :style="sidebarContentStyle" class="flyout-sidebar-content">
        <slot name="sidebar"/>
      </div>
    </div>
    <div :style="contentStyle"
         class="flyout-content"
         @click.passive="onContentClick"
         @touchstart.passive="onContentTouchStart"
         @touchmove.passive="onContentTouchMove"
         @touchcancel.passive="onContentTouchCancel"
         @touchend.passive="onContentTouchEnd">
      <slot/>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from "vue";
import Timer from "@/framework/Timer";
import { unregister } from "register-service-worker";

type Move = "content" | "sidebar" | "both";
type Side = "left" | "right";

export default Vue.extend({
  props: {
    // The duration of the animation in milliseconds.
    duration: {
      default: 300,
      type: Number
    },
    // Which element to animate. The sidebar, content or both.
    move: {
      default: "both",
      type: String as () => Move
    },
    overlayOpacity: {
      default: 0.1,
      type: Number
    },
    // Show the flyout on the left or the right hand side.
    side: {
      default: "right",
      type: String as () => Side
    },
    // Only starts moving the sidebar when the user drags for more than the specified threshold.
    threshold: {
      default: 20,
      type: Number
    },
    // The number of px needed for the sidebar can be opened completely, otherwise it closes.
    tolerance: {
      default: 70,
      type: Number
    },
    // True if the flyout is open, otherwise false.
    value: {
      default: false,
      type: Boolean
    },
    // The width of the flyout.
    width: {
      default: 256,
      type: Number
    }
  },
  data() {
    return {
      closing: false,
      opening: false,
      moved: false,
      isOpen: false,

      scrolling: false,
      scrollTimeout: undefined as NodeJS.Timer | undefined,
      onScrollInternal: (e: Event) => {},

      startOffsetX: 0,
      currentOffsetX: 0,
      translateX: 0,
      translateTo: 0
    };
  },
  computed: {
    flyoutClass(): any {
      let classes: any = {
        "flyout-opening": this.opening,
        "flyout-closing": this.closing,
        "flyout-left": this.side === "left",
        "flyout-right": this.side === "right"
      };
      classes[`flyout-move-${this.move}`] = true;
      return classes;
    },
    flyoutStyle(): any {
      const opacity = Math.abs(this.translateX) / this.width;
      const style: any = {
        "--flyout-overlay-opacity": opacity * this.overlayOpacity,
        "--flyout-sidebar-content-opacity": opacity,
        "--flyout-transition-duration": `${this.duration}ms`
      };
      if (this.move === "both") {
        return {
          ...style,
          transform: this.transformString
        };
      }
      return style;
    },
    sidebarStyle(): any {
      let style: any = {
        width: this.sidebarWidth
      };
      if (this.move === "sidebar") {
        style.transform = this.sidebarTransformString;
      }
      return style;
    },
    sidebarContentStyle(): any {
      return {
        opacity: Math.abs(this.translateX) / this.width
      };
    },
    contentStyle(): any {
      if (this.move === "content") {
        return {
          transform: this.transformString
        };
      }
      return undefined;
    },
    sidebarTransformString(): string {
      if (this.translateX === 0) {
        return "";
      }
      return `translateX(${this.translateX - this.orientation * this.width}px)`;
    },
    sidebarWidth(): string {
      return `${this.width}px`;
    },
    transformString(): string {
      if (this.translateX === 0) {
        return "";
      }
      return `translateX(${this.translateX}px)`;
    },
    orientation(): number {
      return this.side === "right" ? -1 : 1;
    }
  },
  watch: {
    value(newValue, oldValue) {
      if (this.value) {
        this.open();
      } else {
        this.close();
      }
    }
  },
  created() {
    this.translateTo = this.width;
    this.translateTo *= this.orientation;

    if (document) {
      const self = this;
      this.onScrollInternal = this.decouple(document, "scroll", function() {
        // Decouple scroll event
        if (!self.moved) {
          if (self.scrollTimeout) {
            clearTimeout(self.scrollTimeout);
          }
          self.scrolling = true;
          self.scrollTimeout = setTimeout(function() {
            self.scrolling = false;
          }, 250);
        }
      });
    }
  },
  destroyed() {
    if (document) {
      document.removeEventListener("scroll", this.onScrollFn);
    }
  },
  methods: {
    async open() {
      this.$emit("opening");
      this.translateXTo(this.translateTo);
      this.closing = false;
      this.opening = true;
      this.isOpen = true;
      if (document) {
        document.documentElement!.classList.add("flyout-open");
      }

      await Timer.delay(this.duration + 50);

      this.opening = false;
      this.$emit("input", this.isOpen);
    },
    async close() {
      if (!(this.isOpen || this.opening)) {
        return;
      }

      this.$emit("closing");
      this.translateXTo(0);
      this.closing = true;
      this.opening = false;
      this.isOpen = false;

      await Timer.delay(this.duration + 50);

      if (document) {
        document.documentElement!.classList.remove("flyout-open");
      }
      this.translateX = 0;
      this.closing = false;
      this.$emit("input", this.isOpen);
    },
    onContentClick() {
      if (this.isOpen) {
        this.close();
      }
    },
    onContentTouchStart(event: TouchEvent) {
      if (event.touches === undefined) {
        return;
      }
      // Reset values
      this.moved = false;
      this.closing = false;
      this.opening = false;
      this.startOffsetX = event.touches[0].pageX;
    },
    onContentTouchCancel() {
      // Reset values
      this.moved = false;
      this.closing = false;
      this.opening = false;
    },
    onContentTouchMove(event: TouchEvent) {
      // Translates content
      if (
        this.scrolling ||
        event.touches === undefined ||
        this.hasIgnoredElements(event.target as HTMLElement)
      ) {
        return;
      }

      const dif_x = event.touches[0].clientX - this.startOffsetX;
      let translateX = (this.currentOffsetX = dif_x);

      if (Math.abs(translateX) > this.width) {
        return;
      }

      if (Math.abs(dif_x) > this.threshold) {
        this.opening = true;

        const oriented_dif_x = dif_x * this.orientation;

        if (
          (this.isOpen && oriented_dif_x > 0) ||
          (!this.isOpen && oriented_dif_x < 0)
        ) {
          return;
        }

        // if (!this.moved) {
        //   this.$emit("translatestart");
        // }

        if (oriented_dif_x <= 0) {
          translateX = dif_x + this.width * this.orientation;
          this.opening = false;
        }

        this.translateX = translateX;
        // this.$emit("translate", this.translateX);
        this.moved = true;
      } else if (this.isOpen) {
        this.closing = true;
      }
    },
    onContentTouchEnd() {
      // Toggles flyout
      if (this.moved) {
        // this.$emit("translateend");
        this.opening && Math.abs(this.currentOffsetX) > this.tolerance
          ? this.open()
          : this.close();
      }
      this.moved = false;
      this.closing = false;
      this.opening = false;
    },
    onScrollFn(event: Event) {
      return this.onScrollInternal(event);
    },
    translateXTo(translateX: number) {
      // Translates content and updates currentOffset with a given X point
      this.currentOffsetX = translateX;
      this.translateX = translateX;
    },
    hasIgnoredElements(element: HTMLElement | undefined) {
      if (element) {
        while (element.parentNode) {
          if (element.getAttribute("data-flyout-ignore") !== null) {
            return element;
          }
          element = element.parentNode as HTMLElement;
        }
      }
      return null;
    },
    decouple(node: Node, event: string, fn: () => void) {
      let eve: Event;
      let tracking = false;

      function captureEvent(e: Event) {
        eve = e;
        track();
      }

      function track() {
        if (!tracking) {
          window.requestAnimationFrame(update);
          tracking = true;
        }
      }

      function update() {
        fn.call(node, eve);
        tracking = false;
      }

      node.addEventListener(event, captureEvent, { passive: true });

      return captureEvent;
    }
  }
});
</script>

<style lang="scss">
.flyout {
  --flyout-content-background-colour: var(--global-background-colour);
  --flyout-sidebar-background-colour: var(--global-primary-colour);
}

.flyout-content {
  // A background-color is required
  background-color: var(--flyout-content-background-colour);
  min-height: 100vh;
  position: relative;
}

.flyout-content:before {
  content: "";
  background-color: rgb(0, 0, 0);
  bottom: 0;
  display: block;
  opacity: 0;
  position: absolute;
  top: 0;
  width: 100%;
  will-change: opacity;
  visibility: hidden;
  z-index: 99;
}

.flyout-sidebar {
  background-color: var(--flyout-sidebar-background-colour);
  bottom: 0;
  min-height: 100vh;
  overflow-y: auto;
  position: fixed;
  top: 0;
  visibility: hidden;
}

.flyout-sidebar-content {
  will-change: opacity;
}

.flyout-left {
  .flyout-sidebar {
    left: 0;
  }
}

.flyout-right {
  .flyout-sidebar {
    right: 0;
  }
}

.flyout-move-content {
  .flyout-content {
    will-change: transform;
    z-index: 1;
  }

  .flyout-sidebar {
    z-index: 0;
  }

  &.flyout-opening {
    .flyout-content {
      transition-duration: var(--flyout-transition-duration);
      transition-property: transform;
      transition-timing-function: ease(out-quint);
    }
  }

  &.flyout-closing {
    .flyout-content {
      transition-duration: var(--flyout-transition-duration);
      transition-property: transform;
      transition-timing-function: ease(in-quint);
    }
  }
}

.flyout-move-sidebar {
  .flyout-content {
    z-index: 0;
  }

  .flyout-sidebar {
    will-change: transform;
    z-index: 1;
  }

  &.flyout-left {
    .flyout-sidebar {
      transform: translate(-100%, 0);
    }
  }

  &.flyout-right {
    .flyout-sidebar {
      transform: translate(100%, 0);
    }
  }

  &.flyout-opening {
    .flyout-sidebar {
      transition-duration: var(--flyout-transition-duration);
      transition-property: transform;
      transition-timing-function: ease(out-quint);
    }
  }

  &.flyout-closing {
    .flyout-sidebar {
      transition-duration: var(--flyout-transition-duration);
      transition-property: transform;
      transition-timing-function: ease(in-quint);
    }
  }
}

.flyout-move-both {
  &.flyout {
    will-change: transform;
  }

  &.flyout-left {
    .flyout-sidebar {
      transform: translate(-100%, 0);
    }
  }

  &.flyout-right {
    .flyout-sidebar {
      transform: translate(100%, 0);
    }
  }

  &.flyout-opening {
    &.flyout {
      transition-duration: var(--flyout-transition-duration);
      transition-property: transform;
      transition-timing-function: ease(out-quint);
    }
  }

  &.flyout-closing {
    &.flyout {
      transition-duration: var(--flyout-transition-duration);
      transition-property: transform;
      transition-timing-function: ease(in-quint);
    }
  }
}

.flyout-opening {
  .flyout-content:before {
    opacity: var(--flyout-overlay-opacity);
    transition-duration: var(--flyout-transition-duration);
    transition-property: opacity;
    transition-timing-function: ease(out-quint);
    visibility: visible;
  }

  .flyout-sidebar {
    visibility: visible;
  }

  .flyout-sidebar-content {
    transition-duration: var(--flyout-transition-duration);
    transition-property: opacity;
    transition-timing-function: ease(out-quint);
  }
}

.flyout-closing {
  .flyout-content:before {
    opacity: var(--flyout-overlay-opacity);
    transition-duration: var(--flyout-transition-duration);
    transition-property: opacity;
    transition-timing-function: ease(in-quint);
    visibility: visible;
  }

  .flyout-sidebar {
    visibility: visible;
  }

  .flyout-sidebar-content {
    transition-duration: var(--flyout-transition-duration);
    transition-property: opacity;
    transition-timing-function: ease(in-quint);
  }
}

.flyout-open {
  body {
    overflow: hidden;
  }

  .flyout-content {
    overflow: hidden;
  }

  .flyout-content:before {
    opacity: var(--flyout-overlay-opacity);
    visibility: visible;
  }

  .flyout-sidebar {
    visibility: visible;
  }
}
</style>
