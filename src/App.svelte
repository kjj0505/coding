<script>
  import * as Tone from 'tone';
  import { writable, get } from 'svelte/store';
  import { tick, onMount } from 'svelte';

  const NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

  let tuningNotes = ["E4", "B3", "G3", "D3", "A2", "E2"];
  let tuning = ["E", "B", "G", "D", "A", "E"];
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
  let timeTopInput = timeSignature.split('/')[0];
  let timeBottomInput = timeSignature.split('/')[1];

  let saveName = "";
  let savedTabs = [];
  let showSavedTabs = false;
  let selectedTabs = [];

  let dragSource = null;
  let isDragging = false;

  let selectedCell = null;

  let tuningSelectIndex = null; // 현재 선택된 튜닝 인덱스

  function selectTuningNote(stringIndex, note) {
    const baseOctaves = [4, 3, 3, 3, 2, 2];
    tuning[stringIndex] = note;
    tuningNotes[stringIndex] = `${note}${baseOctaves[stringIndex]}`;
    tuningSelectIndex = null; // 드롭다운 닫기
  } 

  loadSavedTabs();

  onMount(() => {
  // 드래그 중지
  window.addEventListener('mouseup', () => {
    isDragging = false;
  });

  // 키보드 단축키 처리
  window.addEventListener('keydown', (e) => {
    if (e.key === 'Backspace') {
      if (selectedCell) {
        e.preventDefault();
        deleteCell(selectedCell.stringIndex, selectedCell.colIndex);
        selectedCell = null;
      }
    }

    if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
      undo();
    }
  });

  // ✅ 드롭다운 외부 클릭 시 닫기
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

  function handleDragStart(stringIndex, colIndex) {
    dragSource = { stringIndex, colIndex };
    isDragging = true;
  }

  function handleDrop(targetString, targetCol) {
    if (!dragSource) return;
    saveHistory();
    const temp = tab[targetString][targetCol];
    tab[targetString][targetCol] = tab[dragSource.stringIndex][dragSource.colIndex];
    tab[dragSource.stringIndex][dragSource.colIndex] = temp;
    dragSource = null;
    isDragging = false;
  }

  function handleDragEnd() {
    isDragging = false;
    dragSource = null;
  }

  function setGuitar(type) {
    selectedGuitar = type;
  }

  function setTuning(mode) {
    selectedTuning = mode;
    if (mode === "standard") {
      tuningNotes = ["E4", "B3", "G3", "D3", "A2", "E2"];
      tuning = ["E", "B", "G", "D", "A", "E"];
    } else if (mode === "drop d") {
      tuningNotes = ["D4", "A3", "D3", "G3", "D2", "E2"];
      tuning = ["D", "A", "D", "G", "B", "E"];
    } else if (mode === "dadgad") {
      tuningNotes = ["D4", "A3", "G3", "D3", "A2", "D2"];
      tuning = ["D", "A", "G", "D", "A", "D"];
    } else if (mode === "open g") {
      tuningNotes = ["D4", "B3", "G3", "D3", "G2", "D2"];
      tuning = ["D", "B", "G", "D", "G", "D"];
    }
  }

  function getNoteFromString(stringNote, fret) {
    const baseNote = stringNote.slice(0, -1);
    const baseOctave = parseInt(stringNote.slice(-1));
    const baseIndex = NOTES.indexOf(baseNote);
    const totalSemitone = baseIndex + fret;
    const note = NOTES[totalSemitone % 12];
    const octave = baseOctave + Math.floor(totalSemitone / 12);
    return `${note}${octave}`;
  }

  function play(note) {
    if (isMuted) return;
    Tone.start();
    const synth = new Tone.Synth().toDestination();
    synth.triggerAttackRelease(note, '8n');
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
          const fret = tab[stringIndex][col];
          if (fret !== "" && !isNaN(fret)) {
            const stringNote = tuningNotes[5 - stringIndex];
            const note = getNoteFromString(stringNote, parseInt(fret));
            play(note);
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
    if (currentTabPos + 1 >= tab[0].length) {
      tab = tab.map(row => [...row, "", ""]);
      await tick();
      if (tabContainer) tabContainer.scrollLeft = tabContainer.scrollWidth;
    }
    const stringNote = tuningNotes[5 - stringIndex];
    const note = getNoteFromString(stringNote, fretIndex);
    play(note);
    if (tab[stringIndex]) {
      tab[stringIndex][currentTabPos] = (fretIndex + 1).toString();
    }
    currentTabPos += 2;
  }

  function deleteCell(stringIndex, colIndex) {
    saveHistory();
    tab[stringIndex][colIndex] = "";
  }

  function selectCell(stringIndex, colIndex) {
    selectedCell = { stringIndex, colIndex };
  }

  let showTuningOptions = false;
  let showTuningOptions2 = false;

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

  function formatFret(n) {
    return n === "" ? "" : n.toString();
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
      tuningNotes,
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
      tuningNotes = data.tuningNotes;
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
  return i >= 3; // 5번줄(인덱스 4), 6번줄(인덱스 5)이면 true
}





</script>

<style>
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
                draggable={fret !== ""}
                class:is-dragging={isDragging}
                on:dragstart={() => handleDragStart(i, j)}
                on:dragover|preventDefault
                on:drop={() => handleDrop(i, j)}
                on:dragend={handleDragEnd}
                on:click={() => selectCell(i, j)}
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
    <button on:click={toggleTuner} class="setting">tuner</button>
    {#if showTuningOptions}
      <div class="tunerset">
        <button class:selected={selectedTuning === "standard"} on:click={() => setTuning("standard")}>standard</button>
        <button class:selected={selectedTuning === "drop d"} on:click={() => setTuning("drop d")}>drop d</button>
        <button class:selected={selectedTuning === "dadgad"} on:click={() => setTuning("dadgad")}>DADGAD</button>
        <button class:selected={selectedTuning === "open g"} on:click={() => setTuning("open g")}>Open G</button>
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
          <div class="fret">
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
