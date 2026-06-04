<script lang="ts" setup>
import { computed } from "vue";
import { useStorage } from "@vueuse/core";

const playNoteOnWrongNote = useStorage("playNoteOnWrongNote", true);
const jianpuMode = useStorage("jianpuMode", false);
const highlightWrongNote = useStorage("highlightWrongNote", true);
const selectedScore = useStorage("selectedScore", "samples/bach-prelude_1.mxl");

const options = computed(() => ({
  playNoteOnWrongNote: playNoteOnWrongNote.value,
  jianpuMode: jianpuMode.value,
  highlightWrongNote: highlightWrongNote.value,
  selectedScore: selectedScore.value
}));

defineExpose({
  options
});
</script>

<template>
  <div class="options">
    <label>
      playNoteOnWrongNote
      <input type="checkbox" v-model="playNoteOnWrongNote" />
    </label>
    <label>
      highlightWrongNote
      <input type="checkbox" v-model="highlightWrongNote" />
    </label>
    <label>
      jianpuMode
      <input type="checkbox" v-model="jianpuMode" />
    </label>
    <label>selectedScore
      <select v-model="selectedScore">
        <option value="samples/bach-prelude_1">Bach - Prelude 1</option>
        <option value="samples/mozart-das_veilchen">Mozart - Das Veilchen</option>
        <option value="evil/Continuer à t'aimer (Lumière Opera House) – Clair Obscur_ Expedition 33 – Lorien Testard">Lumiere Opera House</option>
      </select>
    </label>
  </div>
</template>

<style scoped lang="scss">
.options {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  width: 40ch;
}

label {
  display: flex;
  justify-content: space-between;
}
</style>
