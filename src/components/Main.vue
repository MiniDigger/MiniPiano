<script setup lang="ts">
import {
  BoundingBox,
  GraphicalLine,
  GraphicalNote,
  OpenSheetMusicDisplay as OSMD,
  OutlineAndFillStyleEnum,
  PointF2D
} from "opensheetmusicdisplay";
import { computed, onMounted, ref, useTemplateRef, watch } from "vue";
import MidiControls from "@/components/MidiControls.vue";
import { Utilities } from "webmidi";
import Options from "@/components/Options.vue";

const midiControls = useTemplateRef("midi");
const optionsRef = useTemplateRef("options");
const options = computed(() => optionsRef.value?.options);

const container = useTemplateRef("osmdContainer");
let osmd: OSMD | null = null;

const expectedNotes = ref<number[]>([]);
const expectedObjects = new Map<number, GraphicalNote>();
const playedNotes = ref<number[]>([]);

onMounted(async () => {
  osmd = new OSMD(container.value!, { autoResize: true, backend: "svg", followCursor: false });
});

watch(() => options.value?.selectedScore, async (newValue) => {
  if (!osmd || !newValue) return;
  console.log("new score selected", newValue);
  const [folder, fileName] = newValue.split("/");
  const file = (await import(`@/assets/${folder}/${fileName}.mxl`)).default;
  console.log("loading new score", file);
  await osmd.load(file);

  // hide voice for das veilchen // TODO do this automatically
  // osmd.Sheet.Instruments[0].Visible = false;

  reset();
});

watch(() => options.value?.jianpuMode, (newValue) => {
  if (!osmd) return;
  // TODO requires paid version
  osmd.EngravingRules.JianpuAlwaysUsed = newValue;

  reset();
});


function next() {
  // const oldElements = osmd?.cursor.GNotesUnderCursor();
  // oldElements?.forEach((elem) => elem.setColor("black", {}));

  osmd?.cursor.next();

  recordNextNotes();
}

function reset() {
  if (!osmd) return;

  if (!osmd.Sheet) {
    osmd.clear();
    return;
  }

  osmd.render();
  osmd.cursor.show();
  osmd.cursor.reset();

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
  if (!osmd) return;

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
    console.log("miss", identifier, number);
    if (options.value?.playNoteOnWrongNote) {
      playCurrent();
    }
    if (options.value?.highlightWrongNote) {
      // TODO somehow display the missed note?
      const notesUnderCursor = osmd.cursor.GNotesUnderCursor();
      for (const firstNote of notesUnderCursor) {
        if (firstNote.sourceNote.isRest()) {
          continue;
        }
        const firstMidiNote = firstNote.sourceNote.halfTone + 12;
        const actualMidiNote = number;
        // todo meh doesnt work, midi notes are not spaced equally, because of black keys
        const moveBy = (actualMidiNote - firstMidiNote) / 4;
        console.log("missed note", firstMidiNote, actualMidiNote, "move by", moveBy);
        drawCross(new PointF2D(firstNote.PositionAndShape.AbsolutePosition.x, firstNote.PositionAndShape.AbsolutePosition.y - moveBy));
        // osmd.Drawer.drawBoundingBox(elem.PositionAndShape, "red");
        break;
      }
    }
  }
}

function playCurrent() {
  midiControls.value?.getOutput()?.sendAllSoundOff();
  expectedNotes.value.forEach((note) => {
    midiControls.value?.getOutput()?.playNote(note, { duration: 500, attack: 0.5 });
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

  console.log("click", osmdPos);

  // drawCross(osmdPos);

  for (const measureEntry of osmd.GraphicSheet.MeasureList) {
    for (const graphicalMeasure of measureEntry) {
      if (graphicalMeasure.PositionAndShape.pointLiesInsideBorders(osmdPos)) {
        // osmd.Drawer.drawBoundingBox(graphicalMeasure.PositionAndShape, "red", true);

        const region = new BoundingBox(undefined);
        region.BorderLeft = osmdPos.x - 1;
        region.BorderTop = osmdPos.y - 1;
        region.BorderRight = osmdPos.x + 1;
        region.BorderBottom = osmdPos.y + 1;
        region.AbsolutePosition = new PointF2D(osmdPos.x, osmdPos.y);
        region.calculateAbsolutePosition();

        const objectsInRegion = graphicalMeasure.PositionAndShape.getObjectsInRegion(region, true, GraphicalNote.name);
        for (const note of objectsInRegion) {
          if (note instanceof GraphicalNote) {
            console.log(note, note.getNoteheadSVGs());
            // TODO for E5 G5 this colors both notes...
            note.setColor("orange", {});
            // osmd.Drawer.drawBoundingBox(note.PositionAndShape.Parent, "blue");

            const midiNote = note.sourceNote.halfTone + 12;
            console.log("clicked note", midiNote, Utilities.toNoteIdentifier(midiNote, 0));
            midiControls.value?.getOutput()?.playNote(midiNote, { duration: 500, attack: 0.6 });
          }
        }
      }
    }
  }
}

function drawCross(position: PointF2D, size = 2, color = OutlineAndFillStyleEnum.BaseWritingColor) {
  if (!osmd) return;
  osmd.Drawer.drawLineAsHorizontalRectangle(
    new GraphicalLine(new PointF2D(position.x - size / 2, position.y), new PointF2D(position.x + size / 2, position.y),
      0.1, color),
    0
  );
  osmd.Drawer.drawLineAsVerticalRectangle(
    new GraphicalLine(new PointF2D(position.x, position.y - size / 2), new PointF2D(position.x, position.y + size / 2),
      0.1, color),
    0
  );
}
</script>

<template>
  <div class="controls">
    <div>
      <MidiControls ref="midi" @noteOn="noteOn" />
      <div class="cursor-controls">
        <button @click="reset">reset</button>
        <button @click="next">next</button>
        <button @click="playCurrent">playCurrent</button>
        <button @click="next();playCurrent();">playNext</button>
        <button @click="dumpNotes">dump</button>
      </div>
    </div>
    <Options ref="options" />
  </div>
  <div ref="osmdContainer" @click="click" />
</template>

<style scoped lang="scss">
.controls {
  display: flex;
  justify-content: space-between;
  padding: 1rem;
}

.cursor-controls {
  display: flex;
  gap: 0.5rem;
}
</style>
