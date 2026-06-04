<template>
  <div class="calendar-app">
    <h1>📅 2026 行事曆管理系統</h1>

    <div class="toolbar">
      <div class="file-ops">
        <input type="file" ref="fileInput" @change="handleImport" accept=".ics" style="display: none" />
        <button @click="$refs.fileInput.click()" class="btn-import">匯入 ICS 檔案</button>
        <button @click="exportICS" class="btn-export">匯出為 ICS</button>
      </div>
      <button @click="syncToGoogleSheets" :disabled="isSyncing" class="btn-sync">
        {{ isSyncing ? '同步中...' : '同步至 Google Sheets' }}
      </button>
    </div>

    <section class="add-event">
      <h3>新增事項</h3>
      <div class="form-grid">
        <input v-model="form.summary" placeholder="事件名稱 (Summary)" />
        <input v-model="form.start" type="datetime-local" />
        <input v-model="form.end" type="datetime-local" />
        <select v-model="form.category">
          <option value="">選擇現有分類</option>
          <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
        </select>
        <input v-model="newCategory" placeholder="或新增分類..." @keyup.enter="addNewCategory" />
        <textarea v-model="form.description" placeholder="事件描述..."></textarea>
        <button @click="addEvent">新增到清單</button>
      </div>
    </section>

    <section class="event-list">
      <h3>事件清單 ({{ events.length }})</h3>
      <table>
        <thead>
          <tr>
            <th>摘要</th>
            <th>分類</th>
            <th>開始時間</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(ev, idx) in events" :key="idx">
            <td>{{ ev.summary }}</td>
            <td><span class="badge">{{ ev.category || '未分類' }}</span></td>
            <td>{{ new Date(ev.start).toLocaleString() }}</td>
            <td><button @click="events.splice(idx, 1)" class="btn-delete">刪除</button></td>
          </tr>
        </tbody>
      </table>
    </section>
  </div>
</template>

<script setup>
import { ref } from 'vue';

// 請將此處替換為您 GAS 部署後的 Web App URL
const GAS_URL = 'YOUR_GOOGLE_APPS_SCRIPT_URL';

const events = ref([]);
const categories = ref(['工作', '私人', '假日']);
const newCategory = ref('');
const isSyncing = ref(false);

const form = ref({ summary: '', start: '', end: '', description: '', category: '' });

const addNewCategory = () => {
  if (newCategory.value && !categories.value.includes(newCategory.value)) {
    categories.value.push(newCategory.value);
    form.value.category = newCategory.value;
    newCategory.value = '';
  }
};

const addEvent = () => {
  if (!form.value.summary || !form.value.start) return alert('請填寫標題與時間');
  events.value.push({ ...form.value });
  form.value = { summary: '', start: '', end: '', description: '', category: '' };
};

const handleImport = (e) => {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (f) => parseICS(f.target.result);
  reader.readAsText(file);
};

const parseICS = (data) => {
  const vevents = data.split('BEGIN:VEVENT');
  vevents.shift();
  const imported = vevents.map(block => {
    const get = (key) => (block.match(new RegExp(`${key}:(.*)`)) || [])[1]?.trim() || '';
    // 轉換 ICS 格式日期 (YYYYMMDDTHHMMSS) 為 HTML datetime-local 相容格式
    const convertDate = (d) => d ? `${d.substr(0,4)}-${d.substr(4,2)}-${d.substr(6,2)}T${d.substr(9,2)}:${d.substr(11,2)}` : '';
    return {
      summary: get('SUMMARY'),
      start: convertDate(get('DTSTART')),
      end: convertDate(get('DTEND')),
      description: get('DESCRIPTION'),
      category: get('CATEGORIES')
    };
  });
  events.value = [...events.value, ...imported];
};

const exportICS = () => {
  let ics = "BEGIN:VCALENDAR\nVERSION:2.0\n";
  events.value.forEach(ev => {
    const toIcsDate = (d) => d.replace(/[-:]/g, '') + '00Z';
    ics += `BEGIN:VEVENT\nSUMMARY:${ev.summary}\nDTSTART:${toIcsDate(ev.start)}\nDTEND:${toIcsDate(ev.end || ev.start)}\nDESCRIPTION:${ev.description}\nCATEGORIES:${ev.category}\nEND:VEVENT\n`;
  });
  ics += "END:VCALENDAR";
  const blob = new Blob([ics], { type: 'text/calendar' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `calendar-export-${new Date().toISOString().split('T')[0]}.ics`;
  a.click();
};

const syncToGoogleSheets = async () => {
  if (events.value.length === 0) return;
  isSyncing.value = true;
  try {
    // 注意：Google Apps Script 的 doPost 在 no-cors 下無法讀取 Response，但資料會成功傳達
    await fetch(GAS_URL, {
      method: 'POST',
      mode: 'no-cors',
      cache: 'no-cache',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(events.value)
    });
    alert('同步請求已發送！請檢查 Google 試算表。');
  } catch (err) {
    alert('同步發生錯誤：' + err.message);
  } finally {
    isSyncing.value = false;
  }
};
</script>

<style scoped>
.calendar-app { font-family: system-ui; max-width: 900px; margin: auto; padding: 20px; }
.toolbar { display: flex; justify-content: space-between; margin-bottom: 20px; }
.form-grid { display: grid; gap: 10px; padding: 15px; background: #f9f9f9; border-radius: 8px; }
input, select, textarea { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
table { width: 100%; border-collapse: collapse; margin-top: 20px; }
th, td { text-align: left; padding: 12px; border-bottom: 1px solid #eee; }
button { cursor: pointer; padding: 8px 16px; border-radius: 4px; border: none; background: #42b883; color: white; }
.btn-sync { background: #35495e; }
.btn-delete { background: #ff5252; font-size: 12px; }
.badge { background: #e0f2f1; color: #00796b; padding: 2px 8px; border-radius: 10px; font-size: 12px; }
</style>