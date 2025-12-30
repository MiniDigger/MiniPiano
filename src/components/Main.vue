<script setup lang="ts">
import {
  BoundingBox, GraphicalNote,
  OpenSheetMusicDisplay as OSMD,
  PointF2D, VexFlowMeasure
} from "opensheetmusicdisplay";
import { onMounted, ref, useTemplateRef } from "vue";
import MidiControls from "@/components/MidiControls.vue";
import { Utilities } from "webmidi";

const midiControls = useTemplateRef("midi");

const container = useTemplateRef("osmdContainer");
let osmd: OSMD | null = null;

const expectedNotes = ref<number[]>([]);
const expectedObjects = new Map<number, GraphicalNote>();
const playedNotes = ref<number[]>([]);

onMounted(async () => {
  osmd = new OSMD(container.value!, { autoResize: true, backend: "svg", followCursor: false });
  // const file = (await import("@/assets/samples/mozart-das_veilchen.xml")).default;
  const file = (await import("@/assets/samples/bach-prelude_1.mxl")).default;
  console.log("file", file);
  await osmd.load(file);
  // hide voice for das veilchen // TODO do this automatically
  // osmd.Sheet.Instruments[0].Visible = false;

  reset();
});

function next() {
  // const oldElements = osmd?.cursor.GNotesUnderCursor();
  // oldElements?.forEach((elem) => elem.setColor("black", {}));

  osmd?.cursor.next();

  recordNextNotes();
}

function reset() {
  osmd?.render();
  osmd?.cursor?.show();
  osmd?.cursor?.reset();

  recordNextNotes();
}

function recordNextNotes() {
  expectedNotes.value = [];
  playedNotes.value = [];

  const newElements = osmd?.cursor.GNotesUnderCursor();
  newElements?.forEach((elem) => {
    elem.setColor("blue", {});
    if (elem.sourceNote.isRest()) {
      return;
    }
    const note = elem.sourceNote.halfTone + 12;
    console.log("play", note, Utilities.toNoteIdentifier(note, 0));
    expectedNotes.value.push(note);
    expectedObjects.set(note, elem);
  });

  // TODO if expected is all pauses, we need to auto continue somehow
  // like wait for a short time and then call next()
}

function noteOn([identifier, number]: [string, number]) {
  if (expectedNotes.value.includes(number)) {
    expectedObjects.get(number)?.setColor("green", {});
    console.log("hit", identifier, number);
    if (!playedNotes.value.includes(number)) {
      playedNotes.value.push(number);
    }
    if (playedNotes.value.length == expectedNotes.value.length) {
      console.log("next!");
      next();
    }
  } else {
    // TODO somehow display the missed note?
    console.log("miss", identifier, number);
    playCurrent();
  }
}

function playCurrent() {
  midiControls.value?.getOutput()?.sendAllSoundOff();
  expectedNotes.value.forEach((note) => {
    midiControls.value?.getOutput()?.playNote(note, { duration: 500, attack: 0.6 });
  });
}

// TODO use this for a playback button
function dumpNotes() {
  const allNotes = [];
  osmd!.cursor.reset();
  const iterator = osmd!.cursor.Iterator;

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

function click(e: MouseEvent) {
  if (!osmd) return;

  const svgPos = osmd.GraphicSheet.domToSvg(new PointF2D(e.pageX, e.pageY));
  const osmdPos = osmd.GraphicSheet.svgToOsmd(svgPos);

  console.log(osmd);
  console.log("click", osmdPos);


  // osmd.Drawer.drawLineAsHorizontalRectangle(new GraphicalLine(new PointF2D(osmdPos.x - 1, osmdPos.y), new PointF2D(osmdPos.x + 1, osmdPos.y), 0.1, OutlineAndFillStyleEnum.BaseWritingColor, "blue"), 0);
  // osmd.Drawer.drawLineAsVerticalRectangle(new GraphicalLine(new PointF2D(osmdPos.x, osmdPos.y - 1), new PointF2D(osmdPos.x, osmdPos.y + 1), 0.1, OutlineAndFillStyleEnum.BaseWritingColor, "blue"), 0);


  for (let measureEntry of osmd.GraphicSheet.MeasureList) {
    for (let graphicalMeasure of measureEntry) {
      if (graphicalMeasure.PositionAndShape.pointLiesInsideBorders(osmdPos)) {
        osmd.Drawer.drawBoundingBox(graphicalMeasure.PositionAndShape, "red", true);

        const region = new BoundingBox(undefined);
        region.BorderLeft = osmdPos.x - 1;
        region.BorderTop = osmdPos.y - 1;
        region.BorderRight = osmdPos.x + 1;
        region.BorderBottom = osmdPos.y + 1;
        region.AbsolutePosition = new PointF2D(osmdPos.x, osmdPos.y);
        region.calculateAbsolutePosition();

        const objectsInRegion = graphicalMeasure.PositionAndShape.getObjectsInRegion(region, true, GraphicalNote.name);
        for (let note of objectsInRegion) {
          if (note instanceof GraphicalNote) {
            note.setColor("orange", {});
            // osmd.Drawer.drawBoundingBox(note.PositionAndShape.Parent, "blue");

            const midiNote = note.sourceNote.halfTone + 12;
            console.log("clicked note", midiNote, Utilities.toNoteIdentifier(midiNote, 0));
            midiControls.value?.getOutput()?.playNote(midiNote, { duration: 500, attack: 0.6 });
          }
        }

        // TODO for testing, add a random note to the measure
        const vexMeasure = (graphicalMeasure as VexFlowMeasure);
      }
    }
  }
}
</script>

<template>
  <MidiControls ref="midi" @noteOn="noteOn" />
  <button @click="reset">reset</button>
  <button @click="next">next</button>
  <button @click="playCurrent">playCurrent</button>
  <button @click="next();playCurrent();">playNext</button>
  <button @click="dumpNotes">dump</button>
  <div ref="osmdContainer" @click="click" />
</template>

<style scoped>

</style>
