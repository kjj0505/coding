<script>
  import * as Tone from 'tone';
  import { writable, get } from 'svelte/store';
  import { tick, onMount } from 'svelte';

  const NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

  export const tuningNotes = writable(["E2", "A2", "D3", "G3", "B3", "E4"]);
  let tuning = ["E", "A", "D", "G", "B", "E"];
  let selectedTuning = "standard";
  let selectedGuitar = "acoustic";

  let tab = Array(6).fill().map(() => Array(59).fill(""));
  let currentTabPos = 0;
  let isMuted = false;
  const isPlaying = writable(false);

  let undoStack = [];

  let tabContainer;
  let timeSignature = "4/4";
  let editingTimeTop = false;
  let timeTopInput = "4";
  let timeBottomInput = "4";

  let saveName = "";
  let savedTabs = [];
  let showSavedTabs = false;
  let selectedTabs = [];

  let dragSource = null;
  let dragStart = null;
  let dragEnd = null;
  let selectedRange = null;
  let isDragging = false;
  let isSwapping = false;
  let isShiftDown = false;

  let selectedCell = null;
  let tuningSelectIndex = null;

  function selectTuningNote(stringIndex, note) {
    const baseOctaves = [2, 2, 3, 3, 4, 4]; // 6번~1번줄
    tuning[stringIndex] = note;
    const updated = [...get(tuningNotes)];
    updated[stringIndex] = `${note}${baseOctaves[stringIndex]}`;
    tuningNotes.set(updated);
    tuningSelectIndex = null;
  }

  loadSavedTabs();


  let rightDragStart = null;
  let rightDragEnd = null;
  let isRightDragging = false;

  const pressedKeys = new Set();


  onMount(() => {
  // 👉 우클릭 메뉴 방지 (마우스 오른쪽 클릭 시 context 메뉴 안 뜨게)
  window.addEventListener('contextmenu', e => e.preventDefault());

  // 👉 마우스 업 이벤트 (왼쪽 드래그 해제, 우클릭 드래그 지속음 처리)
  window.addEventListener('mouseup', (e) => {
  if (e.button === 2 && isRightDragging && rightDragStart && rightDragEnd) {
    const startRow = rightDragStart.stringIndex;
    const startCol = Math.min(rightDragStart.colIndex, rightDragEnd.colIndex);
    const endCol = Math.max(rightDragStart.colIndex, rightDragEnd.colIndex);

    const baseFret = tab[startRow][startCol];
    if (baseFret !== "") {
      saveHistory();
      tab[startRow][startCol] = baseFret;
      for (let j = startCol + 1; j <= endCol; j++) {
        tab[startRow][j] = "~";
      }
    }

    rightDragStart = null;
    rightDragEnd = null;
    isRightDragging = false;
  }

  isDragging = false;
  isSwapping = false;
});


  // 👉 키 다운 이벤트 처리
  window.addEventListener('keydown', (e) => {
    if (e.key === 'Shift') isShiftDown = true;

    if (e.key === 'Backspace') {
      e.preventDefault();
      if (selectedRange) {
        deleteRange(selectedRange);
        selectedRange = null;
      } else if (selectedCell) {
        deleteCell(selectedCell.stringIndex, selectedCell.colIndex);
        selectedCell = null;
      }
    }

    if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
      undo();
    }
  });

  window.addEventListener("keydown", async (e) => {
  if (hoveredFretIndex == null) return;

  const pressedNum = parseInt(e.key);
  if (pressedNum >= 1 && pressedNum <= 6 && !pressedKeys.has(pressedNum)) {
    pressedKeys.add(pressedNum);

    const stringIndex = 6 - pressedNum;
    const stringNote = get(tuningNotes)[stringIndex];
    const note = getNoteFromString(stringNote, hoveredFretIndex);

    if (!isMuted) {
      play(note);
    }

    saveHistory();

    const col = findNextAvailableCol(0, 2);
    if (col + 1 >= tab[0].length) {
      tab = tab.map(row => [...row, "", ""]);
      await tick();
      if (tabContainer) tabContainer.scrollLeft = tabContainer.scrollWidth;
    }

    tab[stringIndex][col] = (hoveredFretIndex + 1).toString();
  }
});

  window.addEventListener("keyup", (e) => {
  if (e.key === 'Shift') isShiftDown = false;

  const releasedNum = parseInt(e.key);
  if (pressedKeys.has(releasedNum)) {
    pressedKeys.delete(releasedNum);
  }
});

  // 👉 키 업 이벤트 (Shift 키 상태 해제)
  window.addEventListener('keyup', (e) => {
    if (e.key === 'Shift') isShiftDown = false;
  });

  // 👉 튜닝 노트 선택 닫기 처리 (바깥 클릭 시)
  window.addEventListener('click', (e) => {
    const el = e.target;
    if (!(el instanceof Element)) return;
    const insideLabel = el.closest('.string-label');
    const insideDropdown = el.closest('.note-selector');
    if (!insideLabel && !insideDropdown) {
      tuningSelectIndex = null;
    }
  });
});


  function saveHistory() {
    const snapshot = tab.map(row => [...row]);
    undoStack.push(snapshot);
    if (undoStack.length > 100) undoStack.shift();
  }

  function undo() {
    if (undoStack.length === 0) return;
    tab = undoStack.pop();
  }

function handleDragStart(i, j) {
  if (isShiftDown) {
    // Shift 드래그는 빈칸이어도 허용
    dragStart = { stringIndex: i, colIndex: j };
    isSwapping = false;
  } else {
    // 일반 드래그는 음이 있을 때만 허용
    if (tab[i][j] === "") return;
    dragSource = { stringIndex: i, colIndex: j };
    isSwapping = true;
  }
  isDragging = true;
}

  function handleDrop(i, j) {
    if (!isDragging) return;

    if (isShiftDown && dragStart) {
      dragEnd = { stringIndex: i, colIndex: j };
      const startRow = Math.min(dragStart.stringIndex, dragEnd.stringIndex);
      const endRow = Math.max(dragStart.stringIndex, dragEnd.stringIndex);
      const startCol = Math.min(dragStart.colIndex, dragEnd.colIndex);
      const endCol = Math.max(dragStart.colIndex, dragEnd.colIndex);

      selectedRange = {
        start: { stringIndex: startRow, colIndex: startCol },
        end: { stringIndex: endRow, colIndex: endCol }
      };

      dragStart = null;
      dragEnd = null;

    } else if (!isShiftDown && dragSource) {
      saveHistory();
      const temp = tab[i][j];
      tab[i][j] = tab[dragSource.stringIndex][dragSource.colIndex];
      tab[dragSource.stringIndex][dragSource.colIndex] = temp;
      dragSource = null;
    }

    isDragging = false;
    isSwapping = false;
  }

  function deleteRange(range) {
    saveHistory();
    for (let i = range.start.stringIndex; i <= range.end.stringIndex; i++) {
      for (let j = range.start.colIndex; j <= range.end.colIndex; j++) {
        tab[i][j] = "";
      }
    }
  }

  function setGuitar(type) {
    selectedGuitar = type;
  }

  function setTuning(mode) {
    selectedTuning = mode;
    if (mode === "standard") {
      tuning = ["E", "A", "D", "G", "B", "E"];
      tuningNotes.set(["E2", "A2", "D3", "G3", "B3", "E4"]);
    } else if (mode === "drop d") {
      tuning = ["D", "A", "D", "G", "B", "E"];
      tuningNotes.set(["D2", "A2", "D3", "G3", "B3", "E4"]);
    } else if (mode === "dadgad") {
      tuning = ["D", "A", "D", "G", "A", "D"];
      tuningNotes.set(["D2", "A2", "D3", "G3", "A3", "D4"]);
    } else if (mode === "open g") {
      tuning = ["D", "G", "D", "G", "B", "D"];
      tuningNotes.set(["D2", "G2", "D3", "G3", "B3", "D4"]);
    }
  }
  function getNoteFromString(stringNote, fret) {
    const match = stringNote.match(/^([A-G]#?)(\d+)$/);
    if (!match) return "";
    const [, baseNote, octaveStr] = match;
    const baseIndex = NOTES.indexOf(baseNote);
    const baseOctave = parseInt(octaveStr);
    const totalSemitone = baseIndex + fret;
    const note = NOTES[totalSemitone % 12];
    const octave = baseOctave + Math.floor(totalSemitone / 12);
    return `${note}${octave}`;
  }

  function play(note, duration = 0.3) {
  if (isMuted) return;
  Tone.start();
  const synth = new Tone.Synth().toDestination();
  synth.triggerAttackRelease(note, duration); // duration: 초 단위
}

  function resetTab() {
    saveHistory();
    tab = Array(6).fill().map(() => Array(59).fill(""));
    currentTabPos = 0;
    if (tabContainer) tabContainer.scrollLeft = 0;
  }

  function getLastActiveColumn() {
    for (let col = tab[0].length - 1; col >= 0; col--) {
      for (let i = 0; i < 6; i++) {
        if (tab[i][col] !== "") return col;
      }
    }
    return -1;
  }

  async function playTab() {
  if (get(isPlaying)) return;
  const lastCol = getLastActiveColumn();
  if (lastCol === -1) return;
  isPlaying.set(true);
  await Tone.start();

  let delayMs = 300;
  if (timeSignature === "3/4") delayMs = 400;
  else if (timeSignature === "6/8") delayMs = 200;
  else if (timeSignature === "2/4") delayMs = 350;
  else if (timeSignature === "5/4") delayMs = 300;

  try {
    for (let col = 0; col <= lastCol; col++) {
      if (!get(isPlaying)) break;

      for (let stringIndex = 0; stringIndex < 6; stringIndex++) {
        const cell = tab[stringIndex][col];

        // 시작 음이면 재생하고 ~ 개수만큼 sustain 계산
        if (cell !== "" && cell !== "~" && !isNaN(cell)) {
  let sustain = 1;
  let lookahead = col + 1;
  while (lookahead <= lastCol && tab[stringIndex][lookahead] === "~") {
    sustain++;
    lookahead++;
  }
  const stringNote = get(tuningNotes)[stringIndex];
  const note = getNoteFromString(stringNote, parseInt(cell));
  play(note, (sustain * delayMs) / 1000);
}

      }

      await new Promise(res => setTimeout(res, delayMs));
    }
  } finally {
    isPlaying.set(false);
  }
}


  function pauseTab() {
    isPlaying.set(false);
  }

  async function handleFretClick(stringIndex, fretIndex) {
    if (stringIndex === 6) return;
    saveHistory();

    const col = findNextAvailableCol(0, 2);

    if (col + 1 >= tab[0].length) {
      tab = tab.map(row => [...row, "", ""]);
      await tick();
      if (tabContainer) tabContainer.scrollLeft = tabContainer.scrollWidth;
    }

    const stringNote = get(tuningNotes)[stringIndex];
    const note = getNoteFromString(stringNote, fretIndex);
    play(note);
    tab[stringIndex][col] = (fretIndex + 1).toString();
  }

  function deleteCell(stringIndex, colIndex) {
  saveHistory();

  const value = tab[stringIndex][colIndex];

  // ① 현재 셀이 프렛 번호면 → 오른쪽 ~들 삭제
  if (value !== "" && !isNaN(value)) {
    let i = colIndex + 1;
    while (i < tab[stringIndex].length && tab[stringIndex][i] === "~") {
      tab[stringIndex][i] = "";
      i++;
    }
    tab[stringIndex][colIndex] = "";
    return;
  }

  // ② 현재 셀이 ~이면 → 왼쪽 숫자까지 포함해 모두 삭제
  if (value === "~") {
    // 왼쪽으로 숫자 찾기
    let start = colIndex - 1;
    while (start >= 0 && tab[stringIndex][start] === "~") {
      start--;
    }

    // 숫자일 경우 삭제 처리
    if (start >= 0 && !isNaN(tab[stringIndex][start])) {
      tab[stringIndex][start] = "";
      for (let i = start + 1; i < tab[stringIndex].length && tab[stringIndex][i] === "~"; i++) {
        tab[stringIndex][i] = "";
      }
    }

    // 현재 ~도 삭제
    tab[stringIndex][colIndex] = "";
    return;
  }

  // ③ 일반 삭제
  tab[stringIndex][colIndex] = "";
}


  function selectCell(stringIndex, colIndex) {
    selectedCell = { stringIndex, colIndex };
  }

  function toggleTuner() {
    showTuningOptions = !showTuningOptions;
  }

  function toggleTuner2() {
    showTuningOptions2 = !showTuningOptions2;
  }

  function toggleSavedTabs() {
    showSavedTabs = !showSavedTabs;
    if (!showSavedTabs) selectedTabs = [];
  }

  function formatFret(cell) {
  return cell === "" ? "" : cell;
}


  function hasAnyNotes() {
    return tab.some(row => row.some(cell => cell !== ""));
  }

  function startEditTop() {
    editingTimeTop = true;
    timeTopInput = timeSignature.split('/')[0];
  }

  function applyTimeSignature() {
    if (!/^\d+$/.test(timeTopInput)) return;
    if (!["2", "4", "8", "16"].includes(timeBottomInput)) return;
    timeSignature = `${timeTopInput}/${timeBottomInput}`;
    editingTimeTop = false;
  }

  function onTimeTopKeydown(e) {
    if (e.key === "Enter") {
      if (timeTopInput === "") return;
      applyTimeSignature();
    } else if (e.key === "Escape") {
      editingTimeTop = false;
    }
  }

  function onTimeBottomChange(e) {
    timeBottomInput = e.target.value;
    applyTimeSignature();
  }

  function saveTab() {
    if (!saveName) {
      alert("저장할 이름을 입력하세요!");
      return;
    }

    if (localStorage.getItem(`tab_${saveName}`)) {
      alert(`같은 이름 '${saveName}' 의 악보가 이미 존재합니다.\n다른 이름을 입력해주세요.`);
      return;
    }

    const tabData = {
      tab,
      tuningNotes: get(tuningNotes),
      tuning,
      selectedTuning,
      selectedGuitar,
      timeSignature
    };
    localStorage.setItem(`tab_${saveName}`, JSON.stringify(tabData));
    loadSavedTabs();
    alert(`'${saveName}'으로 저장되었습니다.`);
    saveName = "";
  }

  function loadSavedTabs() {
    savedTabs = [];
    for (let key in localStorage) {
      if (key.startsWith("tab_")) {
        savedTabs.push(key.replace("tab_", ""));
      }
    }
  }

  function loadTab(name) {
    const json = localStorage.getItem(`tab_${name}`);
    if (json) {
      const data = JSON.parse(json);
      saveHistory();
      tab = data.tab;
      tuningNotes.set(data.tuningNotes);
      tuning = data.tuning;
      selectedTuning = data.selectedTuning;
      selectedGuitar = data.selectedGuitar;
      timeSignature = data.timeSignature;
      currentTabPos = getLastActiveColumn() + 2;
    }
    showSavedTabs = false;
    selectedTabs = [];
  }

  function toggleTabSelection(name) {
    if (selectedTabs.includes(name)) {
      selectedTabs = selectedTabs.filter(n => n !== name);
    } else {
      selectedTabs = [...selectedTabs, name];
    }
  }

  function deleteSelectedTabs() {
    if (selectedTabs.length === 0) {
      alert("삭제할 악보를 선택하세요.");
      return;
    }
    const really = confirm(`선택한 ${selectedTabs.length}개의 악보를 정말 삭제하시겠습니까?`);
    if (!really) return;

    selectedTabs.forEach(name => {
      localStorage.removeItem(`tab_${name}`);
    });
    savedTabs = savedTabs.filter(name => !selectedTabs.includes(name));
    selectedTabs = [];
    alert("삭제 완료!");
  }

  function isLowerString(i) {
    return i >= 3;
  }

  function findNextAvailableCol(start = 0, step = 2) {
    const maxCol = tab[0].length;
    for (let col = start; col < maxCol; col += step) {
      let isEmpty = true;
      for (let stringIndex = 0; stringIndex < 6; stringIndex++) {
        if (tab[stringIndex][col] !== "") {
          isEmpty = false;
          break;
        }
      }
      if (isEmpty) return col;
    }
    return maxCol;
  }

  let showTuningOptions = false;
  let showTuningOptions2 = false;


  let hoveredFret = null;
  let hoveredFretIndex = null; 


  let showChordBox = false;

const chords = {
  C: { frets: ["x", 3, 2, 0, 1, 0] },
  G: { frets: [3, 2, 0, 0, 0, 3] },
  Am: { frets: ["x", 0, 2, 2, 1, 0] },
  F: { frets: [1, 3, 3, 2, 1, 1] },
  D: { frets: ["x", "x", 0, 2, 3, 2] },
  E: { frets: [0, 2, 2, 1, 0, 0] }
};

function toggleChordBox() {
  showChordBox = !showChordBox;
}

function applyChord(chordName) {
  const chord = chords[chordName];
  if (!chord) return;

  saveHistory();

  const col = findNextAvailableCol(0, 2);
  if (col + 1 >= tab[0].length) {
    tab = tab.map(row => [...row, "", ""]);
  }

  // 각 줄에 프렛 번호 입력
  for (let i = 0; i < 6; i++) {
    const fret = chord.frets[i];
    tab[i][col] = fret === "x" ? "" : fret.toString();
  }

  // 동시에 코드 소리 재생
  playChord(chordName);

}



// 코드 UI toggle 함수

// 코드 클릭 시 호출: 악보에 입력 + 소리 재생

// 코드 구성음을 동시에 재생
function playChord(chordName, duration = 0.6) {
  const chord = chords[chordName];
  if (!chord) return;

  Tone.start();
  const now = Tone.now();
  const synth = new Tone.PolySynth().toDestination();

  chord.frets.forEach((fret, stringIndex) => {
    if (fret === "x") return;

    const stringNote = get(tuningNotes)[stringIndex];
    const note = getNoteFromString(stringNote, fret);
    if (!isMuted) {
      synth.triggerAttackRelease(note, duration, now);
    }
  });
}






  



</script>




<style>
  .chord-box {
  display: grid;
  grid-template-columns: repeat(3, auto);
  gap: 6px;
  background: white;
  padding: 10px;
  border: 1px solid #aaa;
  border-radius: 6px;
  box-shadow: 2px 2px 6px rgba(0,0,0,0.2);
  margin-left: 10px;
}
.chord-button {
  padding: 6px 10px;
  background: #eee;
  border: 1px solid #999;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}
.chord-button:hover {
  background: #ccc;
}
  .container { height: 100%; width: 100%; display: flex; flex-direction: column; background: white; margin: 0; padding: 0; font-family: sans-serif; }
  .top-bar { display: flex; justify-content: space-between; padding: 1rem; }
  .tab-display { background: #ccc; padding: 1rem; font-family: 'Courier New', monospace; font-size: 14px; overflow-x: auto; overflow-y: hidden; white-space: pre; box-sizing: border-box; max-width: 100%; }
  .controls { display: flex; gap: 0.5rem; padding: 0.5rem 1rem; align-items: center; }
  .fretboard-wrapper { flex: 1; display: flex; overflow: auto; align-items: stretch; }
  .string-labels { display: flex; flex-direction: column; justify-content: center; gap: 24px; }
  .string-label { background: red; color: white; padding: 0.1rem 0.4rem; border-radius: 4px; text-align: center; font-weight: bold; margin-right: 5px; }
  .tunerset { display: flex; }
  .tunerset > button { margin: 8px; }
  .tunerset > button.selected { background: #666; color: white; font-weight: bold; }
  .fretboard { flex: 1; display: grid; grid-template-columns: repeat(16, 1fr); grid-template-rows: repeat(7, 1fr); gap: 3px; background: #999; height: 350px; }
  .fret { position: relative; background: #ccc; border: 1px solid #aaa; }
  .dot { position: absolute; top: 50%; left: 50%; width: 20px; height: 16px; background: black; border-radius: 50%; transform: translate(-50%, -50%); }
  .click-zone { position: absolute; bottom: -10px; left: 0; width: 100%; height: 40%; cursor: pointer; }
  button { padding: 0.5rem 1rem; border: none; border-radius: 10px; background: #ddd; }
  button:disabled { opacity: 0.5; cursor: not-allowed; }
  .footer { display: flex; justify-content: space-between; }
  .tab-table { border-collapse: collapse; font-family: 'Courier New', monospace; font-size: 14px; table-layout: auto; width: max-content; }
  .tab-table td { min-width: 24px; height: 24px; line-height: 24px; position: relative; text-align: center; padding: 0; vertical-align: middle; }
  .tab-table .time-signature-cell { height: 24px; vertical-align: middle; }
  .tab-table td::after { content: ""; position: absolute; bottom: 50%; left: 0; width: 100%; height: 1px; background: black; transform: translateY(50%); }
  .tab-table td:not(:empty)::after { display: none; }
  .tab-table td.is-dragging { background: rgba(0,0,0,0.03); outline: 1px dashed #0d0d0d; }
  .tab-table td.is-dragging::after { background: rgba(0,0,0,0.2); }
  .time-sig-inline { display: inline-flex; align-items: center; gap: 2px; height: 100%; }
  .time-sig-text, .time-sig-slash { font-size: 12px; line-height: 12px; }
  .time-top-input { width: 20px; font-size: 12px; height: 12px; line-height: 12px; border: none; border-bottom: 1px solid black; text-align: center; background: transparent; padding: 0; margin: 0; }
  .time-top-input:focus { outline: none; }
  .time-bottom-select { font-size: 12px; line-height: 12px; height: 18px; padding: 0; margin: 0; border: 1px solid black; background: white; }
  .saved-tabs { padding: 1rem; background: #eee; max-height: 200px; overflow-y: auto; }
  .saved-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 8px; }
  .saved-item { padding: 8px; border: 1px solid #aaa; border-radius: 6px; cursor: pointer; text-align: center; }
  .saved-item.selected { background: #9f9e9e; color: #fff; font-weight: bold; }
  .tab-table td.selected {
  background: rgba(0, 0, 0, 0.15);
}
  .note-selector {
  position: absolute;
  left: 110%;
  top: 0;
  background: white;
  border: 1px solid #aaa;
  border-radius: 6px;
  z-index: 10;
  box-shadow: 2px 2px 5px rgba(0,0,0,0.2);
  display: grid;
  grid-template-columns: repeat(3, auto); /* 3열로 정렬 */
  gap: 4px;
  padding: 6px;
}

.note-option {
  padding: 4px 8px;
  cursor: pointer;
  white-space: nowrap;
  text-align: center;
  min-width: 30px;
}

.note-option:hover {
  background: #f0f0f0;
}

.note-selector.above {
  top: auto;
  bottom: -152px; 
  transform: translateY(-100%);
}

.disabled-note {  
  background: #ddd;
  color: #777;
  font-weight: bold;
  pointer-events: none;
}
</style>

<div class="container"> 
  <div class="top-bar">
    <button>guitar site</button>
    <div>
      <button on:click={resetTab}>reset</button>
      <input placeholder="이름" bind:value={saveName} />
      <button on:click={saveTab}>save</button>
      <button on:click={toggleSavedTabs}>saved</button>
      <button on:click={pauseTab} disabled={!$isPlaying}>pause</button>
      <button on:click={playTab} disabled={$isPlaying || !hasAnyNotes()}>play</button>
    </div>
  </div>

  {#if showSavedTabs}
    <div class="saved-tabs">
      <button style="margin-bottom: 8px;" on:click={deleteSelectedTabs}>Delete</button>
      <div class="saved-grid">
        {#each savedTabs as name}
          <div
            class="saved-item"
            class:selected={selectedTabs.includes(name)}
            on:click={() => toggleTabSelection(name)}
            on:dblclick={() => loadTab(name)}
          >
            {name} {selectedTabs.includes(name) ? "✅" : ""}
          </div>
        {/each}
      </div>
    </div>
  {/if}

  <div class="tab-display" bind:this={tabContainer}>
    <table class="tab-table" spellcheck="false">
      <tbody>
        {#each tab as line, i}
          <tr>
            <td class="time-signature-cell">
              {#if i === 0}
                <span class="time-sig-inline">
                  {#if editingTimeTop}
                    <input
                      class="time-top-input"
                      bind:value={timeTopInput}
                      on:keydown={onTimeTopKeydown}
                      on:blur={applyTimeSignature}
                      maxlength="2"
                      autofocus
                    />
                  {:else}
                    <span class="time-sig-text" on:click={startEditTop}>{timeTopInput}</span>
                  {/if}
                  <span class="time-sig-slash">/</span>
                  <select
                    class="time-bottom-select"
                    bind:value={timeBottomInput}
                    on:change={onTimeBottomChange}
                  >
                    <option value="2">2</option>
                    <option value="4">4</option>
                    <option value="8">8</option>
                    <option value="16">16</option>
                  </select>
                </span>
              {:else}
                &nbsp;
              {/if}
            </td>
            {#each line as fret, j}
              <td
  draggable={true}
  class:is-dragging={isDragging && isSwapping}
  class:selected={selectedRange &&
    i >= selectedRange.start.stringIndex &&
    i <= selectedRange.end.stringIndex &&
    j >= selectedRange.start.colIndex &&
    j <= selectedRange.end.colIndex
  }
  on:dragstart={() => handleDragStart(i, j)}
  on:dragover|preventDefault
  on:drop={() => handleDrop(i, j)}
  on:dragend={() => isDragging = false}
  on:click={() => selectCell(i, j)}
  on:mousedown={(e) => {
    if (e.button === 2) {
      rightDragStart = { stringIndex: i, colIndex: j };
      rightDragEnd = { stringIndex: i, colIndex: j };
      isRightDragging = true;
    }
  }}
  on:mouseenter={(e) => {
    if (isRightDragging && e.buttons === 2) {
      rightDragEnd = { stringIndex: i, colIndex: j };
    }
  }}
>
  {@html formatFret(fret)}
</td>
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>
  </div>

  <div class="controls">
  <!-- 1) tuner 버튼 -->
  <button on:click={toggleTuner} class="setting">tuner</button>

  <!-- 2) 튜닝 옵션: tuner 바로 다음에 배치 -->
  {#if showTuningOptions}
    <div class="tunerset">
      <button class:selected={selectedTuning === "standard"} on:click={() => setTuning("standard")}>standard</button>
      <button class:selected={selectedTuning === "drop d"} on:click={() => setTuning("drop d")}>drop d</button>
      <button class:selected={selectedTuning === "dadgad"} on:click={() => setTuning("dadgad")}>DADGAD</button>
      <button class:selected={selectedTuning === "open g"} on:click={() => setTuning("open g")}>Open G</button>
    </div>
  {/if}

  <!-- 3) code 버튼 -->
  <button on:click={toggleChordBox} class="setting">code</button>

  <!-- 4) 코드 목록: code 바로 다음에 배치 -->
  {#if showChordBox}
    <div class="tunerset chord-set">
      <button on:click={() => applyChord("C")}>C</button>
      <button on:click={() => applyChord("G")}>G</button>
      <button on:click={() => applyChord("Am")}>Am</button>
      <button on:click={() => applyChord("F")}>F</button>
      <button on:click={() => applyChord("D")}>D</button>
      <button on:click={() => applyChord("E")}>E</button>
    </div>
  {/if}
</div>


  <div class="fretboard-wrapper">
    <div class="string-labels">
      {#each tuning as t, i}
  <div style="position: relative;">
    <div class="string-label" on:click={() => tuningSelectIndex = tuningSelectIndex === i ? null : i}>{t}</div>
    {#if tuningSelectIndex === i}
  <div class="note-selector {isLowerString(i) ? 'above' : ''}">
    {#each NOTES as note}
  <div
    class="note-option {note === tuning[i] ? 'disabled-note' : ''}"
    on:click={() => note !== tuning[i] && selectTuningNote(i, note)}
  >
    {note}
  </div>
{/each}
  </div>
{/if}
  </div>
{/each}

    </div>
    <div class="fretboard">
      {#each Array(7) as _, stringIndex}
  {#each Array(16) as _, fretIndex}
    <div
      class="fret"
      on:mouseenter={() => {
        hoveredFretIndex = fretIndex;
        hoveredFret = stringIndex;
      }}
      on:mouseleave={() => {
        hoveredFretIndex = null;
        hoveredFret = null;
      }}
    >
            {#if stringIndex < 6}
              <div class="click-zone" on:click={() => handleFretClick(stringIndex, fretIndex)}></div>
            {/if}
            {#if [2, 4, 6, 8, 14].includes(fretIndex) && stringIndex === 3}
              <div class="dot"></div>
            {:else if fretIndex === 11 && (stringIndex === 2 || stringIndex === 4)}
              <div class="dot"></div>
            {/if}
          </div>
        {/each}
      {/each}
    </div>
  </div>

  <div class="footer">
    <div class="controls">
      <button on:click={toggleTuner2} class="setting">guitar</button>
      {#if showTuningOptions2}
        <div class="tunerset">
          <button class:selected={selectedGuitar === 'acoustic'} on:click={() => setGuitar('acoustic')}>acoustic guitar</button>
          <button class:selected={selectedGuitar === 'electric'} on:click={() => setGuitar('electric')}>electric guitar</button>
        </div>
      {/if}
    </div>
    <div>
      <button style="border-radius: 50%;" on:click={() => isMuted = !isMuted}>
        {#if isMuted} 🔇 {:else} 🔊 {/if}
      </button>
    </div>
  </div>
</div>
