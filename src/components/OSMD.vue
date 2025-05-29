<script setup lang="ts">
import { OpenSheetMusicDisplay as OSMD, VexFlowGraphicalNote } from "opensheetmusicdisplay";
import { onMounted, ref, useTemplateRef } from "vue";

const container = useTemplateRef("osmdContainer");
const osmd = ref<OSMD | null>(null);
onMounted(async () => {
  osmd.value = new OSMD(container.value!, { autoResize: true, backend: "svg", followCursor: false });
  const file = (await import("@/assets/mozart-das_veilchen.xml")).default;
  console.log("file", file);
  await osmd.value.load(file, "Das Veilchen");
  await osmd.value.render();
  osmd.value.cursor.show();
  window.osmd = osmd.value;
});

function next() {
  const oldElements = osmd.value?.cursor.GNotesUnderCursor();
  oldElements?.forEach((elem) => color(elem as VexFlowGraphicalNote, "black"));

  osmd.value?.cursor.next();

  const newElements = osmd.value?.cursor.GNotesUnderCursor();
  newElements?.forEach((elem) => color(elem as VexFlowGraphicalNote, "red"));
}

function color(gNote: VexFlowGraphicalNote, color: string) {
  const svg = gNote.getSVGGElement(); // stavenote
  for (const noteHead of svg.getElementsByClassName("vf-notehead")) {
    (noteHead.children[0] as SVGGElement).style.fill = color;
  }
  const stem = gNote.getStemSVG();
  if (stem) {
    if (stem.children.length) {
      (stem.children[0] as SVGGElement).style.stroke = color;
    } else {
      stem.style.stroke = color;
    }
  }
  // for (const beam of gNote.getBeamSVGs()) {
  //   if (beam.children.length) {
  //     beam.children[0].style.fill = color
  //   } else {
  //     beam.style.fill = color
  //   }
  // }
}

function dumpNotes() {
  const allNotes = [];
  osmd.value!.cursor.reset();
  const iterator = osmd.value!.cursor.Iterator;

  while (!iterator.EndReached) {
    const voices = iterator.CurrentVoiceEntries;
    for (let i = 0; i < voices.length; i++) {
      const v = voices[i];
      const notes = v.Notes;
      for (let j = 0; j < notes.length; j++) {
        const note = notes[j];
        // make sure our note is not silent
        if (note != null && note.halfTone != 0 && !note.isRest()) {
          allNotes.push({
            "note": note.halfTone + 12, // see issue #224
            "time": iterator.currentTimeStamp.RealValue * 4
          });
        }
      }
    }
    iterator.moveToNext();
  }
  console.log(allNotes);
}
</script>

<template>
  <button @click="next">next</button>
  <button @click="dumpNotes">dump</button>
  <div ref="osmdContainer" />
</template>

<style scoped>

</style>
