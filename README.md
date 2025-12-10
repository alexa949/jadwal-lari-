<!doctype html>
<html lang="id">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Jadwal Lari Harian — Atur Hari & Jam</title>
  <style>
    :root{--bg:#0f172a;--card:#0b1220;--accent:#06b6d4;--muted:#94a3b8;color-scheme:dark}
    *{box-sizing:border-box}
    body{font-family:system-ui,Segoe UI,Roboto,Arial;min-height:100vh;margin:0;padding:24px;background:linear-gradient(180deg,#071029, #071428);color:#e6eef8}
    .wrap{max-width:900px;margin:0 auto}
    header{display:flex;align-items:center;gap:16px;margin-bottom:18px}
    h1{margin:0;font-size:20px}
    .card{background:rgba(255,255,255,0.03);padding:16px;border-radius:12px;box-shadow:0 6px 18px rgba(2,6,23,0.6)}
    form{display:grid;grid-template-columns:1fr 150px;gap:12px;align-items:end}
    label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
    .field{display:flex;gap:8px;align-items:center}
    input[type="text"], input[type="time"], select{width:100%;padding:8px 10px;border-radius:8px;border:1px solid rgba(255,255,255,0.06);background:transparent;color:inherit}
    .days{display:flex;gap:6px;flex-wrap:wrap}
    .day{padding:6px 8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:rgba(255,255,255,0.01);cursor:pointer;font-size:13px}
    .day input{display:none}
    .day.active{background:linear-gradient(90deg, rgba(6,182,212,0.12), rgba(6,182,212,0.06));border-color:rgba(6,182,212,0.3)}
    button{padding:10px 12px;border-radius:10px;border:0;cursor:pointer;background:var(--accent);color:#042027;font-weight:600}
    .actions{display:flex;gap:8px}
    .list{margin-top:18px}
    .day-block{margin-top:12px}
    .day-title{font-weight:700;margin:6px 0 8px}
    ul{list-style:none;padding:0;margin:0;display:grid;gap:8px}
    li{display:flex;justify-content:space-between;gap:12px;align-items:center;padding:10px;border-radius:10px;background:rgba(255,255,255,0.02);border:1px solid rgba(255,255,255,0.02)}
    .meta{display:flex;gap:8px;align-items:center}
    .badge{font-size:13px;padding:6px 8px;border-radius:8px;background:rgba(255,255,255,0.03)}
    .small{font-size:13px;color:var(--muted)}
    .btn-ghost{background:transparent;border:1px solid rgba(255,255,255,0.04);color:var(--muted);padding:6px 8px;border-radius:8px}
    .controls{display:flex;gap:8px}
    .muted{color:var(--muted)}
    footer{margin-top:18px;text-align:center;color:var(--muted);font-size:13px}
    @media (max-width:640px){form{grid-template-columns:1fr;}}
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div class="card" style="padding:12px;border-radius:10px;display:flex;align-items:center;gap:10px">
        <svg width="34" height="34" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 2v6" stroke="#06b6d4" stroke-width="1.4" stroke-linecap="round" stroke-linejoin="round"/><path d="M20 6l-4 19H8L4 6" stroke="#06b6d4" stroke-width="1.4" stroke-linecap="round" stroke-linejoin="round"/></svg>
        <div>
          <h1>Jadwal Lari Harian</h1>
          <div class="small muted">Atur hari & jam. Data tersimpan di browser (LocalStorage).</div>
        </div>
      </div>
    </header>

    <section class="card">
      <form id="form">
        <div>
          <label for="title">Nama Aktivitas</label>
          <input id="title" type="text" placeholder="Contoh: Lari 5K / Interval" required />

          <div style="margin-top:10px">
            <label>Pilih Hari (boleh lebih dari 1)</label>
            <div class="days" id="days">
              <!-- days injected by JS -->
            </div>
          </div>
        </div>

        <div>
          <label for="time">Jam</label>
          <input id="time" type="time" required />

          <div style="margin-top:10px">
            <label class="small">Aksi</label>
            <div class="actions">
              <button id="addBtn" type="submit">Tambah</button>
              <button id="exportBtn" type="button" class="btn-ghost">Export</button>
              <button id="importBtn" type="button" class="btn-ghost">Import</button>
            </div>
          </div>
        </div>
      </form>

      <div class="list" id="scheduleList">
        <!-- schedule rendered here -->
      </div>

      <footer>
        Tip: ketuk nama hari untuk memilih atau mengosongkan. Tombol export menghasilkan JSON, import menerima JSON.
      </footer>
    </section>
  </div>

  <script>
    // Simple running schedule app — single-file
    const DAYS = ['Senin','Selasa','Rabu','Kamis','Jumat','Sabtu','Minggu'];
    const KEY = 'runningSchedule_v1';

    const daysContainer = document.getElementById('days');
    const form = document.getElementById('form');
    const titleInput = document.getElementById('title');
    const timeInput = document.getElementById('time');
    const scheduleList = document.getElementById('scheduleList');
    const addBtn = document.getElementById('addBtn');
    const exportBtn = document.getElementById('exportBtn');
    const importBtn = document.getElementById('importBtn');

    // state
    let items = load();
    let editingId = null;

    // render day buttons
    function renderDayButtons(){
      daysContainer.innerHTML = '';
      DAYS.forEach((d, i) => {
        const el = document.createElement('label');
        el.className = 'day';
        el.dataset.index = i;
        el.innerHTML = `<input type="checkbox" value="${i}" /> <span>${d}</span>`;
        el.addEventListener('click', (e) => {
          // toggle visual class
          el.classList.toggle('active');
          const cb = el.querySelector('input'); cb.checked = !cb.checked;
          e.preventDefault();
        });
        daysContainer.appendChild(el);
      });
    }

    function getSelectedDays(){
      const checked = [];
      daysContainer.querySelectorAll('.day').forEach((el, idx) => {
        const cb = el.querySelector('input');
        if(cb.checked) checked.push(idx);
      });
      return checked;
    }

    function clearForm(){
      titleInput.value = '';
      timeInput.value = '';
      daysContainer.querySelectorAll('.day').forEach(el => { el.classList.remove('active'); el.querySelector('input').checked = false });
      editingId = null;
      addBtn.textContent = 'Tambah';
    }

    function save(){
      localStorage.setItem(KEY, JSON.stringify(items));
    }

    function load(){
      try{
        const raw = localStorage.getItem(KEY);
        if(!raw) return [];
        return JSON.parse(raw);
      }catch(e){
        console.error('Gagal load', e);
        return [];
      }
    }

    function addItem(data){
      items.push(data);
      save();
      render();
    }

    function updateItem(id, data){
      const idx = items.findIndex(x => x.id === id);
      if(idx !== -1){ items[idx] = {...items[idx], ...data}; save(); render(); }
    }

    function deleteItem(id){
      items = items.filter(x => x.id !== id);
      save();
      render();
    }

    function render(){
      // group by day index
      const grouped = {}; DAYS.forEach((d,i)=> grouped[i]=[]);
      items.forEach(it => {
        it.days.forEach(dayIdx => grouped[dayIdx].push(it));
      });

      scheduleList.innerHTML = '';
      DAYS.forEach((day, idx) => {
        const block = document.createElement('div');
        block.className = 'day-block card';
        const title = document.createElement('div');
        title.className = 'day-title';
        title.textContent = day;
        block.appendChild(title);

        const list = document.createElement('ul');
        if(grouped[idx].length === 0){
          const empty = document.createElement('div');
          empty.className = 'small muted';
          empty.textContent = 'Tidak ada jadwal.';
          block.appendChild(empty);
        }else{
          grouped[idx].sort((a,b)=> a.time.localeCompare(b.time));
          grouped[idx].forEach(it => {
            const li = document.createElement('li');
            const meta = document.createElement('div'); meta.className='meta';
            const name = document.createElement('div'); name.innerHTML = `<div style=\"font-weight:700\">${it.title}</div><div class=\"small muted\">${it.time}</div>`;
            meta.appendChild(name);

            const controls = document.createElement('div'); controls.className='controls';
            const editBtn = document.createElement('button'); editBtn.className='btn-ghost'; editBtn.textContent='Edit';
            editBtn.addEventListener('click', ()=> startEdit(it.id));
            const delBtn = document.createElement('button'); delBtn.className='btn-ghost'; delBtn.textContent='Hapus';
            delBtn.addEventListener('click', ()=>{ if(confirm('Hapus jadwal ini?')) deleteItem(it.id) });
            controls.appendChild(editBtn); controls.appendChild(delBtn);

            li.appendChild(meta);
            li.appendChild(controls);
            list.appendChild(li);
          });
          block.appendChild(list);
        }

        scheduleList.appendChild(block);
      });
    }

    function startEdit(id){
      const it = items.find(x=>x.id===id); if(!it) return;
      titleInput.value = it.title;
      timeInput.value = it.time;
      // set day buttons
      daysContainer.querySelectorAll('.day').forEach((el, idx)=>{
        const cb = el.querySelector('input');
        if(it.days.includes(idx)){ el.classList.add('active'); cb.checked = true } else { el.classList.remove('active'); cb.checked = false }
      });
      editingId = id; addBtn.textContent = 'Simpan';
      window.scrollTo({top:0,behavior:'smooth'});
    }

    // form submit
    form.addEventListener('submit', (e)=>{
      e.preventDefault();
      const title = titleInput.value.trim();
      const time = timeInput.value;
      const days = getSelectedDays();
      if(!title || !time || days.length===0){ alert('Isi nama, jam, dan pilih minimal 1 hari.'); return; }

      if(editingId){
        updateItem(editingId, {title, time, days});
      }else{
        addItem({ id: Date.now().toString(36), title, time, days });
      }
      clearForm();
    });

    exportBtn.addEventListener('click', ()=>{
      const data = JSON.stringify(items, null, 2);
      // show in prompt so user can copy
      const ok = confirm('Salin jadwal ke clipboard? (OK = salin)');
      if(ok){ navigator.clipboard.writeText(data).then(()=> alert('Tersalin ke clipboard')) }
    });

    importBtn.addEventListener('click', ()=>{
      const raw = prompt('Tempel JSON jadwal di sini:');
      if(!raw) return;
      try{
        const parsed = JSON.parse(raw);
        if(Array.isArray(parsed)){
          items = parsed; save(); render(); alert('Import berhasil');
        }else alert('Format tidak sesuai (harus array jadwal)');
      }catch(e){ alert('JSON tidak valid') }
    });

    // init
    renderDayButtons();
    render();

    // optional: simple alarm when time matches (tab must be open)
    setInterval(()=>{
      const now = new Date();
      const hhmm = now.toTimeString().slice(0,5);
      const dow = (now.getDay() + 6) % 7; // convert JS (0=Sun) to our 0=Senin
      items.forEach(it=>{
        if(it.time === hhmm && it.days.includes(dow)){
          // only alert once per minute — naive approach
          if(!it._lastAlert || it._lastAlert !== hhmm){
            it._lastAlert = hhmm;
            try{ new Notification('Jadwal Lari', {body: `${it.title} — ${DAYS[dow]} ${it.time}`}) }catch(e){ console.log('Notif blocked or not supported') }
          }
        }
      });
    }, 1000);

    // request notification permission
    if('Notification' in window && Notification.permission !== 'granted'){
      Notification.requestPermission().then(()=>{});
    }
  </script>
</body>
</html>
