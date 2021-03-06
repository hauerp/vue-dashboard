<template>
  <div ref="dashboard" class="dashboard-container" :style="style">
    <slot>
      <WidgetBox
        :dragSelector="dragSelector"
        v-for="item in layoutSort"
        :boxId="item.id"
        :draggable="item.pinned===false&&dragSelector!=='*'"
        :key="item.id"
      />
    </slot>
    <WidgetBox :dragSelector="dragSelector" class="placeholder" boxId="::placeholder::"></WidgetBox>
  </div>
</template>

<style>
.dashboard-container {
  position: relative;
  transition: min-width ease-out 0.1s, min-height ease-out 0.1s;
}

.dashboard-container .widget-box.placeholder {
  border: 1px dashed #000;
  background: none;
  z-index: 0;
}

.dashboard-container .widget-box.placeholder {
  transition: none !important;
}
</style>

<script>
import * as utils from "./utils.js";

export const List = new Set();

import WidgetBox from "./WidgetBox.vue";
export default {
  components: {
    WidgetBox
  },
  props: {
    layout: {
      type: Array,
      required: true
    },
    dragSelector: {
      type: String,
      default: "*" //  default: '*' .widget-move
    },
    /*
    cellSize: {
      type: Object,
      default() {
        return {
          w: 60,
          h: 60
        };
      }
    },*/
    maxColumnCount: {
      type: Number,
      default: 12
    },
    maxRowCount: {
      type: Number,
      default: Infinity
    },
    margin: {
      type: Number,
      default: 5
    },
    outerMargin: {
      type: Number,
      default: 0
    },
    bubbleUp: {
      type: Boolean,
      default: false
    },
    autoAddLayoutForNewBox: {
      type: Boolean,
      required: false,
      default: true
    },
    defaultSize: {
      type: Object,
      required: false,
      default() {
        return {
          w: 1,
          h: 1
        };
      }
    },
    fixLayoutOnLoad: {
      type: Boolean,
      required: false,
      default: true
    }
  },
  watch: {
    layout(newLayout) {
      if (this.fixLayoutOnLoad) {
        if (utils.layoutHasCollisions(newLayout)) {
          this.updateLayout(utils.fixLayout(newLayout, this.bubbleUp));
        }
      }
    }
  },
  data() {
    return {
      placeholder: {
        hidden: true,
        position: {
          x: 0,
          y: 0,
          w: 1,
          h: 1
        }
      },
      dragging: {
        boxLayout: null,
        offset: {
          x: 0,
          y: 0
        }
      },
      resizing: {
        boxLayout: null,
        offset: {
          x: 0,
          y: 0
        }
      },
      cellSize: {
          w: 60,
          h: 60
        },
      isMounted: false
    };
  },
  computed: {
    layoutSort(){
      return this.layout.sort((a, b) => a.id.localeCompare(b.id));
    },
    style() {
      return {};
      var layoutSize = utils.getLayoutSize(this.layout);
      return {
        minWidth:
          layoutSize.w * this.cellSize.w +
          (layoutSize.w - 1) * this.margin +
          2 * this.outerMargin +
          "px",
        minHeight:
          layoutSize.h * this.cellSize.h +
          (layoutSize.h - 1) * this.margin +
          2 * this.outerMargin +
          "px"
      };
    },
    pinnedLayout() {
      return this.layout.filter(boxLayout => {
        return boxLayout.pinned;
      });
    },
    layoutMap() {
      let map = new Map();
      this.layout.forEach(boxLayout => {
        map.set(boxLayout.id, boxLayout);
      });
      return map;
    }
  },
  methods: {
    getBoxLayoutById(id) {
      if (id === "::placeholder::") {
        return this.placeholder;
      }
      return this.layoutMap.get(id);
    },
    getPixelPositionById(id) {
      if (this.dragging.boxLayout && this.dragging.boxLayout.id === id) {
        let pixels = utils.positionToPixels(
          this.dragging.boxLayout.position,
          this.cellSize,
          this.margin,
          this.outerMargin
        );
        pixels.x += this.dragging.offset.x;
        pixels.y += this.dragging.offset.y;
        return pixels;
      }

      if (this.resizing.boxLayout && this.resizing.boxLayout.id === id) {
        let pixels = utils.positionToPixels(
          this.resizing.boxLayout.position,
          this.cellSize,
          this.margin,
          this.outerMargin
        );
        pixels.w += this.resizing.offset.x;
        pixels.h += this.resizing.offset.y;
        return pixels;
      }

      var boxLayout = this.getBoxLayoutById(id);
      return utils.positionToPixels(
        boxLayout.position,
        this.cellSize,
        this.margin,
        this.outerMargin
      );
    },
    isBoxVisible(id) {
      var boxLayout = this.getBoxLayoutById(id);
      return boxLayout && !boxLayout.hidden;
    },
    getPositionByPixel(x, y) {
      return {
        x: Math.round(x / (this.cellSize.w + this.margin)),
        y: Math.round(y / (this.cellSize.h + this.margin))
      };
    },
    updateLayout(layout) {
      this.$emit("update:layout", layout);
    },
    registerBox(box) {
      this.enableResizing(box);
      this.enableDragging(box);
      if (this.isMounted && this.autoAddLayoutForNewBox) {
        this.createBoxLayout(box.$props.boxId);
      }
    },
    unregisterBox(box) {},
    enableDragging(box) {
      var initialLayout;
      var isDragging = false;

      let validateTargetPosition = (targetX, targetY) => {
        if (
          targetX + this.dragging.boxLayout.position.w >
          this.maxColumnCount
        ) {
          targetX = this.maxColumnCount - this.dragging.boxLayout.position.w;
        } else {
          if (targetX < 0) {
            targetX = 0;
          }
        }
        if (targetY + this.dragging.boxLayout.position.h > this.maxRowCount) {
          targetY = this.maxRowCount - this.dragging.boxLayout.position.h;
        } else {
          if (targetY < 0) {
            targetY = 0;
          }
        }
        return {
          targetX,
          targetY
        };
      };

      box.$on("dragStart", evt => {
        var boxLayout = this.getBoxLayoutById(box.boxId);
        if (boxLayout.pinned) {
          return;
        }
        isDragging = true;

        // find box
        this.dragging.boxLayout = boxLayout;
        this.placeholder = {
          ... this.dragging.boxLayout
        };

        // clone layout
        initialLayout = utils.sortLayout(this.layout);

        this.$emit("drag:start", initialLayout);
      });

      box.$on("dragUpdate", evt => {
        if (!isDragging) {
          return;
        }
        this.dragging.offset.x = evt.offset.x;
        this.dragging.offset.y = evt.offset.y;

        var moveBy = this.getPositionByPixel(evt.offset.x, evt.offset.y);
        if (
          !utils.isFree(this.pinnedLayout, {
            ...this.dragging.boxLayout.position,
            x: this.dragging.boxLayout.position.x + moveBy.x,
            y: this.dragging.boxLayout.position.y + moveBy.y
          })
        ) {
          return;
        }

        let { targetX, targetY } = validateTargetPosition(
          this.dragging.boxLayout.position.x + moveBy.x,
          this.dragging.boxLayout.position.y + moveBy.y
        );

        // check if box has moved
        if (
          this.placeholder.position.x === targetX &&
          this.placeholder.position.y === targetY
        ) {
          return;
        }
        this.placeholder = utils.updateBoxPosition(this.placeholder, {
          x: targetX,
          y: targetY
        });

        var newLayout = [this.placeholder];
        initialLayout.forEach(boxLayout => {
          if (boxLayout.id === this.dragging.boxLayout.id) {
            return;
          }
          newLayout.push(
            utils.moveBoxToFreePlace(newLayout, boxLayout, this.bubbleUp)
          );
        });

        if (this.bubbleUp) {
          newLayout = utils.layoutBubbleUp(newLayout);
          this.placeholder = newLayout.find(boxLayout => {
            return boxLayout.id === this.dragging.boxLayout.id;
          });
        }
        this.updateLayout(newLayout);

        this.$emit("drag:update", newLayout);
      });

      box.$on("dragEnd", evt => {
        if (!isDragging) {
          return;
        }
        var moveBy = this.getPositionByPixel(evt.offset.x, evt.offset.y);
        if (
          utils.isFree(this.pinnedLayout, {
            ...this.dragging.boxLayout.position,
            x: this.dragging.boxLayout.position.x + moveBy.x,
            y: this.dragging.boxLayout.position.y + moveBy.y
          })
        ) {
          let { targetX, targetY } = validateTargetPosition(
            this.dragging.boxLayout.position.x + moveBy.x,
            this.dragging.boxLayout.position.y + moveBy.y
          );

          this.placeholder = utils.updateBoxPosition(this.placeholder, {
            x: targetX,
            y: targetY
          });
        }

        this.dragging.boxLayout = utils.updateBoxPosition(
          this.dragging.boxLayout,
          {
            x: this.placeholder.position.x,
            y: this.placeholder.position.y
          }
        );

        var newLayout = [this.dragging.boxLayout];
        initialLayout.forEach(boxPosition => {
          if (boxPosition.id === this.dragging.boxLayout.id) {
            return;
          }
          newLayout.push(
            utils.moveBoxToFreePlace(newLayout, boxPosition, this.bubbleUp)
          );
        });

        if (this.bubbleUp) {
          newLayout = utils.layoutBubbleUp(newLayout);
        }
        this.updateLayout(newLayout);

        this.dragging.boxLayout = null;
        this.dragging.offset.x = 0;
        this.dragging.offset.y = 0;

        this.placeholder.hidden = true;
        isDragging = false;

        this.$emit("drag:end", newLayout);
        this.$emit("UpdateLayout");
      });
    },
    enableResizing(box) {
      var initialLayout;
      var isResizing = false;

      let validateTargetSize = (targetW, targetH) => {
        if (
          this.resizing.boxLayout.position.x + targetW >
          this.maxColumnCount
        ) {
          targetW = this.maxColumnCount - this.resizing.boxLayout.position.x;
        } else {
          if (targetW < 1) {
            targetW = 1;
          }
        }
        if (this.resizing.boxLayout.position.y + targetH > this.maxRowCount) {
          targetH = this.maxRowCount - this.resizing.boxLayout.position.y;
        } else {
          if (targetH < 1) {
            targetH = 1;
          }
        }
        return {
          targetW,
          targetH
        };
      };

      box.$on("resizeStart", evt => {
        var boxLayout = this.getBoxLayoutById(box.boxId);
        if (boxLayout.pinned) {
          return;
        }
        isResizing = true;

        // find box
        this.resizing.boxLayout = boxLayout;
        this.placeholder = {
          ...this.resizing.boxLayout
        };

        // clone layout
        initialLayout = utils.sortLayout(this.layout);

        this.$emit("resize:start", initialLayout);
      });

      box.$on("resizeUpdate", evt => {
        if (!isResizing) {
          return;
        }
        this.resizing.offset.x = evt.offset.x;
        this.resizing.offset.y = evt.offset.y;

        var resizeBy = this.getPositionByPixel(evt.offset.x, evt.offset.y);
        if (
          !utils.isFree(this.pinnedLayout, {
            ...this.resizing.boxLayout.position,
            w: this.resizing.boxLayout.position.w + resizeBy.x,
            h: this.resizing.boxLayout.position.h + resizeBy.y
          })
        ) {
          return;
        }

        let { targetW, targetH } = validateTargetSize(
          this.resizing.boxLayout.position.w + resizeBy.x,
          this.resizing.boxLayout.position.h + resizeBy.y
        );

        // check if box size has changed
        if (
          this.placeholder.position.w === targetW &&
          this.placeholder.position.h === targetH
        ) {
          return;
        }
        this.placeholder = utils.updateBoxPosition(this.placeholder, {
          w: targetW,
          h: targetH
        });

        var newLayout = [this.placeholder];
        initialLayout.forEach(boxLayout => {
          if (boxLayout.id === this.resizing.boxLayout.id) {
            return;
          }
          newLayout.push(
            utils.moveBoxToFreePlace(newLayout, boxLayout, this.bubbleUp)
          );
        });

        if (this.bubbleUp) {
          newLayout = utils.layoutBubbleUp(newLayout);
          this.placeholder = newLayout.find(boxLayout => {
            return boxLayout.id === this.resizing.boxLayout.id;
          });
        }
        this.updateLayout(newLayout);

        this.$emit("resize:update", newLayout);
      });

      box.$on("resizeEnd", evt => {
        if (!isResizing) {
          return;
        }
        var resizeBy = this.getPositionByPixel(evt.offset.x, evt.offset.y);
        if (
          utils.isFree(this.pinnedLayout, {
            ...this.resizing.boxLayout.position,
            w: this.resizing.boxLayout.position.w + resizeBy.x,
            h: this.resizing.boxLayout.position.h + resizeBy.y
          })
        ) {
          let { targetW, targetH } = validateTargetSize(
            this.resizing.boxLayout.position.w + resizeBy.x,
            this.resizing.boxLayout.position.h + resizeBy.y
          );

          this.placeholder = utils.updateBoxPosition(this.placeholder, {
            w: targetW,
            h: targetH
          });
        }

        this.resizing.boxLayout = utils.updateBoxPosition(
          this.resizing.boxLayout,
          {
            w: this.placeholder.position.w,
            h: this.placeholder.position.h
          }
        );

        var newLayout = [this.resizing.boxLayout];
        initialLayout.forEach(boxPosition => {
          if (boxPosition.id === this.resizing.boxLayout.id) {
            return;
          }
          newLayout.push(
            utils.moveBoxToFreePlace(newLayout, boxPosition, this.bubbleUp)
          );
        });

        if (this.bubbleUp) {
          newLayout = utils.layoutBubbleUp(newLayout);
        }
        this.updateLayout(newLayout);

        this.resizing.boxLayout = null;
        this.resizing.offset.x = 0;
        this.resizing.offset.y = 0;

        this.placeholder.hidden = true;

        this.$emit("resize:end", newLayout);

        this.$emit("UpdateLayout");
      });
    },
    createBoxLayout(...boxIds) {
      boxIds = boxIds.filter(boxId => !this.getBoxLayoutById(boxId));

      if (boxIds.length) {
        let newLayout = [...this.layout];
        boxIds.forEach(boxId => {
          newLayout.push(
            utils.moveBoxToFreePlace(
              newLayout,
              {
                id: boxId,
                hidden: false,
                position: {
                  x: 0,
                  y: 0,
                  ...this.defaultSize
                }
              },
              this.bubbleUp
            )
          );
        });
        this.updateLayout(newLayout);
      }
    },
    WindowResize(e) {
      if(this.$refs.dashboard==undefined) return;
      var widthCell = parseInt((this.$refs.dashboard.clientWidth - this.margin*11) / 12);
      this.cellSize = {
        w: widthCell,
        h: this.cellSize.h
      };
    }
  },
  created() {
    List.add(this);
  },
  mounted() {
    this.isMounted = true;
    let boxIds = this.$children.map(box => box.$props.boxId);
    this.createBoxLayout(...boxIds);
    this.$nextTick(function() {
      this.WindowResize();
      window.addEventListener("resize", this.WindowResize);
    });
  },
  beforeDestroy() {
    List.delete(this);
    window.removeEventListener("resize", this.WindowResize);
  }
};
</script>
