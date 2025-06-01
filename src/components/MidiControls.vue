<script setup lang="ts">
import { type NoteMessageEvent, WebMidi } from "webmidi";
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";

const inputDevices = ref<string[]>([]);
const outputDevices = ref<string[]>([]);
const inputDeviceName = ref<string | undefined>(undefined);
const outputDeviceName = ref<string | undefined>(undefined);
const input = computed(() => inputDeviceName.value ? WebMidi.getInputByName(inputDeviceName.value) : undefined);
const output = computed(() => outputDeviceName.value ? WebMidi.getOutputByName(outputDeviceName.value) : undefined);

const emit = defineEmits<{
  noteOn: [identifier: string, number: number][]
}>();

watch(input, async (newValue) => {
  console.log("new input device", newValue?.name);
  if (!newValue?.hasListener("noteon", onNote)) {
    newValue?.addListener("noteon", onNote);
  }
});
watch(output, async (newValue) => {
  console.log("new output device", newValue?.name);
});

function onNote(e: NoteMessageEvent) {
  if (e.port === input.value) {
    emit("noteOn", [e.note.identifier, e.note.number])
  }
}

onMounted(async () => {
  await WebMidi.enable();

  await reloadDevices();
  WebMidi.addListener("portschanged", reloadDevices);
});

onBeforeUnmount(() => WebMidi.disable());

async function reloadDevices() {
  await disconnect();

  inputDevices.value = WebMidi.inputs.map(input => input.name);
  outputDevices.value = WebMidi.outputs.map(output => output.name);

  inputDeviceName.value = inputDevices.value.length > 0 ? inputDevices.value[0] : undefined;
  outputDeviceName.value = outputDevices.value.length > 0 ? outputDevices.value[0] : undefined;
}

async function disconnect() {
  inputDeviceName.value = undefined;
  outputDeviceName.value = undefined;
}

defineExpose({
  getOutput: () => output.value,
  getInput: () => input.value,
  disconnect,
});
</script>

<template>
  <div class="midi-controls">
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
    <button @click="disconnect">Disconnect</button>
  </div>
</template>

<style scoped>
.midi-controls {
  display: flex;
  gap: 1rem;
}
</style>
