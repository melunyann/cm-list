<script>
import jsPDF from 'jspdf'
import html2canvas from 'html2canvas'

const ISSUER_KEY = 'juchu_daichou_issuer_v1'

function todayStr(){
  const d = new Date();
  const pad = n => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
}

export default {
  name: 'DocumentTool',
  data(){
    return {
      docType: 'estimate',
      form: {
        docNo: '',
        clientName: '',
        honorific: '様',
        title: '',
        issueDate: todayStr(),
        validUntil: '',
        dueDate: '',
        deliveryDate: '',
        bankInfo: '',
        notes: ''
      },
      items: [ { id: 1, name: '', qty: 1, unitPrice: 0 } ],
      issuer: { name: '', address: '', tel: '', email: '' },
      generating: false
    };
  },
  computed: {
    docLabel(){
      if(this.docType === 'estimate') return '見積書';
      if(this.docType === 'invoice') return '請求書';
      return '納品書';
    },
    subtotal(){
      return this.items.reduce((sum, it) => sum + (Number(it.qty) || 0) * (Number(it.unitPrice) || 0), 0);
    },
    tax(){
      return Math.round(this.subtotal * 0.1);
    },
    total(){
      return this.subtotal + this.tax;
    }
  },
  watch: {
    issuer: {
      deep: true,
      handler(v){
        try{ localStorage.setItem(ISSUER_KEY, JSON.stringify(v)); }catch(e){}
      }
    }
  },
  methods: {
    addItem(){
      this.items.push({ id: Date.now(), name: '', qty: 1, unitPrice: 0 });
    },
    removeItem(idx){
      if(this.items.length <= 1) return;
      this.items.splice(idx, 1);
    },
    loadIssuer(){
      try{
        const raw = localStorage.getItem(ISSUER_KEY);
        if(raw) this.issuer = JSON.parse(raw);
      }catch(e){ /* noop */ }
    },
    async downloadPDF(){
      this.generating = true;
      await this.$nextTick();
      try{
        const el = this.$refs.previewArea;
        const canvas = await html2canvas(el, { scale: 2, useCORS: true, backgroundColor: '#ffffff' });

        const pdf = new jsPDF({ unit: 'mm', format: 'a4' });
        const pageWidthMm = 210;
        const pageHeightMm = 297;
        const pxPerMm = canvas.width / pageWidthMm;
        const pageHeightPx = Math.floor(pageHeightMm * pxPerMm);

        let renderedPx = 0;
        let firstPage = true;

        while(renderedPx < canvas.height){
          const sliceHeightPx = Math.min(pageHeightPx, canvas.height - renderedPx);
          const sliceCanvas = document.createElement('canvas');
          sliceCanvas.width = canvas.width;
          sliceCanvas.height = sliceHeightPx;
          const ctx = sliceCanvas.getContext('2d');
          ctx.drawImage(
            canvas,
            0, renderedPx, canvas.width, sliceHeightPx,
            0, 0, canvas.width, sliceHeightPx
          );
          const sliceData = sliceCanvas.toDataURL('image/png');
          if(!firstPage) pdf.addPage();
          const sliceHeightMm = sliceHeightPx / pxPerMm;
          pdf.addImage(sliceData, 'PNG', 0, 0, pageWidthMm, sliceHeightMm);
          renderedPx += sliceHeightPx;
          firstPage = false;
        }

        const safeName = (this.form.clientName || '書類').replace(/[\\/:*?"<>|]/g, '');
        pdf.save(`${this.docLabel}_${safeName}_${this.form.issueDate}.pdf`);
      }catch(err){
        console.error(err);
        alert('PDFの作成に失敗しました。もう一度お試しください。');
      }finally{
        this.generating = false;
      }
    }
  },
  mounted(){
    this.loadIssuer();
    if(!this.form.docNo){
      const prefix = this.docType === 'estimate' ? 'EST-' : this.docType === 'invoice' ? 'INV-' : 'DEL-';
      this.form.docNo = prefix + Date.now().toString().slice(-8);
    }
  }
};
</script>

<template>
  <div class="doc-tool">
    <div class="doc-type-tabs">
      <button :class="{active: docType==='estimate'}" @click="docType='estimate'">見積書</button>
      <button :class="{active: docType==='invoice'}" @click="docType='invoice'">請求書</button>
      <button :class="{active: docType==='delivery'}" @click="docType='delivery'">納品書</button>
    </div>

    <div class="doc-grid">

      <!-- ===== 入力フォーム ===== -->
      <div class="doc-form">
        <h3>発行者情報（自社・保存されます）</h3>
        <div class="field"><label>会社名・屋号</label><input type="text" v-model="issuer.name" placeholder="〇〇デザイン"></div>
        <div class="field"><label>住所</label><input type="text" v-model="issuer.address" placeholder="東京都〇〇区…"></div>
        <div class="field"><label>電話番号</label><input type="text" v-model="issuer.tel" placeholder="090-0000-0000"></div>
        <div class="field"><label>メールアドレス</label><input type="text" v-model="issuer.email" placeholder="example@mail.com"></div>

        <h3>宛先・基本情報</h3>
        <div class="field"><label>宛先名</label><input type="text" v-model="form.clientName" placeholder="田中商事"></div>
        <div class="field">
          <label>敬称</label>
          <select v-model="form.honorific">
            <option value="様">様</option>
            <option value="御中">御中</option>
          </select>
        </div>
        <div class="field"><label>件名</label><input type="text" v-model="form.title" placeholder="ロゴデザイン制作"></div>
        <div class="field"><label>書類番号</label><input type="text" v-model="form.docNo"></div>
        <div class="field"><label>発行日</label><input type="date" v-model="form.issueDate"></div>

        <template v-if="docType==='estimate'">
          <div class="field"><label>有効期限</label><input type="date" v-model="form.validUntil"></div>
        </template>
        <template v-else-if="docType==='invoice'">
          <div class="field"><label>支払期限</label><input type="date" v-model="form.dueDate"></div>
        </template>
        <template v-else>
          <div class="field"><label>納品日</label><input type="date" v-model="form.deliveryDate"></div>
        </template>

        <h3>品目</h3>
        <div class="item-row" v-for="(it, idx) in items" :key="it.id">
          <input class="item-name" type="text" v-model="it.name" placeholder="品目名">
          <input class="item-qty" type="number" min="0" v-model.number="it.qty" placeholder="数量">
          <input class="item-price" type="number" min="0" v-model.number="it.unitPrice" placeholder="単価">
          <button class="item-remove" @click="removeItem(idx)" :disabled="items.length===1">×</button>
        </div>
        <button class="btn-ghost" @click="addItem">＋ 品目を追加</button>

        <template v-if="docType==='invoice'">
          <h3>振込先</h3>
          <textarea v-model="form.bankInfo" rows="3" placeholder="〇〇銀行 〇〇支店 普通 1234567 名義：..."></textarea>
        </template>

        <h3>備考</h3>
        <textarea v-model="form.notes" rows="3" placeholder="振込先や補足事項など"></textarea>

        <button class="btn-primary doc-download" @click="downloadPDF" :disabled="generating">
          {{ generating ? '作成中…' : 'PDFダウンロード' }}
        </button>
      </div>

      <!-- ===== プレビュー（この見た目のままPDF化されます） ===== -->
      <div class="doc-preview-wrap">
        <div class="doc-preview" ref="previewArea">
          <div class="doc-header">
            <h1>{{ docLabel }}</h1>
            <div class="doc-meta">
              <div>No. {{ form.docNo }}</div>
              <div>発行日：{{ form.issueDate || '—' }}</div>
              <div v-if="docType==='estimate' && form.validUntil">有効期限：{{ form.validUntil }}</div>
              <div v-if="docType==='invoice' && form.dueDate">お支払期限：{{ form.dueDate }}</div>
              <div v-if="docType==='delivery' && form.deliveryDate">納品日：{{ form.deliveryDate }}</div>
            </div>
          </div>

          <div class="doc-parties">
            <div class="doc-to">{{ form.clientName || '（宛先未入力）' }} {{ form.honorific }}</div>
            <div class="doc-from">
              <div>{{ issuer.name || '（発行者名未入力）' }}</div>
              <div v-if="issuer.address">{{ issuer.address }}</div>
              <div v-if="issuer.tel">{{ issuer.tel }}</div>
              <div v-if="issuer.email">{{ issuer.email }}</div>
            </div>
          </div>

          <div class="doc-title" v-if="form.title">件名：{{ form.title }}</div>

          <table class="doc-table">
            <thead><tr><th>品目</th><th>数量</th><th>単価</th><th>金額</th></tr></thead>
            <tbody>
              <tr v-for="it in items" :key="it.id">
                <td>{{ it.name || '—' }}</td>
                <td class="num">{{ it.qty }}</td>
                <td class="num">¥{{ (it.unitPrice || 0).toLocaleString('ja-JP') }}</td>
                <td class="num">¥{{ ((it.qty || 0) * (it.unitPrice || 0)).toLocaleString('ja-JP') }}</td>
              </tr>
            </tbody>
          </table>
          <div class="doc-totals">
            <div><span>小計</span><span>¥{{ subtotal.toLocaleString('ja-JP') }}</span></div>
            <div><span>消費税（10%）</span><span>¥{{ tax.toLocaleString('ja-JP') }}</span></div>
            <div class="grand"><span>合計金額</span><span>¥{{ total.toLocaleString('ja-JP') }}</span></div>
          </div>

          <div class="doc-notes" v-if="docType==='invoice' && form.bankInfo">
            <h4>お振込先</h4>
            <p>{{ form.bankInfo }}</p>
          </div>

          <div class="doc-notes" v-if="form.notes">
            <h4>備考</h4>
            <p>{{ form.notes }}</p>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
.doc-type-tabs{ display:flex; gap: 4px; margin-bottom: 18px; }
.doc-type-tabs button{
  background: var(--cd); border: 1px solid var(--ls); color: var(--is);
  padding: 8px 18px; font-size: 13.5px; font-family: inherit; cursor: pointer;
  border-radius: var(--rd);
}
.doc-type-tabs button.active{ background: var(--ik); color: #fff; border-color: var(--ik); }

.doc-grid{
  display: grid; grid-template-columns: minmax(280px, 380px) 1fr;
  gap: 22px; align-items: start;
}
@media (max-width: 860px){
  .doc-grid{ grid-template-columns: 1fr; }
}

.doc-form{
  background: var(--cd); border: 1px solid var(--ln); border-radius: var(--rd);
  padding: 20px; display: flex; flex-direction: column; gap: 10px;
}
.doc-form h3{
  font-size: 12.5px; font-weight: 500; color: var(--is);
  margin: 14px 0 2px; letter-spacing: .03em;
}
.doc-form h3:first-child{ margin-top: 0; }
.field{ display:flex; flex-direction: column; gap: 4px; }
.field label{ font-size: 11.5px; color: var(--is); }
.field input, .field select, .doc-form > textarea, .terms-input{
  font-family: inherit; font-size: 13.5px; padding: 8px 9px;
  border: 1px solid var(--ls); border-radius: var(--rd); background: #fff; color: var(--ik);
}
.field input:focus, .field select:focus, .doc-form > textarea:focus, .terms-input:focus{
  outline: none; border-color: var(--ac);
}
.doc-form > textarea{ resize: vertical; font-family: inherit; }

.item-row{ display: grid; grid-template-columns: 1fr 60px 90px 26px; gap: 6px; align-items: center; }
.item-row input{
  font-family: inherit; font-size: 13px; padding: 7px 8px;
  border: 1px solid var(--ls); border-radius: var(--rd); background: #fff; color: var(--ik);
}
.item-remove{
  background: none; border: 1px solid var(--ls); color: var(--is);
  border-radius: var(--rd); cursor: pointer; height: 32px;
}
.item-remove:hover:not(:disabled){ border-color: var(--dg); color: var(--dg); }
.item-remove:disabled{ opacity: .35; cursor: default; }

.terms-input{ resize: vertical; line-height: 1.6; }

.doc-download{ margin-top: 8px; width: 100%; }

.doc-preview-wrap{ overflow-x: auto; }
.doc-preview{
  background: #fff; color: #26302A; width: 210mm; min-height: 297mm;
  padding: 18mm 16mm; box-sizing: border-box; margin: 0 auto;
  box-shadow: 0 0 0 1px var(--ln);
  font-family: 'Zen Kaku Gothic New', sans-serif;
}
.doc-header{ display:flex; justify-content: space-between; align-items: flex-end; border-bottom: 2px solid var(--ik); padding-bottom: 10px; margin-bottom: 22px; }
.doc-header h1{ font-family: 'Zen Old Mincho', serif; font-size: 26px; margin: 0; letter-spacing: .1em; }
.doc-meta{ text-align: right; font-size: 11.5px; color: var(--is); line-height: 1.7; }

.doc-parties{ display:flex; justify-content: space-between; margin-bottom: 18px; }
.doc-to{ font-size: 16px; font-weight: 600; border-bottom: 1px solid var(--ik); padding-bottom: 6px; min-width: 200px; }
.doc-from{ font-size: 12px; color: var(--ik); text-align: right; line-height: 1.7; }

.doc-title{ font-size: 13.5px; margin-bottom: 16px; padding-bottom: 8px; border-bottom: 1px solid var(--ln); }

.doc-table{ width: 100%; border-collapse: collapse; margin-bottom: 14px; }
.doc-table th{
  text-align: left; font-size: 11.5px; font-weight: 500; color: var(--is);
  background: var(--pp); padding: 8px 10px; border: 1px solid var(--ls);
}
.doc-table td{ font-size: 13px; padding: 9px 10px; border: 1px solid var(--ln); }
.doc-table td.num, .doc-table th:nth-child(n+2){ text-align: right; }

.doc-totals{ margin-left: auto; width: 260px; font-size: 13.5px; }
.doc-totals div{ display:flex; justify-content: space-between; padding: 6px 4px; }
.doc-totals .grand{ border-top: 2px solid var(--ik); font-weight: 700; font-size: 15px; margin-top: 4px; padding-top: 10px; }

.doc-terms{ white-space: pre-wrap; font-size: 12.5px; line-height: 1.9; }

.doc-notes{ margin-top: 20px; padding-top: 12px; border-top: 1px solid var(--ln); }
.doc-notes h4{ font-size: 12px; color: var(--is); margin: 0 0 6px; }
.doc-notes p{ font-size: 12.5px; white-space: pre-wrap; margin: 0; }
</style>
