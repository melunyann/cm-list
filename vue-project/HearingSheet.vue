<script>
import jsPDF from 'jspdf'
import html2canvas from 'html2canvas'

function todayStr(){
  const d = new Date();
  const pad = n => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
}

export default {
  name: 'HearingSheet',
  data(){
    return {
      form: {
        clientName: '',
        hearingDate: todayStr(),
        platform: '',
        overview: '',
        references: '',
        budget: '',
        deadlineHope: '',
        deliverables: '',
        ngPoints: '',
        notes: ''
      },
      generating: false
    };
  },
  methods: {
    resetForm(){
      if(!confirm('入力内容をすべてクリアします。よろしいですか？')) return;
      this.form = {
        clientName: '', hearingDate: todayStr(), platform: '', overview: '',
        references: '', budget: '', deadlineHope: '', deliverables: '',
        ngPoints: '', notes: ''
      };
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

        const safeName = (this.form.clientName || '依頼者').replace(/[\\/:*?"<>|]/g, '');
        pdf.save(`ヒアリングシート_${safeName}_${this.form.hearingDate}.pdf`);
      }catch(err){
        console.error(err);
        alert('PDFの作成に失敗しました。もう一度お試しください。');
      }finally{
        this.generating = false;
      }
    }
  }
};
</script>

<template>
  <div class="doc-tool">
    <div class="doc-grid">

      <!-- ===== 入力フォーム ===== -->
      <div class="doc-form">
        <h3>基本情報</h3>
        <div class="field"><label>依頼者名</label><input type="text" v-model="form.clientName" placeholder="田中さま"></div>
        <div class="field"><label>ヒアリング日</label><input type="date" v-model="form.hearingDate"></div>
        <div class="field"><label>プラットフォーム</label><input type="text" v-model="form.platform" placeholder="ココナラ など"></div>

        <h3>ご要望</h3>
        <div class="field">
          <label>ご希望の内容・イメージ</label>
          <textarea v-model="form.overview" rows="4" placeholder="どんなものを作りたいか、雰囲気、テイストなど"></textarea>
        </div>
        <div class="field">
          <label>参考URL・資料</label>
          <textarea v-model="form.references" rows="2" placeholder="参考にしたいサイトや画像のリンクなど"></textarea>
        </div>
        <div class="field">
          <label>納品物・仕様</label>
          <textarea v-model="form.deliverables" rows="3" placeholder="サイズ、形式、点数など具体的な仕様"></textarea>
        </div>

        <h3>条件</h3>
        <div class="field"><label>ご予算感</label><input type="text" v-model="form.budget" placeholder="〜1万円 など"></div>
        <div class="field"><label>ご希望の納期</label><input type="text" v-model="form.deadlineHope" placeholder="◯月◯日ごろ など"></div>

        <h3>その他</h3>
        <div class="field">
          <label>NGなこと・避けたい表現</label>
          <textarea v-model="form.ngPoints" rows="2" placeholder="使いたくない色、避けたいモチーフなど"></textarea>
        </div>
        <div class="field">
          <label>備考</label>
          <textarea v-model="form.notes" rows="3" placeholder="その他、自由に記入"></textarea>
        </div>

        <button class="btn-primary doc-download" @click="downloadPDF" :disabled="generating">
          {{ generating ? '作成中…' : 'PDFダウンロード' }}
        </button>
        <button class="btn-ghost" @click="resetForm">入力内容をクリア</button>
      </div>

      <!-- ===== プレビュー ===== -->
      <div class="doc-preview-wrap">
        <div class="doc-preview" ref="previewArea">
          <div class="doc-header">
            <h1>ヒアリングシート</h1>
            <div class="doc-meta">
              <div>ヒアリング日：{{ form.hearingDate || '—' }}</div>
              <div v-if="form.platform">プラットフォーム：{{ form.platform }}</div>
            </div>
          </div>

          <div class="doc-parties">
            <div class="doc-to">{{ form.clientName || '（依頼者未入力）' }} 様</div>
          </div>

          <div class="hs-block">
            <h4>ご希望の内容・イメージ</h4>
            <p>{{ form.overview || '—' }}</p>
          </div>
          <div class="hs-block" v-if="form.references">
            <h4>参考URL・資料</h4>
            <p>{{ form.references }}</p>
          </div>
          <div class="hs-block" v-if="form.deliverables">
            <h4>納品物・仕様</h4>
            <p>{{ form.deliverables }}</p>
          </div>

          <div class="hs-row">
            <div class="hs-col">
              <h4>ご予算感</h4>
              <p>{{ form.budget || '—' }}</p>
            </div>
            <div class="hs-col">
              <h4>ご希望の納期</h4>
              <p>{{ form.deadlineHope || '—' }}</p>
            </div>
          </div>

          <div class="hs-block" v-if="form.ngPoints">
            <h4>NGなこと・避けたい表現</h4>
            <p>{{ form.ngPoints }}</p>
          </div>
          <div class="hs-block" v-if="form.notes">
            <h4>備考</h4>
            <p>{{ form.notes }}</p>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
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
.field input, .field textarea{
  font-family: inherit; font-size: 13.5px; padding: 8px 9px;
  border: 1px solid var(--ls); border-radius: var(--rd); background: #fff; color: var(--ik);
}
.field input:focus, .field textarea:focus{ outline: none; border-color: var(--ac); }
.field textarea{ resize: vertical; font-family: inherit; }

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

.doc-parties{ margin-bottom: 22px; }
.doc-to{ font-size: 16px; font-weight: 600; border-bottom: 1px solid var(--ik); padding-bottom: 6px; display: inline-block; min-width: 200px; }

.hs-block{ margin-bottom: 18px; }
.hs-block h4{ font-size: 12px; color: var(--is); margin: 0 0 6px; }
.hs-block p{ font-size: 13px; white-space: pre-wrap; margin: 0; line-height: 1.8; }

.hs-row{ display: flex; gap: 24px; margin-bottom: 18px; }
.hs-col{ flex: 1; }
.hs-col h4{ font-size: 12px; color: var(--is); margin: 0 0 6px; }
.hs-col p{ font-size: 13px; margin: 0; }
</style>
