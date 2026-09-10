<script>
import DocumentTool from './DocumentTool.vue'

const STORAGE_KEY = 'juchu_daichou_vue_v1';
const STATUS_LABELS = ['受注', '作業中', '納品済'];

function todayStr(){
  const d = new Date();
  const pad = n => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
}

function blankForm(){
  return {
    client: '', platform: '', account: '', plan: '', content: '',
    amount: 0, qty: 1, status: 0, paid: false,
    date: todayStr(), deadline: '', completedDate: '', memo: ''
  };
}

export default {
  components: { DocumentTool },
  data(){
    return {
      view: 'dashboard',
      entries: [],
      editingId: null,
      form: blankForm(),
      listFilter: { search: '', status: 'all', paid: 'all', year: 'all', plan: 'all' },
      STATUS_LABELS
    };
  },

  computed: {
    activeCount(){
      return this.entries.filter(e => e.status === 1).length;
    },
    activeList(){
      return this.entries
        .filter(e => e.status === 1)
        .sort((a, b) => (a.deadline || '').localeCompare(b.deadline || ''));
    },
    unpaidCount(){
      return this.entries.filter(e => !e.paid).length;
    },
    dueThisMonthList(){
      const ym = todayStr().slice(0, 7);
      return this.entries
        .filter(e => e.deadline && e.deadline.slice(0, 7) === ym && e.status !== 2)
        .sort((a, b) => a.deadline.localeCompare(b.deadline));
    },
    dueSoonList(){
      const today = new Date(); today.setHours(0, 0, 0, 0);
      const soon = new Date(today); soon.setDate(soon.getDate() + 7);
      return this.entries
        .filter(e => {
          if(!e.deadline || e.status === 2) return false;
          const d = new Date(e.deadline);
          return d >= today && d <= soon;
        })
        .sort((a, b) => a.deadline.localeCompare(b.deadline));
    },
    unpaidList(){
      return this.entries
        .filter(e => !e.paid)
        .sort((a, b) => (a.deadline || '').localeCompare(b.deadline || ''));
    },
    filteredEntries(){
      return this.entries.filter(e => {
        if(this.listFilter.status !== 'all' && String(e.status) !== this.listFilter.status) return false;
        if(this.listFilter.paid === 'paid' && !e.paid) return false;
        if(this.listFilter.paid === 'unpaid' && e.paid) return false;
        if(this.listFilter.year !== 'all' && (e.date || '').slice(0, 4) !== this.listFilter.year) return false;
        if(this.listFilter.plan !== 'all' && (e.plan || '') !== this.listFilter.plan) return false;
        if(this.listFilter.search){
          const q = this.listFilter.search.toLowerCase();
          if(!e.client.toLowerCase().includes(q) && !(e.content || '').toLowerCase().includes(q)) return false;
        }
        return true;
      }).sort((a, b) => (a.orderNo || 0) - (b.orderNo || 0));
    },
    yearOptions(){
      const years = new Set(this.entries.map(e => (e.date || '').slice(0, 4)).filter(Boolean));
      return Array.from(years).sort((a, b) => b.localeCompare(a));
    },
    planOptions(){
      const plans = new Set(this.entries.map(e => e.plan).filter(Boolean));
      return Array.from(plans).sort();
    },
    filteredRevenue(){
      return this.filteredEntries.reduce((sum, e) => sum + (Number(e.amount) || 0), 0);
    },
    totalRevenue(){
      return this.entries.reduce((sum, e) => sum + (Number(e.amount) || 0), 0);
    },
    totalCount(){
      return this.entries
        .filter(e => e.status === 2)
        .reduce((sum, e) => sum + (Number(e.qty) || 1), 0);
    },
    revenueByYear(){
      const map = {};
      this.entries.forEach(e => {
        const y = (e.date || '').slice(0, 4) || '不明';
        map[y] = (map[y] || 0) + (Number(e.amount) || 0);
      });
      return Object.entries(map)
        .sort((a, b) => b[0].localeCompare(a[0]))
        .map(([year, total]) => ({ year, total }));
    }
  },

  methods: {
    loadEntries(){
      try{
        const raw = localStorage.getItem(STORAGE_KEY);
        this.entries = raw ? JSON.parse(raw) : [];
      }catch(e){
        this.entries = [];
      }
    },
    saveEntries(){
      try{
        localStorage.setItem(STORAGE_KEY, JSON.stringify(this.entries));
      }catch(e){
        alert('保存に失敗しました。ブラウザの設定をご確認ください');
      }
    },
    openNew(){
      this.editingId = null;
      this.form = blankForm();
      this.view = 'detail';
    },
    openDetail(id){
      const e = this.entries.find(x => x.id === id);
      if(!e) return;
      this.editingId = id;
      this.form = { ...e };
      this.view = 'detail';
    },
    saveForm(){
      if(!this.form.client || !this.form.content){
        alert('依頼者名と依頼内容は必須です');
        return;
      }
      if(this.form.status === 2 && !this.form.completedDate){
        this.form.completedDate = todayStr();
      }
      if(this.editingId){
        const idx = this.entries.findIndex(e => e.id === this.editingId);
        this.entries[idx] = { ...this.form, id: this.editingId, orderNo: this.entries[idx].orderNo };
      }else{
        const nextNo = this.entries.reduce((m, e) => Math.max(m, e.orderNo || 0), 0) + 1;
        this.entries.push({ ...this.form, id: Date.now(), orderNo: nextNo });
      }
      this.saveEntries();
      this.view = 'list';
    },
    deleteEntry(){
      if(!confirm('この依頼を削除しますか？元に戻せません。')) return;
      this.entries = this.entries.filter(e => e.id !== this.editingId);
      this.saveEntries();
      this.view = 'list';
    },
    exportJSON(){
      const blob = new Blob([JSON.stringify(this.entries, null, 2)], { type: 'application/json' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `juchu-daichou-backup-${todayStr()}.json`;
      a.click();
      URL.revokeObjectURL(url);
    },
    importJSON(event){
      const file = event.target.files[0];
      if(!file) return;
      const reader = new FileReader();
      reader.onload = e => {
        try{
          const data = JSON.parse(e.target.result);
          if(!Array.isArray(data)) throw new Error('not an array');
          if(!confirm(`${data.length}件のデータを読み込みます。現在のデータは上書きされます。よろしいですか？`)){
            event.target.value = '';
            return;
          }
          this.entries = data;
          this.saveEntries();
          alert('読み込みが完了しました');
        }catch(err){
          alert('JSONの読み込みに失敗しました。ファイルの形式をご確認ください');
        }
        event.target.value = '';
      };
      reader.readAsText(file);
    }
  },

  mounted(){
    this.loadEntries();
  }
};
</script>

<template>
  <div id="app">

    <div class="topbar">
      <h1>受注帳</h1>
      <nav class="tabs">
        <button :class="{active: view==='dashboard'}" @click="view='dashboard'">ダッシュボード</button>
        <button :class="{active: view==='list'}" @click="view='list'">受注一覧</button>
        <button :class="{active: view==='doc'}" @click="view='doc'">各種書類</button>
      </nav>
      <div class="backup-actions">
        <button @click="exportJSON">JSON書き出し</button>
        <label class="file-btn">JSON読み込み<input type="file" accept="application/json" @change="importJSON"></label>
      </div>
    </div>

    <!-- ============ Dashboard ============ -->
    <section v-show="view==='dashboard'">
      <div class="stat-grid">
        <div class="stat">
          <div class="label">作業中の数</div>
          <div class="value">{{ activeCount }}件</div>
        </div>
        <div class="stat">
          <div class="label">今月納期の案件</div>
          <div class="value">{{ dueThisMonthList.length }}件</div>
        </div>
        <div class="stat warn">
          <div class="label">支払い待ちの案件</div>
          <div class="value">{{ unpaidCount }}件</div>
        </div>
        <div class="stat accent">
          <div class="label">納期が近い案件</div>
          <div class="value">{{ dueSoonList.length }}件</div>
        </div>
        <div class="stat">
          <div class="label">全体の売上</div>
          <div class="value">¥{{ totalRevenue.toLocaleString('ja-JP') }}</div>
        </div>
        <div class="stat">
          <div class="label">総合の実績数</div>
          <div class="value">{{ totalCount }}件</div>
        </div>
      </div>


      <div class="dash-section">
        <h3>納期が近い案件</h3>
        <ul class="mini-list" v-if="dueSoonList.length">
          <li v-for="e in dueSoonList" :key="e.id" @click="openDetail(e.id)">
            <span>{{ e.client }}（{{ e.plan || e.content }}）</span>
            <span class="deadline">{{ e.deadline }}</span>
          </li>
        </ul>
        <p class="dash-empty" v-else>直近7日以内が納期の案件はありません。</p>
      </div>

      <div class="dash-section">
        <h3>今月納期の案件</h3>
        <ul class="mini-list" v-if="dueThisMonthList.length">
          <li v-for="e in dueThisMonthList" :key="e.id" @click="openDetail(e.id)">
            <span>{{ e.client }}（{{ e.plan || e.content }}）</span>
            <span class="deadline">{{ e.deadline }}</span>
          </li>
        </ul>
        <p class="dash-empty" v-else>今月が納期の案件はありません。</p>
      </div>

            <div class="dash-section">
        <h3>作業中の案件</h3>
        <ul class="mini-list" v-if="activeList.length">
          <li v-for="e in activeList" :key="e.id" @click="openDetail(e.id)">
            <span>{{ e.client }}（{{ e.plan || e.content }}）</span>
            <span class="deadline">{{ STATUS_LABELS[e.status] }}</span>
          </li>
        </ul>
        <p class="dash-empty" v-else>現在作業中の案件はありません。</p>
      </div>

      <div class="dash-section">
        <h3>支払い待ちの案件</h3>
        <ul class="mini-list" v-if="unpaidList.length">
          <li v-for="e in unpaidList" :key="e.id" @click="openDetail(e.id)">
            <span>{{ e.client }}（{{ e.plan || e.content }}）</span>
            <span class="deadline">¥{{ e.amount.toLocaleString('ja-JP') }}</span>
          </li>
        </ul>
        <p class="dash-empty" v-else>支払い待ちの案件はありません。</p>
      </div>

      <div class="dash-section">
        <h3>年別売上</h3>
        <ul class="mini-list" v-if="revenueByYear.length">
          <li v-for="r in revenueByYear" :key="r.year">
            <span>{{ r.year }}年</span>
            <span class="deadline">¥{{ r.total.toLocaleString('ja-JP') }}</span>
          </li>
        </ul>
        <p class="dash-empty" v-else>まだ売上データがありません。</p>
      </div>
    </section>

    <!-- ============ List ============ -->
    <section v-show="view==='list'">
      <div class="list-toolbar">
        <button class="btn-primary" @click="openNew">＋ 新規記帳</button>
        <input type="text" v-model="listFilter.search" placeholder="依頼者・依頼内容で検索…">
        <select v-model="listFilter.status">
          <option value="all">すべての状態</option>
          <option value="0">受注</option>
          <option value="1">作業中</option>
          <option value="2">納品済</option>
        </select>
        <select v-model="listFilter.paid">
          <option value="all">支払すべて</option>
          <option value="paid">支払い済み</option>
          <option value="unpaid">未払い</option>
        </select>
        <select v-model="listFilter.year">
          <option value="all">すべての年</option>
          <option v-for="y in yearOptions" :key="y" :value="y">{{ y }}年</option>
        </select>
        <select v-model="listFilter.plan">
          <option value="all">すべてのプラン</option>
          <option v-for="p in planOptions" :key="p" :value="p">{{ p }}</option>
        </select>
      </div>

      <div class="filtered-summary" v-if="filteredEntries.length">
        絞り込み結果：{{ filteredEntries.length }}件／売上合計 <strong>¥{{ filteredRevenue.toLocaleString('ja-JP') }}</strong>
      </div>

      <div class="table-wrap" v-if="filteredEntries.length">
        <table>
          <thead>
            <tr><th>依頼者名</th><th>プラン</th><th>金額</th><th>状態</th><th>納期</th></tr>
          </thead>
          <tbody>
            <tr v-for="e in filteredEntries" :key="e.id" @click="openDetail(e.id)">
              <td>{{ e.client }}</td>
              <td>{{ e.plan || '—' }}</td>
              <td class="amount">¥{{ e.amount.toLocaleString('ja-JP') }}</td>
              <td>
                <span class="status-pill">
                  <span class="dot" :class="'st' + e.status"></span>{{ STATUS_LABELS[e.status] }}
                </span>
              </td>
              <td class="date">{{ e.deadline || '—' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
      <p class="empty" v-else>まだ記帳がありません。「＋ 新規記帳」から最初の依頼を記帳しましょう。</p>
    </section>

    <!-- ============ Document tool ============ -->
    <section v-show="view==='hearing'">
  <HearingSheet v-if="view==='hearing'" />
</section>
    <section v-show="view==='doc'">
      <DocumentTool v-if="view==='doc'" />
    </section>

    <!-- ============ Detail / edit ============ -->
    <section class="detail-view" v-show="view==='detail'">
      <button class="btn-ghost" @click="view='list'">← 一覧に戻る</button>
      <h2>{{ editingId ? '依頼の編集' : '新しい依頼を記帳' }}</h2>

      <div class="form-grid">
        <div class="field">
          <label>依頼者名</label>
          <input type="text" v-model="form.client" placeholder="田中さま">
        </div>
        <div class="field">
          <label>プラットフォーム</label>
          <input type="text" v-model="form.platform" placeholder="ココナラ">
        </div>
        <div class="field">
          <label>アカウント</label>
          <input type="text" v-model="form.account" placeholder="出品アカウント名など">
        </div>
        <div class="field">
          <label>プラン</label>
          <input type="text" v-model="form.plan" placeholder="ベーシックプランなど">
        </div>
        <div class="field wide">
          <label>依頼内容</label>
          <input type="text" v-model="form.content" placeholder="ロゴデザイン一式">
        </div>
        <div class="field">
          <label>金額（円）</label>
          <input type="number" v-model.number="form.amount" min="0" step="1">
        </div>
        <div class="field">
          <label>点数</label>
          <input type="number" v-model.number="form.qty" min="1" step="1">
        </div>
        <div class="field">
          <label>状態</label>
          <select v-model.number="form.status">
            <option :value="0">受注</option>
            <option :value="1">作業中</option>
            <option :value="2">納品済</option>
          </select>
        </div>
        <div class="field">
          <label>支払い状況</label>
          <label class="checkbox-row">
            <input type="checkbox" v-model="form.paid">
            <span>支払い済み</span>
          </label>
        </div>
        <div class="field">
          <label>受付日</label>
          <input type="date" v-model="form.date">
        </div>
        <div class="field">
          <label>納期</label>
          <input type="date" v-model="form.deadline">
        </div>
        <div class="field">
          <label>完了日</label>
          <input type="date" v-model="form.completedDate">
        </div>
        <div class="field wide">
          <label>メモ</label>
          <textarea v-model="form.memo" rows="3" placeholder="補足事項など"></textarea>
        </div>
      </div>

      <div class="detail-actions">
        <button v-if="editingId" class="btn-danger" @click="deleteEntry">この依頼を削除</button>
        <span v-else></span>
        <button class="btn-primary" @click="saveForm">保存する</button>
      </div>
    </section>

  </div>
</template>

<style>
:root{
  --pp: #F1F3ED;
  --cd: #FFFFFF;
  --ik: #26302A;
  --is: #6E786E;
  --if: #A2AB9C;
  --ln: #E1E6DB;
  --ls: #CBD4C3;
  --ac: #A9863C;
  --do: #A2AB9C;
  --dp: #A9863C;
  --dd: #4C7C8C;
  --dg: #9A4A42;
  --rd: 4px;
}
@import url('https://fonts.googleapis.com/css2?family=Zen+Old+Mincho:wght@600;700&family=Zen+Kaku+Gothic+New:wght@400;500;700&family=JetBrains+Mono:wght@400;500;700&display=swap');

*{ box-sizing: border-box; }
body{
  background: var(--pp);
  color: var(--ik);
  font-family: 'Zen Kaku Gothic New', sans-serif;
  -webkit-font-smoothing: antialiased;
  min-height: 100vh;
  margin: 0;
}
#app{ max-width: 1080px; margin: 0 auto; padding: 28px 20px 70px; }

.topbar{
  display:flex; align-items:center; justify-content:space-between;
  flex-wrap: wrap; gap: 14px;
  border-bottom: 1px solid var(--ls);
  padding-bottom: 16px; margin-bottom: 24px;
}
.topbar h1{
  font-family: 'Zen Old Mincho', serif;
  font-weight: 700; font-size: 24px; margin: 0; letter-spacing: .05em;
}
.tabs{ display:flex; gap: 4px; }
.tabs button{
  background: none; border: 1px solid transparent; color: var(--is);
  padding: 7px 14px; font-size: 13.5px; font-family: inherit; cursor: pointer;
  border-radius: var(--rd);
}
.tabs button.active{ background: var(--ik); color: #fff; }
.tabs button:not(.active):hover{ border-color: var(--ls); }
.backup-actions{ display:flex; gap: 8px; }
.backup-actions button, .backup-actions .file-btn{
  background: var(--cd); border: 1px solid var(--ls); color: var(--is);
  padding: 7px 12px; font-size: 12px; border-radius: var(--rd); cursor: pointer;
  font-family: inherit; display: inline-flex; align-items: center;
}
.backup-actions button:hover, .backup-actions .file-btn:hover{ border-color: var(--ac); color: var(--ac); }
.file-btn input{ display:none; }

button{ font-family: inherit; }

.stat-grid{
  display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 12px; margin-bottom: 28px;
}
.stat{
  background: var(--cd); border: 1px solid var(--ln); border-radius: var(--rd);
  padding: 16px 16px;
}
.stat .label{ font-size: 12px; color: var(--is); }
.stat .value{
  font-family: 'JetBrains Mono', monospace; font-size: 26px; font-weight: 700;
  margin-top: 6px; color: var(--ik);
}
.stat.warn .value{ color: var(--dg); }
.stat.accent .value{ color: var(--ac); }

.dash-section{ margin-bottom: 24px; }
.dash-section h3{
  font-size: 13px; font-weight: 500; color: var(--is);
  margin: 0 0 10px; letter-spacing: .03em;
}
.mini-list{ list-style:none; margin:0; padding:0; background: var(--cd); border: 1px solid var(--ln); border-radius: var(--rd); overflow: hidden; }
.mini-list li{
  display:flex; justify-content: space-between; align-items:center;
  padding: 10px 14px; font-size: 13.5px; cursor: pointer;
  border-bottom: 1px solid var(--ln);
}
.mini-list li:last-child{ border-bottom: none; }
.mini-list li:hover{ background: #FAFBF8; }
.mini-list .deadline{ font-family: 'JetBrains Mono', monospace; color: var(--is); font-size: 12.5px; }
.dash-empty{ color: var(--if); font-size: 13px; padding: 6px 2px; }

.list-toolbar{ display:flex; gap: 8px; flex-wrap: wrap; align-items:center; margin-bottom: 14px; }
.list-toolbar input[type=text], .list-toolbar select{
  font-family: inherit; font-size: 13px; padding: 7px 9px;
  border: 1px solid var(--ls); border-radius: var(--rd); background: var(--cd); color: var(--ik);
}
.list-toolbar input[type=text]{ flex:1; min-width: 160px; }

.filtered-summary{
  font-size: 12.5px; color: var(--is); margin: -4px 0 12px; padding: 0 2px;
}
.filtered-summary strong{ color: var(--ac); font-family: 'JetBrains Mono', monospace; }

.btn-primary{
  background: var(--ik); color: #fff; border: none; padding: 9px 18px;
  border-radius: var(--rd); font-size: 13.5px; font-weight: 500; cursor: pointer;
}
.btn-primary:hover{ background: #14190f; }
.btn-ghost{
  background: var(--cd); border: 1px solid var(--ls); color: var(--is);
  padding: 8px 14px; border-radius: var(--rd); font-size: 13px; cursor: pointer;
}
.btn-ghost:hover{ border-color: var(--ac); color: var(--ac); }
.btn-danger{
  background: #fff; border: 1px solid var(--dg); color: var(--dg);
  padding: 9px 16px; border-radius: var(--rd); font-size: 13.5px; cursor: pointer;
}
.btn-danger:hover{ background: var(--dg); color: #fff; }

.table-wrap{
  border: 1px solid var(--ls); border-radius: var(--rd);
  background: var(--cd); overflow: hidden;
}
table{ width:100%; border-collapse: collapse; }
thead th{
  text-align:left; font-size: 11.5px; color: var(--is); font-weight: 500;
  padding: 10px 14px; border-bottom: 1px solid var(--ls);
}
tbody td{
  padding: 11px 14px; font-size: 13.5px; border-bottom: 1px solid var(--ln);
  vertical-align: middle;
}
tbody tr{ cursor: pointer; }
tbody tr:hover{ background: #FAFBF8; }
tbody tr:last-child td{ border-bottom: none; }
td.amount{ font-family:'JetBrains Mono', monospace; }
td.date{ font-family:'JetBrains Mono', monospace; color: var(--is); }

.status-pill{
  display:inline-flex; align-items:center; gap:6px;
  font-size: 12.5px; padding: 4px 9px; border-radius: 999px;
  border: 1px solid var(--ls);
}
.status-pill .dot{ width:7px; height:7px; border-radius:50%; }
.dot.st0{ background: var(--do); }
.dot.st1{ background: var(--dp); }
.dot.st2{ background: var(--dd); }

.empty{ padding: 40px 20px; text-align:center; color: var(--is); font-size: 14px; }

.detail-view h2{
  font-family: 'Zen Old Mincho', serif; font-size: 20px; font-weight: 600;
  margin: 18px 0 20px;
}
.form-grid{
  display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px 18px; background: var(--cd); border: 1px solid var(--ln);
  border-radius: var(--rd); padding: 22px 24px; margin-bottom: 20px;
}
.field{ display:flex; flex-direction: column; gap: 6px; }
.field.wide{ grid-column: 1 / -1; }
.field label{ font-size: 12px; color: var(--is); }
.field input, .field select, .field textarea{
  font-family: inherit; font-size: 14px; padding: 9px 10px;
  border: 1px solid var(--ls); border-radius: var(--rd);
  background: #fff; color: var(--ik);
}
.field textarea{ resize: vertical; font-family: inherit; }
.field input:focus, .field select:focus, .field textarea:focus{
  outline: none; border-color: var(--ac);
}
.checkbox-row{
  display:flex; align-items:center; gap: 8px; font-size: 14px;
  padding: 9px 10px; border: 1px solid var(--ls); border-radius: var(--rd);
  cursor: pointer; color: var(--ik);
}
.checkbox-row input{ accent-color: var(--ac); width:16px; height:16px; }

.detail-actions{ display:flex; justify-content: space-between; }

@media (prefers-reduced-motion: reduce){ *{ transition: none !important; } }
</style>
