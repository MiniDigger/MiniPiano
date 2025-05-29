<script setup lang="ts">
import { type NoteMessageEvent, WebMidi } from "webmidi";
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";

const inputDevices = ref<string[]>([]);
const outputDevices = ref<string[]>([]);
const inputDeviceName = ref<string | undefined>(undefined);
const outputDeviceName = ref<string | undefined>(undefined);
const input = computed(() => outputDeviceName.value ? WebMidi.getInputByName(outputDeviceName.value) : undefined);
const output = computed(() => outputDeviceName.value ? WebMidi.getOutputByName(outputDeviceName.value) : undefined);

watch(input, async (newValue) => {
  console.log("new device", newValue?.name);
  await newValue?.open();
  newValue?.addListener("noteon", noteOn);
});

function noteOn(e: NoteMessageEvent) {

  console.log("note", e.note.identifier, e.note.number);
}

onMounted(async () => {
  await WebMidi.enable();

  await reloadDevices();
  WebMidi.addListener("portschanged", reloadDevices);
});
onBeforeUnmount(() => WebMidi.disable());

async function reloadDevices() {
  await input.value?.destroy();
  await output.value?.destroy();

  inputDevices.value = WebMidi.inputs.map(input => input.name);
  outputDevices.value = WebMidi.outputs.map(input => input.name);

  inputDeviceName.value = inputDevices.value.length > 0 ? inputDevices.value[0] : undefined;
  outputDeviceName.value = outputDevices.value.length > 0 ? outputDevices.value[0] : undefined;
}

function test() {
  output.value?.sendAllSoundOff();
  output.value?.playNote("C3", { attack: 0.5, duration: 1000 });
  output.value?.playNote("D3", { attack: 0.5, duration: 1000, time: "+1000" });
  output.value?.playNote("E3", { attack: 0.5, duration: 1000, time: "+2000" });
}
</script>

<template>
  <label>
    Input Device
    <select v-model="inputDeviceName">
      <option v-for="device in inputDevices" :value="device">{{ device }}</option>
    </select>
  </label>
  <label>
    Output Device
    <select v-model="outputDeviceName">
      <option v-for="device in outputDevices" :value="device">{{ device }}</option>
    </select>
  </label>
  <button @click="test">Test</button>
</template>

<style scoped>

</style>
