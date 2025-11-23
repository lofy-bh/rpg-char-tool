<template>
  <div class="app-container">
    
    <!-- 1. APP BAR (Oben) -->
    <header class="app-bar">
      <div class="app-bar-left">
        <span class="app-title">RPG Editor</span>
        
        <!-- System Auswahl -->
        <div class="select-wrapper">
          <select v-model="currentSystem" class="system-select">
            <option v-for="sys in availableSystems" :key="sys" :value="sys">
              {{ sys }}
            </option>
          </select>
          <span class="select-arrow">▼</span>
        </div>
      </div>

      <div class="app-bar-right">
        <div v-if="user" class="user-profile">
          <div class="avatar-circle">{{ user.initials }}</div>
          <span class="username">{{ user.name }}</span>
          <button class="btn-text" @click="logout">Logout</button>
        </div>
        <button v-else class="btn-primary" @click="login">
          Login
        </button>
      </div>
    </header>

    <!-- 2. TAB LEISTE -->
    <div class="tabs-container">
      <div 
        v-for="(char, index) in openCharacters" 
        :key="index"
        class="tab-item"
        :class="{ 'tab-active': activeCharIndex === index }"
        @click="activeCharIndex = index"
      >
        <span class="tab-name">{{ char.name || 'Unbenannt' }}</span>
        <span class="tab-close" @click.stop="closeTab(index)">×</span>
      </div>
      <button class="tab-add" @click="addNewTab" title="Neuer Charakter">+</button>
    </div>

    <!-- 3. MAIN CONTENT -->
    <main class="content-area">
      <div class="card" v-if="activeCharacter">
        <div class="card-header">
          <h2>{{ activeCharacter.name || 'Neuer Charakter' }} ({{ currentSystem }})</h2>
          <div class="card-actions">
            <button class="btn-icon" @click="saveCharacter" title="Speichern">💾</button>
            <button class="btn-icon" @click="loadCharacter" title="Laden">📂</button>
          </div>
        </div>

        <div class="card-body split-layout">
          
          <!-- LINKE SPALTE: WERTE -->
          <div class="form-column">
            <div class="input-group">
              <label>Charakter Name</label>
              <input v-model="activeCharacter.name" type="text" placeholder="Name eingeben..." />
            </div>

            <div class="input-group">
              <label>Konzept / Archetyp</label>
              <input v-model="activeCharacter.class" type="text" placeholder="z.B. Solar Exalted" />
            </div>

            <div class="section-title">Attribute</div>
            <div class="row">
              <div class="input-group">
                <label>Körperkraft</label>
                <input v-model.number="activeCharacter.attributes.str" type="number" />
              </div>
              <div class="input-group">
                <label>Geschick</label>
                <input v-model.number="activeCharacter.attributes.dex" type="number" />
              </div>
              <div class="input-group">
                <label>Widerstand</label>
                <input v-model.number="activeCharacter.attributes.sta" type="number" />
              </div>
            </div>
          </div>

          <!-- RECHTE SPALTE: BILD -->
          <div class="image-column">
            <label>Charakter Portrait</label>
            <div 
              class="image-placeholder" 
              @click="triggerImageUpload"
              :style="activeCharacter.image ? { backgroundImage: `url(${activeCharacter.image})` } : {}"
            >
              <span v-if="!activeCharacter.image" class="placeholder-text">
                Klicken um Bild zu laden<br>(wird im JSON gespeichert)
              </span>
            </div>
            <!-- Versteckter File Input -->
            <input 
              type="file" 
              ref="fileInput" 
              accept="image/*" 
              style="display: none" 
              @change="handleImageFile"
            />
            <button v-if="activeCharacter.image" class="btn-text small" @click="activeCharacter.image = ''">Bild entfernen</button>
          </div>

        </div>

        <!-- JSON TEXTAREA -->
        <div class="json-editor-section">
          <label>Raw JSON Data (Editable)</label>
          <textarea 
            class="json-textarea" 
            :value="JSON.stringify(activeCharacter, null, 2)"
            @input="updateJson"
          ></textarea>
        </div>
      </div>
      
      <div v-else class="empty-state">
        <p>Kein Charakter offen. Klicke auf + um zu starten.</p>
      </div>
    </main>

  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import { save, open } from '@tauri-apps/plugin-dialog';
import { writeTextFile, readTextFile } from '@tauri-apps/plugin-fs';

// --- CONFIG ---
const availableSystems = ['Exalted 1st Edition', 'Exalted 3rd Edition', 'Vampire', 'Shadowrun'];
const currentSystem = ref('Exalted 3rd Edition');

// --- USER STATE ---
const user = ref(null);
const login = () => { user.value = { name: 'Storyteller', initials: 'ST' }; };
const logout = () => { user.value = null; };

// --- CHARACTERS / TABS STATE ---
// Wir speichern jetzt eine Liste von offenen Charakteren
const openCharacters = ref([
  {
    name: 'Unbenannt',
    class: '',
    image: '', // Base64 String
    attributes: { str: 1, dex: 1, sta: 1 }
  }
]);
const activeCharIndex = ref(0);

// Computed Property für den aktuell ausgewählten Tab
const activeCharacter = computed(() => {
  return openCharacters.value[activeCharIndex.value];
});

// --- TAB LOGIC ---
const addNewTab = () => {
  openCharacters.value.push({
    name: 'Neu',
    class: '',
    image: '',
    attributes: { str: 1, dex: 1, sta: 1 }
  });
  activeCharIndex.value = openCharacters.value.length - 1;
};

const closeTab = (index) => {
  openCharacters.value.splice(index, 1);
  if (activeCharIndex.value >= openCharacters.value.length) {
    activeCharIndex.value = Math.max(0, openCharacters.value.length - 1);
  }
};

// --- IMAGE HANDLING ---
const fileInput = ref(null);

const triggerImageUpload = () => {
  fileInput.value.click();
};

const handleImageFile = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  // Wir nutzen FileReader um das Bild in einen Base64 String zu wandeln
  // Damit kann es direkt im JSON gespeichert werden (Single Source of Truth)
  const reader = new FileReader();
  reader.onload = (e) => {
    activeCharacter.value.image = e.target.result;
  };
  reader.readAsDataURL(file);
};

// --- JSON HANDLING ---
const updateJson = (e) => {
  try {
    const parsed = JSON.parse(e.target.value);
    // Wir updaten den aktiven Charakter mit dem Text aus der Area
    // Object.assign behält die Reaktivität bei, ist aber sicherer als Komplettaustausch
    Object.assign(activeCharacter.value, parsed);
  } catch (err) {
    // Ungültiges JSON ignorieren wir vorerst (oder könnten UI Feedback geben)
  }
};

// --- TAURI FILE IO ---
const saveCharacter = async () => {
  if (!activeCharacter.value) return;
  try {
    const filePath = await save({
      filters: [{ name: 'RPG Character JSON', extensions: ['json'] }]
    });
    if (!filePath) return;
    await writeTextFile(filePath, JSON.stringify(activeCharacter.value, null, 2));
    alert('Charakter gespeichert!');
  } catch (err) {
    console.error(err);
    alert('Fehler: ' + err);
  }
};

const loadCharacter = async () => {
  try {
    const filePath = await open({
      multiple: false,
      filters: [{ name: 'RPG Character JSON', extensions: ['json'] }]
    });
    if (!filePath) return;
    
    const content = await readTextFile(filePath);
    const loadedChar = JSON.parse(content);
    
    // Als neuen Tab hinzufügen
    openCharacters.value.push(loadedChar);
    activeCharIndex.value = openCharacters.value.length - 1;
    
  } catch (err) {
    console.error(err);
    alert('Fehler beim Laden: ' + err);
  }
};
</script>

<style>
/* CSS Variablen (Unverändert) */
:root {
  --bg-color: #121212;
  --surface-color: #1e1e1e;
  --surface-hover: #2d2d2d;
  --primary-color: #bb86fc;
  --secondary-color: #03dac6;
  --text-high: #ffffff;
  --text-medium: #b0b0b0;
  --border-color: #333333;
  --input-bg: #2c2c2c;
  --app-bar-height: 64px;
}

/* Base Styles (Unverändert) */
body { margin: 0; font-family: 'Roboto', 'Segoe UI', sans-serif; background-color: var(--bg-color); color: var(--text-high); }
.app-container { display: flex; flex-direction: column; height: 100vh; }
.app-bar { height: var(--app-bar-height); background-color: var(--surface-color); display: flex; justify-content: space-between; align-items: center; padding: 0 20px; box-shadow: 0 2px 4px rgba(0,0,0,0.5); z-index: 10; border-bottom: 1px solid var(--border-color); }
.app-bar-left, .app-bar-right { display: flex; align-items: center; gap: 20px; }
.app-title { font-weight: bold; font-size: 1.1rem; color: var(--text-medium); }
.select-wrapper { position: relative; }
.system-select { appearance: none; background-color: var(--input-bg); color: var(--text-high); border: 1px solid var(--border-color); padding: 8px 30px 8px 12px; border-radius: 4px; cursor: pointer; }
.select-arrow { position: absolute; right: 10px; top: 50%; transform: translateY(-50%); font-size: 0.7rem; pointer-events: none; color: var(--text-medium); }
.btn-primary { background-color: var(--primary-color); color: #000; border: none; padding: 8px 16px; border-radius: 4px; font-weight: bold; cursor: pointer; }
.btn-primary:hover { opacity: 0.9; }
.btn-text { background: none; border: none; color: var(--text-medium); cursor: pointer; text-decoration: underline; }
.btn-text.small { font-size: 0.8rem; margin-top: 5px; display: block; text-align: center;}
.user-profile { display: flex; align-items: center; gap: 10px; }
.avatar-circle { width: 32px; height: 32px; background-color: var(--secondary-color); color: #000; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-weight: bold; }
.content-area { flex: 1; padding: 20px; overflow-y: auto; display: flex; flex-direction: column; align-items: center; }
.card { background-color: var(--surface-color); border-radius: 0 8px 8px 8px; width: 100%; max-width: 900px; padding: 20px; box-shadow: 0 4px 6px rgba(0,0,0,0.3); border: 1px solid var(--border-color); }
.card-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid var(--border-color); padding-bottom: 15px; margin-bottom: 20px; }
.card-header h2 { margin: 0; font-size: 1.5rem; color: var(--primary-color); }
.btn-icon { background: none; border: 1px solid var(--border-color); color: var(--text-high); font-size: 1.2rem; padding: 5px 10px; border-radius: 4px; cursor: pointer; margin-left: 5px; }
.input-group { margin-bottom: 15px; display: flex; flex-direction: column; }
.row { display: flex; gap: 15px; }
.row .input-group { flex: 1; }
label { font-size: 0.85rem; color: var(--text-medium); margin-bottom: 5px; }
input { background-color: var(--input-bg); border: 1px solid transparent; border-bottom: 2px solid var(--text-medium); color: var(--text-high); padding: 10px; border-radius: 4px 4px 0 0; font-size: 1rem; outline: none; }
input:focus { border-bottom-color: var(--primary-color); background-color: #333; }
.section-title { color: var(--secondary-color); font-weight: bold; margin: 15px 0 10px 0; border-bottom: 1px solid var(--border-color); display: inline-block; padding-bottom: 2px; }

/* --- NEUE STYLES --- */

/* Tabs Styling */
.tabs-container {
  display: flex;
  background-color: var(--bg-color);
  padding: 10px 20px 0 20px;
  max-width: 900px;
  width: 100%;
  margin: 0 auto; /* Zentriert die Tabs über der Karte */
  overflow-x: auto;
}

.tab-item {
  padding: 10px 20px;
  background-color: var(--surface-color);
  color: var(--text-medium);
  border-radius: 8px 8px 0 0;
  margin-right: 5px;
  cursor: pointer;
  border: 1px solid var(--border-color);
  border-bottom: none;
  display: flex;
  align-items: center;
  gap: 10px;
  transition: background-color 0.2s;
  min-width: 100px;
  justify-content: space-between;
}

.tab-item:hover {
  background-color: var(--surface-hover);
}

.tab-active {
  background-color: var(--surface-color);
  color: var(--primary-color);
  border-top: 2px solid var(--primary-color);
  position: relative;
  /* Überdeckt den Border der Karte */
  margin-bottom: -1px; 
  z-index: 5; 
  border-bottom: 1px solid var(--surface-color); 
}

.tab-close {
  font-size: 1.2rem;
  opacity: 0.5;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.tab-close:hover { opacity: 1; background-color: rgba(255,255,255,0.1); }

.tab-add {
  background: none;
  border: none;
  color: var(--text-medium);
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0 15px;
}
.tab-add:hover { color: var(--primary-color); }

/* Split Layout für Bild & Stats */
.split-layout {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 20px;
}

/* Image Upload Styling */
.image-column {
  display: flex;
  flex-direction: column;
}

.image-placeholder {
  width: 100%;
  aspect-ratio: 3/4; /* Hochformat für Portraits */
  background-color: var(--input-bg);
  border: 2px dashed var(--border-color);
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  color: var(--text-medium);
  font-size: 0.8rem;
  background-size: cover;
  background-position: center;
  transition: border-color 0.2s;
}

.image-placeholder:hover {
  border-color: var(--primary-color);
}

.placeholder-text {
  padding: 10px;
  pointer-events: none; /* Damit Klick durchgeht */
}

/* JSON Textarea */
.json-editor-section {
  margin-top: 30px;
  border-top: 1px solid var(--border-color);
  padding-top: 15px;
}

.json-textarea {
  width: 100%;
  height: 150px;
  background-color: #000;
  color: #0f0; /* Matrix Grün ;) */
  font-family: 'Courier New', monospace;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  padding: 10px;
  resize: vertical;
  font-size: 0.8rem;
  box-sizing: border-box;
}

.empty-state {
  text-align: center;
  color: var(--text-medium);
  margin-top: 50px;
}
</style>