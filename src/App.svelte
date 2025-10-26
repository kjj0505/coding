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

  let sustainExt = []; // { str, start, end }를 저장: 같은 줄(str)에서 start열의 음을 end열까지 유지

  let isComposing = false;

  function toggleMute() {
  isMuted = !isMuted;
  if (isMuted) {
    // 지금 울리는 소리 즉시 끄기
    try { monoSynth?.triggerRelease(); } catch {}
    try { polySynth?.releaseAll?.(); } catch {}
  }
}

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


  onMount(async () => {
  // 👉 우클릭 메뉴 방지 (마우스 오른쪽 클릭 시 context 메뉴 안 뜨게)
  window.addEventListener('contextmenu', e => e.preventDefault());

  // 👉 마우스 업 이벤트 (왼쪽 드래그 해제, 우클릭 드래그 지속음 처리)
  window.addEventListener('mouseup', (e) => {
  if (e.button === 2 && isRightDragging && rightDragStart && rightDragEnd) {
    if (!isComposing) {
    rightDragStart = null;
    rightDragEnd = null;
    isRightDragging = false;
    isDragging = false;
    isSwapping = false;
    return; // 기록 없이 종료
  }
    const startRow = rightDragStart.stringIndex;
    const startCol = Math.min(rightDragStart.colIndex, rightDragEnd.colIndex);
    const endCol = Math.max(rightDragStart.colIndex, rightDragEnd.colIndex);

    // 시작칸이 "~"면 왼쪽의 숫자를 실제 시작점으로 보정
   let headCol = startCol;
   if (tab[startRow][headCol] === "~") {
     let k = headCol - 1;
     while (k >= 0 && tab[startRow][k] === "~") k--;
     if (k >= 0 && tab[startRow][k] !== "" && !isNaN(tab[startRow][k])) {
       headCol = k;
     }
   }
   const headVal = tab[startRow][headCol];
   if (headVal !== "" && !isNaN(headVal)) {
     saveHistory();
     // 1) 지속 구간 기록/업데이트
     const idx = sustainExt.findIndex(s => s.str === startRow && s.start === headCol);
     if (idx >= 0) {
       sustainExt[idx].end = Math.max(sustainExt[idx].end, endCol);
     } else {
       sustainExt.push({ str: startRow, start: headCol, end: endCol });
     }
     // 2) 빈칸만 "~"로 표시 (사이에 다른 숫자가 있어도 덮지 않음)
     for (let j = headCol + 1; j <= endCol; j++) {
       if (tab[startRow][j] === "") tab[startRow][j] = "~";
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
  await buildInstruments(selectedGuitar);     // ✅ 세팅 완료 보장

  // ✅ 브라우저 오디오 정책 해제: 페이지 내 첫 클릭/터치 시 AudioContext 열기
  const unlock = async () => { await ensureAudioReady(); };
  window.addEventListener('pointerdown', unlock, { once: true });
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
    buildInstruments(type);
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

  async function play(note, duration, velocity = 0.85, atTime) {
  if (isMuted || !note) return;
  if (buildingAudio) {                 // 전환이 끝날 때까지 한 틱 양보
   await new Promise(r => setTimeout(r, 0));
   if (buildingAudio) return;         // 혹시 길어지면 그때만 보류
 }         // 전환 중엔 발음 보류(원하면 대기로 바꿔도 OK)
  await ensureAudioReady();
  if (synthsDead()) await buildInstruments(selectedGuitar);
  if (!monoSynth) return;
  const d = (duration ?? (getCellMs() / 1000));
  // 한 음만 나게: 이전 음 즉시 릴리즈
  try { monoSynth.triggerRelease(); } catch {}
  monoSynth.triggerAttackRelease(note, d, atTime, velocity);
}

  function resetTab() {
    saveHistory();
    tab = Array(6).fill().map(() => Array(59).fill(""));
    currentTabPos = 0;

    timeSignature = "4/4";
    bpm = 120;
    sustainExt = []
    
    // 선택/드래그 상태도 정리(선택)
    selectedCell = null;
    selectedRange = null;
    isDragging = false;
    isSwapping = false;

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
  if (get(isPlaying) || buildingAudio) return;
  const lastCol = getLastActiveColumn();
  if (lastCol === -1) return;

  await ensureAudioReady();
  if (synthsDead()) await buildInstruments(selectedGuitar);

  isPlaying.set(true);
  const delayMs = getCellMs();

  try {
    for (let col = 0; col <= lastCol; col++) {
      if (!get(isPlaying)) break;

      // ✅ 이 부분만 교체
      const notes = [];
      let sustainCols = 1;

      for (let stringIndex = 0; stringIndex < 6; stringIndex++) {
        const cell = tab[stringIndex][col];
        if (cell !== "" && cell !== "~" && !isNaN(cell)) {
          // 이 줄의 sustain 길이 측정
          let s = 1, k = col + 1;
          while (k <= lastCol && tab[stringIndex][k] === "~") { s++; k++; }
          sustainCols = Math.max(sustainCols, s);

          const base = get(tuningNotes)[stringIndex];
          const note = getNoteFromString(base, parseInt(cell, 10));
          if (note) notes.push(note);
        }
      }

      if (notes.length === 1) {
        // 단음은 기존 mono 경로 유지
        play(notes[0], (sustainCols * delayMs) / 1000);
      } else if (notes.length > 1) {
        // 코드면 한 번에 ‘동시에’
        const durSec = (sustainCols * delayMs) / 1000;
        try { polySynth?.triggerAttackRelease(notes, durSec, Tone.now(), 0.85); } catch {}
      }

      await new Promise(r => setTimeout(r, delayMs));
    }
  } finally {
    isPlaying.set(false);
  }
}



  function pauseTab() {
    isPlaying.set(false);
  }


  // 악보에서 마지막으로 어떤 줄이든 음이 들어간 열(컬럼) 찾기
function getLastUsedCol() {
  let last = -1;
  for (let i = 0; i < 6; i++) {
    for (let j = tab[i].length - 1; j >= 0; j--) {
      if (tab[i][j] !== "") { last = Math.max(last, j); break; }
    }
  }
  return last;
}

// 필요한 열까지 테이블 길이 늘리기
async function ensureRoom(col) {
  if (col + 1 >= tab[0].length) {
    tab = tab.map(row => [...row, "", ""]);
    await tick();
    if (tabContainer) tabContainer.scrollLeft = tabContainer.scrollWidth;
  }
}




  async function handleFretClick(stringIndex, fretIndex) {
  if (stringIndex === 6) return;

  const stringNote = get(tuningNotes)[stringIndex];
  const note = getNoteFromString(stringNote, fretIndex);
  play(note); // 🔊 소리는 항상 낸다

  // 🎼 compose OFF면 기록하지 않고 종료
  if (!isComposing) return;

  // ⬇️ compose ON일 때만 기록 로직 수행
  saveHistory();

  const last = getLastUsedCol();
  const col  = (last === -1) ? 0 : last + 2;
  await ensureRoom(col);

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
      timeSignature,
      bpm
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
      if (typeof data.timeSignature === 'string') timeSignature = data.timeSignature;
      if (typeof data.bpm === 'number') bpm = data.bpm;
      currentTabPos = getLastActiveColumn() + 2;
    }
    showSavedTabs = false;
    selectedTabs = [];
    buildInstruments(selectedGuitar);
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

async function applyChord(chordName) {
  const chord = chords[chordName];
  if (!chord) return;

  // 🎼 compose OFF면 기록 없이 소리만
  if (!isComposing) {
    playChord(chordName);
    return;
  }

  saveHistory();

  const last = getLastUsedCol();
  const col  = (last === -1) ? 0 : last  + 2;
  await ensureRoom(col);

  // 위(1번줄)→아래(6번줄) 보이도록 뒤집어 기록
  for (let vis = 0; vis < 6; vis++) {
    const dataIndex = 5 - vis;
    const fret = chord.frets[5 - vis];
    tab[dataIndex][col] = fret === "x" ? "" : fret.toString();
  }

  // 소리는 그대로
  playChord(chordName);
}



// 코드 UI toggle 함수

// 코드 클릭 시 호출: 악보에 입력 + 소리 재생

// 코드 구성음을 동시에 재생
function playChord(chordName, duration) {
  const chord = chords[chordName];
   if (!chord || !polySynth || isMuted) return;
 const now = Tone.now();


  const d = (duration ?? (getBeatMs() / 1000));
  for (let visIndex = 0; visIndex < 6; visIndex++) {
   const fret = chord.frets[5 - visIndex]; // ← frets를 거꾸로 읽기
   if (fret === "x") continue;
   const dataIndex   = 5 - visIndex;
   const stringNote  = get(tuningNotes)[dataIndex];
   const note        = getNoteFromString(stringNote, fret);
   polySynth.triggerAttackRelease(note, d, now);
 }
}










function downloadFile(filename, content, mime = "application/octet-stream") {
    const blob = new Blob([content], { type: mime });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    setTimeout(() => {
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    }, 0);
  }

  // JSON 스키마로 저장 (복원용)
  function serializeTabToJson() {
    return JSON.stringify(
      {
        version: 1,
        tab,
        tuningNotes: get(tuningNotes),
        tuning,
        selectedTuning,
        selectedGuitar,
        timeSignature,
        bpm
      },
      null,
      2
    );
  }

  // JSON 로드 시 검증 + 반영
  function deserializeTabFromJson(jsonText) {
    const data = JSON.parse(jsonText);
    if (!data || !Array.isArray(data.tab) || data.tab.length !== 6) {
      throw new Error("올바르지 않은 탭 형식입니다.");
    }
    // 간단 검증: 각 행은 배열이어야 함
    for (const row of data.tab) {
      if (!Array.isArray(row)) throw new Error("탭 데이터가 손상되었습니다.");
    }

    saveHistory();
    tab = data.tab;
    if (Array.isArray(data.tuningNotes)) tuningNotes.set(data.tuningNotes);
    if (Array.isArray(data.tuning)) tuning = data.tuning;
    if (typeof data.selectedTuning === "string") selectedTuning = data.selectedTuning;
    if (typeof data.selectedGuitar === "string") selectedGuitar = data.selectedGuitar;

    // 커서/스크롤 정리
    currentTabPos = getLastActiveColumn() + 2;
    tick().then(() => {
      if (tabContainer) tabContainer.scrollLeft = tabContainer.scrollWidth;
    });
  }
  
  function makeSafeName(name) {
    return (name || "untitled")
      .replace(/[\\/:*?"<>|]+/g, "_")
      .slice(0, 50);
  }

  function exportJSON() {
  if (!hasAnyNotes()) {
    alert("악보가 비어 있어요. 먼저 음을 입력하세요!");
    return;
  }
  const base = makeSafeName(saveName) || "tab";
  const stamp = new Date().toISOString().replace(/[:.]/g, "-");
  const filename = `${base}_${stamp}.json`;
  const content = serializeTabToJson();
  downloadFile(filename, content, "application/json");
}

  let fileInput;

  async function importJSON(e) {
    const file = e?.target?.files?.[0];
    if (!file) return;
    try {
      const text = await file.text();
      deserializeTabFromJson(text);
      alert(`'${file.name}' 불러오기 완료!`);
    } catch (err) {
      console.error(err);
      alert("불러오기에 실패했습니다. 올바른 .json 파일인지 확인하세요.");
    } finally {
      // 같은 파일을 다시 선택해도 change가 뜨도록 입력값 초기화
      e.target.value = "";
    }

  }
let timeSignature = "4/4";
let bpm = 120;                 // 필요하면 슬라이더로 묶어도 됨
const COLUMNS_PER_BEAT = 2;    // 한 박을 그리드 2칸으로 표현(네 코드와 동일)

// 분자/분모 세터 (UI에서 호출)
function setTimeTop(n) {
  const bot = parseInt(timeSignature.split('/')[1]) || 4;
  const top = Math.max(1, Math.min(16, parseInt(n) || 4));
  timeSignature = `${top}/${bot}`;
}
function setTimeBottom(den) {
  const top = parseInt(timeSignature.split('/')[0]) || 4;
  const allowed = [2,4,8,16];
  const bot = allowed.includes(+den) ? +den : 4;
  timeSignature = `${top}/${bot}`;
}

// 시간 계산(분모가 2/4/8/16일 때 정확)
function getBeatMs() {
  const [, bottomStr] = (timeSignature || "4/4").split("/");
  const bottom = parseInt(bottomStr) || 4;     // 2,4,8,16
  const quarterMs = 60000 / bpm;               // 4분음표 길이
  return quarterMs * (4 / bottom);             // 1박 길이
}
function getCellMs() {
  return getBeatMs() / COLUMNS_PER_BEAT;       // 한 칸 길이
}

let timeTopInput = timeSignature.split('/')[0];
$: timeTopInput = timeSignature.split('/')[0]; // timeSignature 바뀌면 동기화






let monoSynth = null;   // 단음(프렛/키보드)
let polySynth = null;   // 코드(동시 발음)
let fxNodes = [];       // 이펙트 노드들 (dispose용)

function disposeInstruments() {
  try { monoSynth?.dispose(); } catch {}
  try { polySynth?.dispose(); } catch {}
  monoSynth = null;
  polySynth = null;
  fxNodes.forEach(n => { try { n.dispose?.(); } catch {} });
  fxNodes = [];
}

let buildingAudio = false;  // ✅ 먼저 선언

// 선택된 타입으로 신스/이펙트 구성
async function buildInstruments(type = selectedGuitar) {
  buildingAudio = true;
  await ensureAudioReady();
  disposeInstruments();
try {
  if (type === 'electric') {
    // 🎸 Electric: AMSynth + Distortion + Chorus + Reverb
    const dist   = new Tone.Distortion(0.32);
   const chorus = new Tone.Chorus({ frequency: 4, delayTime: 2.5, depth: 0.28 });
   const reverb = new Tone.Reverb({ decay: 1.8, wet: 0.22 });
   await reverb.ready;   // ✅ 이펙트 준비 대기
   chorus.start();
    const vol    = new Tone.Volume(-6).toDestination();

    dist.connect(chorus);
    chorus.connect(reverb);
    reverb.connect(vol);
    fxNodes = [dist, chorus, reverb, vol];

    monoSynth = new Tone.AMSynth({
      oscillator: { type: 'sawtooth' },
      envelope: { attack: 0.004, decay: 0.12, sustain: 0.4, release: 0.25 }
    }).connect(dist);

    polySynth = new Tone.PolySynth(Tone.AMSynth, {
      oscillator: { type: 'sawtooth' },
      envelope: { attack: 0.004, decay: 0.12, sustain: 0.4, release: 0.25 }
    }).connect(dist);

  } else {
    // 🎻 Acoustic 느낌: 깨끗한 Synth + Reverb
    fxNodes = [];
  monoSynth = new Tone.Synth().toDestination();
  polySynth = new Tone.PolySynth(Tone.Synth).toDestination();
  }
  buildingAudio = false; 
}
finally {
    buildingAudio = false;  // ✅ 무조건 false로 내려감
  }
}

async function ensureAudioReady() {
  try { await Tone.start(); } catch {}
}

function synthsDead() {
  return !monoSynth || !polySynth || monoSynth.disposed || polySynth.disposed;
}

let chordCol = null;                // 코드 입력 중 고정할 컬럼
const activeNotes = new Map();      // key(숫자) -> note 문자열
let lastFretIndex = 0;



// 🔁 숫자키(1~6) 업: 폴리 모드면 해당 음 Release, 상태 정리
window.addEventListener("keydown", async (e) => {
  const n = parseInt(e.key, 10);
  if (!(n >= 1 && n <= 6)) return;

  // 키 반복 방지
  if (e.repeat) return;
  if (pressedKeys.has(n)) return;
  pressedKeys.add(n);

  const fret = (hoveredFretIndex ?? lastFretIndex ?? 0);
  const stringIndex = 6 - n;
  const stringNote  = get(tuningNotes)[stringIndex];
  const note        = getNoteFromString(stringNote, fret);
  if (!note) return;

  // Shift 누른 상태: 폴리(동시발음) — keyup에서 release
  if (isShiftDown) {
    await ensureAudioReady();
    if (synthsDead()) await buildInstruments(selectedGuitar);
    if (!isMuted && polySynth) {
      try { polySynth.triggerAttack(note); } catch {}
    }
    activeNotes.set(n, note);

    // compose ON이면 같은 column에 기록
    if (isComposing) {
      if (chordCol == null) {
        chordCol = findNextAvailableCol(0, COLUMNS_PER_BEAT);
        await ensureRoom(chordCol);
        saveHistory();
      }
      tab[stringIndex][chordCol] = (fret + 1).toString();
    }
    return;
  }

  // Shift가 아니면: 마우스 클릭과 동일하게 mono로 한 번만 발음
  if (!isMuted) play(note);  // ← 마우스와 동일 경로

  if (isComposing) {
    saveHistory();
    const col = findNextAvailableCol(0, COLUMNS_PER_BEAT);
    await ensureRoom(col);
    tab[stringIndex][col] = (fret + 1).toString();
  }
});

// 🔁 키 업: 폴리 모드에서 개별/전체 릴리즈 처리
window.addEventListener("keyup", (e) => {
  if (e.key === "Shift") {
    try {
      for (const note of activeNotes.values()) {
        try { polySynth?.triggerRelease(note); } catch {}
      }
    } finally {
      activeNotes.clear();
      chordCol = null;
    }
    isShiftDown = false;
    return;
  }

  const n = parseInt(e.key, 10);
  if (pressedKeys.has(n)) pressedKeys.delete(n);

  if (activeNotes.has(n)) {
    const note = activeNotes.get(n);
    try { polySynth?.triggerRelease(note); } catch {}
    activeNotes.delete(n);
    if (activeNotes.size === 0) chordCol = null;
  }
});





</script>




<style>
  .title{
    font-size: 18px;
    font-weight: 800;
    color: #222;
    background: transparent;
    border: none;
    padding: 0 8px 0 0;
    cursor: default;
    line-height: 1;
  }

  .beat{
    position: relative;
    padding-left: 10px;
  }

  .beat::before{
    content: "박자표 (time signature)";
    position: absolute;
    top: -50%;
    font-size: 12px; font-weight: 700; color: #334;
    margin-right: 6px;
  }

  .bpm{
    position: relative;
    padding-left: 10px;
  }

  .bpm::before{
    content: "템포 (BPM)";
    position: absolute;
    top: -50%;
    font-size: 12px; font-weight: 700; color: #334;
  }

  .setting{
    background-color: #979797;
    border: 3px solid #484848;
    color: black;
    font-weight: bold;
    margin-top: 10px;
    margin-bottom: 10px;
  }
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
  .top-bar { display:flex; justify-content:space-between; align-items:center;
  gap:12px; padding:10px 12px; background:#fff; border-bottom:1px solid #e6e9f2;}
  .tab-display { background: #ccc; padding: 1rem; font-family: 'Courier New', monospace; font-size: 14px; overflow-x: auto; overflow-y: hidden; white-space: pre; box-sizing: border-box; max-width: 100%; }
  .controls { display: flex; gap: 0.5rem; padding: 0.5rem 1rem; align-items: center; }
  .fretboard-wrapper { flex: 1; display: flex; overflow: auto; align-items: stretch; }
  .string-labels { display: flex; flex-direction: column; justify-content: center; gap: 24px; }
  .string-label { background: red; color: white; padding: 0.1rem 0.4rem; border-radius: 4px; text-align: center; font-weight: bold; margin-right: 5px; }
  .string-label:hover{cursor: pointer;}
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

.top-bar button.selected { background:#666; color:#fff; font-weight:bold; }





/* 숫자 input/분모 버튼/슬라이더 정리 */
.top-bar input[type="number"]{
  width: 44px; height: 28px;
  text-align: center;
  border: 1px solid #bfc9e8;
  border-radius: 8px;
  background: #fff;
}
.beat > span{ opacity: .8; font-weight: 700; }
.beat button{
  background:#f5f7fb; border:1px solid #d7def1; color:#223;
  border-radius:10px; padding:6px 10px;
}
.beat button.selected{
  background:#364fc7; border-color:#364fc7; color:#fff;
}
.bpm input[type="range"]{ height: 4px; }



button {
  appearance: none;
  -webkit-tap-highlight-color: transparent;
  border: 1px solid #dfe4ee;
  background: linear-gradient(180deg, #ffffff, #f7f9fc);
  color: #243046;
  font-weight: 600;
  font-size: 14px;
  border-radius: 12px;
  padding: 8px 12px;
  line-height: 1;
  display: inline-flex;
  align-items: center;
  gap: 8px;            /* 아이콘과 텍스트 간격 */
  cursor: pointer;
  box-shadow:
    0 1px 0 rgba(255,255,255,.9) inset,
    0 1px 2px rgba(15, 23, 42, 0.04);
  transition:
  background .15s ease,
  border-color .15s ease,
  box-shadow .15s ease,
    transform .04s ease;
}

</style>

<div class="container"> 
  <div class="top-bar">
  <div class=title>guitar site</div>

  <!-- 버튼들은 로그인 여부와 무관하게 사용 가능 -->
  <div style="display:flex; gap:8px; align-items:center; flex-wrap:wrap;">

    <div style="display:flex; gap:6px; align-items:center; flex-wrap:wrap;">
      
      <!-- 분자 직접 입력(원하면) -->
      <input
        type="number" min="1" max="16"
        style="width:36px; text-align:center;"
        bind:value={timeTopInput}
        on:change={() => setTimeTop(timeTopInput)}
      />
      
      <!-- 분모 고정 버튼: 2/4/8/16 -->
      <div class="beat" style="display:flex; gap:4px; align-items:center; margin-left:6px;">
        <span>/</span>
        {#each [2,4,8,16] as den}
          <button on:click={() => setTimeBottom(den)}
                  class:selected={+timeSignature.split('/')[1]===den}>{den}</button>
        {/each}
      </div>
    
      <!-- (선택) BPM 슬라이더 -->
      <div class="bpm" style="display:flex; gap:6px; align-items:center; margin-left:10px;">
        <span>BPM</span>
        <input
      type="range"
      min="40"
      max="240"
      step="1"
      bind:value={bpm}
    />
    <span style="width:28px; text-align:right;">{bpm}</span>
  </div>
</div>

    <button on:click={exportJSON} disabled={!hasAnyNotes()}>export file</button>


    <button on:click={() => fileInput?.click()}>import file</button>
<input type="file" accept=".json" bind:this={fileInput} style="display:none" on:change={importJSON} />
    <button on:click={() => isComposing = !isComposing} class:selected={isComposing}>
  {isComposing ? 'compose: ON' : 'compose: OFF'}
</button>
    <button on:click={resetTab}>reset</button>
    <input placeholder="악보이름" bind:value={saveName} />
    <button on:click={saveTab}>save</button>
    <button on:click={toggleSavedTabs}>saved</button>
    <button on:click={pauseTab} disabled={!$isPlaying}>pause</button>
    <button on:click={playTab} disabled={$isPlaying || !hasAnyNotes() || buildingAudio}>play</button>
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
          title="브라우저 로컬 저장"
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
  on:mouseenter={() => {
   if (isRightDragging) {
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
        lastFretIndex = fretIndex;   // ⬅️ 추가
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
      <button style="border-radius: 50%;" on:click={toggleMute}>
  {#if isMuted} 🔇 {:else} 🔊 {/if}
</button>
    </div>
  </div>
</div>
