<template>
  <div
    style="width: 100%"
    @drop="drop($event)"
    @dragenter.prevent
    @dragover.prevent
  >
    <div
      :style="`width:${canw}px; height:${canh}px; left:${offx}px; top:${offy}px`"
      class="absolute border-2 border-solid border-blue-700 bg-slate-100"
      @dragover="dragOver"
    >
      <img
        v-for="(layer, index) in layers"
        :key="index"
        :src="layer.svgImage"
        class="absolute"
        draggable="false"
        :style="
          `width:${canw}px; height:${canh}px;` +
          `opacity:${index > layerIndex ? '0.5' : '1.0'}`
        "
      />
      <div
        v-for="(cursor, index) in cursors"
        :key="index"
        :style="`width:${curw}px; height:${curh}px; left:${
          cursor.x - curw / 2
        }px; top:${cursor.y - curh / 2}px`"
        :class="`${
          index == pointIndex ? 'border-blue-800' : 'border-blue-400'
        } ${cursor.c ? '' : 'rounded-xl'}`"
        draggable="true"
        class="absolute border-2 border-solid"
        @dragstart="dragStart($event, index)"
        @click="onSelect($event, index)"
      />
    </div>
    <div
      :style="`width:${sidew}px; height:${canh}px; left:${
        offx + canw - 2
      }px; top:${offy}px`"
      class="absolute border-2 border-solid border-blue-700 bg-slate-300"
    >
      <div class="ml-2 mr-2 flex justify-between">
        <div>
          <button
            @click="undo"
            :disabled="!isUndoable()"
            :style="`opacity:${isUndoable() ? '1.0' : '0.5'}`"
          >
            <span class="material-icons">undo</span>
          </button>
          <button
            :style="`opacity:${isRedoable() ? '1.0' : '0.5'}`"
            class="ml-1"
            @click="redo"
            :disabled="!isRedoable()"
          >
            <span class="material-icons">redo</span>
          </button>
        </div>
        <button @click="onClose">
          <span class="material-icons">close</span>
        </button>
      </div>
      <div>
        <button @click="toggleGrid" class="ml-2 flex">
          <span class="material-icons">view_module</span>
          <span>{{ grid }}</span>
        </button>
      </div>
      <div class="ml-2 mr-2 flex justify-between">
        <button @click="togglePoint">
          <span v-if="isSharpCorner()" class="material-icons"
            >check_box_outline_blank</span
          >
          <span v-else class="material-icons">radio_button_unchecked</span>
        </button>
        <button
          :disabled="cursors.length <= 3"
          @click="deletePoint"
          :style="`opacity:${cursors.length > 3 ? '1.0' : '0.5'}`"
        >
          <span class="material-icons">delete</span>
        </button>
        <button @click="splitSegment">
          <span class="material-icons">add_circle</span>
        </button>
      </div>
      <input
        v-model.trim="currentColor"
        v-on:focus="onColorFocus"
        class="focus:shadow-outline w-full appearance-none rounded border py-2 px-3 text-center leading-tight text-gray-700 shadow focus:outline-none"
        id="username"
        type="text"
        :placeholder="'#RRGGBB(AA)'"
      />
      <div :style="`height:${canh / 2}px; overflow-y: scroll`">
        <div v-for="(layer, index) in layers" :key="index">
          <div v-if="index == layerIndex">
            <button @click="insertLayer(index)">
              <span class="material-icons">add</span>
            </button>
          </div>
          <img
            @click="onSelectLayer($event, index)"
            :src="layer.svgImage"
            :style="`width:${sidew}px;height:${sidew / 2}px`"
            class="border-2 border-solid object-fill"
            :class="`${
              index == layerIndex ? 'border-blue-400' : 'border-slate-200'
            }`"
          />
          <div
            v-if="index == layerIndex"
            class="ml-2 mr-2 flex justify-between"
          >
            <button @click="insertLayer(index + 1)">
              <span class="material-icons">add</span>
            </button>
            <button
              v-if="layers.length > 1"
              class="ml-2"
              @click="deleteLayer()"
            >
              <span class="material-icons">delete</span>
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, watch } from "vue";
import {
  Point,
  Layer,
  Drawing,
  svgImageFromPath,
  pathFromPoints,
  splitPoint,
  togglePointType,
} from "@/models/point";

const [canw, canh, offx, offy, curw, curh, sidew] = [
  512, 512, 40, 80, 30, 30, 150,
];

const roundRect: Point[] = [
  { x: canw / 4, y: canh / 4, c: false },
  { x: canw - canw / 4, y: canh / 4, c: false },
  { x: canw - canw / 4, y: canh - canh / 4, c: false },
  { x: canw / 4, y: canh - canh / 4, c: false },
];

interface State {
  layers: Layer[];
  layerIndex: number;
  pointIndex: number;
}

export default defineComponent({
  name: "HomePage",
  components: {},
  props: ["drawing"],
  setup(props, context) {
    const grid = ref<number>(0);
    const undoStack = ref<State[]>([]);
    const undoIndex = ref<number>(0);
    const recordState = () => {
      const array = undoStack.value.filter((state, index) => {
        return index < undoIndex.value;
      });
      array.push({
        layers: layers.value,
        layerIndex: layerIndex.value,
        pointIndex: pointIndex.value,
      });
      undoStack.value = array;
      undoIndex.value = undoStack.value.length;
    };

    const isRedoable = () => {
      return undoIndex.value + 1 < undoStack.value.length;
    };

    const isUndoable = () => {
      return undoIndex.value > 0;
    };
    const undo = () => {
      console.log("undo", isUndoable());
      if (!isUndoable()) {
        return;
      }
      if (!isRedoable()) {
        recordState();
        undoIndex.value -= 1;
      }
      const state = undoStack.value[undoIndex.value - 1];
      layers.value = state.layers;
      updateLayerIndex(state.layerIndex);
      pointIndex.value = state.pointIndex;
      undoIndex.value -= 1;
    };
    const redo = () => {
      if (!isRedoable()) {
        return;
      }
      const state = undoStack.value[undoIndex.value + 1];
      layers.value = state.layers;
      updateLayerIndex(state.layerIndex);
      pointIndex.value = state.pointIndex;
      undoIndex.value += 1;
    };
    const onColorFocus = (evt: Event) => {
      recordState();
    };

    const layerIndex = ref<number>(0);
    const pointIndex = ref<number>(0);
    //console.log("initialLayers", props.initialLayers ? "A" : "B");
    const layers = ref<Layer[]>(
      props.drawing.layers?.length > 0
        ? props.drawing.layers
        : [
            {
              points: roundRect,
              color: "",
              path: "",
              svgImage: "",
            },
          ]
    );
    const cursors = ref<Point[]>([]);
    const currentColor = ref<string>("");
    watch([cursors, currentColor], ([points, color]) => {
      layers.value = layers.value.map((layer, index) => {
        if (index == layerIndex.value) {
          const path = pathFromPoints(points);
          return {
            points,
            color,
            path,
            svgImage: svgImageFromPath(path, color),
          };
        }
        return layer;
      });
    });
    const updateLayerIndex = (index: number) => {
      layerIndex.value = (index + layers.value.length) % layers.value.length;
      const layer = layers.value[layerIndex.value];
      cursors.value = layer.points;
      currentColor.value = layer.color;
    };
    updateLayerIndex(0);

    const offsetX = ref<number>(0);
    const offsetY = ref<number>(0);
    const onSelect = (evt: Event, index: number) => {
      pointIndex.value = index;
    };
    const dragStart = (evt: MouseEvent, index: number) => {
      //evt.dataTransfer.setData('index', index)
      offsetX.value = evt.offsetX;
      offsetY.value = evt.offsetY;
      pointIndex.value = index;
      recordState();
    };
    const dragOver = (evt: MouseEvent) => {
      // const index = evt.dataTransfer.getData('index')
      cursors.value = cursors.value.map((cursor, index) => {
        if (index == pointIndex.value) {
          const g = grid.value;
          const x = Math.max(
            0,
            Math.min(
              canw - g - 1,
              evt.clientX - offx - offsetX.value + curw / 2 - 3
            )
          );
          const y = Math.max(
            0,
            Math.min(
              canh - g - 1,
              evt.clientY - offy - offsetY.value + curh / 2 - 3
            )
          );
          return {
            x: g == 0 ? x : Math.round((x + g / 2) / g) * g,
            y: g == 0 ? y : Math.round((y + g / 2) / g) * g,
            c: cursor.c,
          };
        }
        return cursor;
      });
      evt.preventDefault();
    };
    const togglePoint = () => {
      recordState();
      cursors.value = togglePointType(cursors.value, pointIndex.value);
    };
    const isSharpCorner = () => {
      return cursors.value[pointIndex.value].c;
    };
    const splitSegment = () => {
      recordState();
      cursors.value = splitPoint(cursors.value, pointIndex.value);
      pointIndex.value = pointIndex.value + 1;
    };
    const deletePoint = () => {
      if (cursors.value.length <= 3) {
        return;
      }
      recordState();
      cursors.value = cursors.value.filter((cursor, index) => {
        return index != pointIndex.value;
      });
      pointIndex.value =
        (pointIndex.value + cursors.value.length - 1) % cursors.value.length;
    };
    const insertLayer = (index: number) => {
      recordState();
      const array = layers.value.map((layer) => layer);
      const path = pathFromPoints(roundRect);
      const newLayer = {
        points: roundRect,
        color: currentColor.value,
        path,
        svgImage: svgImageFromPath(path, currentColor.value),
      };
      array.splice(index, 0, newLayer);
      layers.value = array;
      updateLayerIndex(index);
    };
    const deleteLayer = () => {
      if (layers.value.length == 1) {
        return;
      }
      recordState();
      layers.value = layers.value.filter((layer, index) => {
        return index != layerIndex.value;
      });
      updateLayerIndex(layerIndex.value - 1);
    };
    const onSelectLayer = (evt: Event, index: number) => {
      updateLayerIndex(index);
    };
    const drop = (evt: MouseEvent) => {
      evt.preventDefault();
    };
    const toggleGrid = () => {
      grid.value = (grid.value + 8) % 40;
    };
    const onClose = () => {
      const drawing: Drawing = {
        layers: layers.value,
        assetId: 0,
      };
      context.emit("close", drawing);
    };
    return {
      cursors,
      pointIndex,
      currentColor,
      canw,
      canh,
      curw,
      curh,
      offx,
      offy,
      sidew,
      dragStart,
      dragOver,
      drop,
      togglePoint,
      isSharpCorner,
      splitSegment,
      deletePoint,
      onSelect,
      insertLayer,
      deleteLayer,
      onSelectLayer,
      onColorFocus,
      onClose,
      undo,
      redo,
      isUndoable,
      isRedoable,
      layerIndex,
      layers,
      grid,
      toggleGrid,
    };
  },
});
</script>
