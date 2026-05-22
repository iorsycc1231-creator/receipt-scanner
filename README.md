import { useState, useRef, useCallback } from 'react'

// ── Gemini API ────────────────────────────────────────────────────
const PROMPT = `このレシート・領収書の画像を解析して、以下のJSON形式のみで返してください。
説明文・マークダウン・コードブロックは一切不要です。JSONオブジェクトだけを返してください。

{
  "store_name": "店名（不明なら空文字）",
  "invoice_number": "インボイス登録番号（T始まり13桁、なければ空文字）",
  "date": "日付（YYYY-MM-DD形式、不明なら空文字）",
  "items": [
    {
      "name": "商品名・サービス名",
      "amount": 金額(数値),
      "tax_category": "10%標準" または "8%軽減" または "非課税" または "不明"
    }
  ],
  "subtotal": 小計(数値またはnull),
  "tax_8": 消費税8%額(数値またはnull),
  "tax_10": 消費税10%額(数値またはnull),
  "total": 合計金額(数値またはnull),
  "notes": "特記事項（軽減税率マーク等）"
}`

async function callGemini(apiKey, base64Image, mimeType) {
  const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      contents: [{
        parts: [
          { inline_data: { mime_type: mimeType, data: base64Image } },
          { text: PROMPT }
        ]
      }],
      generationConfig: { temperature: 0.1, maxOutputTokens: 1500 }
    })
  })

  if (!res.ok) {
    const err = await res.json().catch(() => ({}))
    const msg = err?.error?.message || `HTTPエラー ${res.status}`
    throw new Error(`Gemini APIエラー: ${msg}`)
  }

  const data = await res.json()
  const raw = data?.candidates?.[0]?.content?.parts?.[0]?.text || ''
  return parseJson(raw)
}

function parseJson(raw) {
  if (!raw) throw new Error('AIからの応答が空でした')
  let text = raw.trim()
    .replace(/^```(?:json)?\s*/i, '')
    .replace(/\s*```\s*$/, '')
    .trim()
  const s = text.indexOf('{'), e = text.lastIndexOf('}')
  if (s === -1 || e === -1) throw new Error(`JSONが見つかりません\n応答: ${text.slice(0, 300)}`)
  try {
    return JSON.parse(text.slice(s, e + 1))
  } catch (err) {
    throw new Error(`JSONパースエラー: ${err.message}`)
  }
}

// ── CSV ───────────────────────────────────────────────────────────
function toCSV(receipts) {
  const h = ['No','店名','インボイス番号','日付','商品名','金額','税区分','小計','消費税(8%)','消費税(10%)','合計金額','備考']
  const esc = v => { const s = String(v ?? ''); return /[,"\n]/.test(s) ? `"${s.replace(/"/g,'""')}"` : s }
  const rows = []
  receipts.forEach((r, i) => {
    const items = r.items?.length ? r.items : [{ name:'', amount:'', tax_category:'' }]
    items.forEach((item, j) => rows.push([
      j===0 ? i+1 : '', j===0 ? r.store_name||'' : '', j===0 ? r.invoice_number||'' : '',
      j===0 ? r.date||'' : '', item.name||'', item.amount??'', item.tax_category||'',
      j===0 ? r.subtotal??'' : '', j===0 ? r.tax_8??'' : '', j===0 ? r.tax_10??'' : '',
      j===0 ? r.total??'' : '', j===0 ? r.notes||'' : ''
    ]))
  })
  return [h, ...rows].map(r => r.map(esc).join(',')).join('\n')
}

// ── App ───────────────────────────────────────────────────────────
export default function App() {
  const [apiKey, setApiKey] = useState(() => localStorage.getItem('gemini_key') || '')
  const [showKey, setShowKey] = useState(false)
  const [receipts, setReceipts] = useState([])
  const [dragging, setDragging] = useState(false)
  const fileRef = useRef()

  const saveKey = (k) => { setApiKey(k); localStorage.setItem('gemini_key', k) }

  const process = useCallback(async (file) => {
    if (!file.type.startsWith('image/')) return
    const id = Date.now() + Math.random()
    const preview = URL.createObjectURL(file)
    setReceipts(p => [...p, { id, preview, name: file.name, status:'loading', data:null, error:null }])

    try {
      if (!apiKey.trim()) throw new Error('APIキーが入力されていません')
      const b64 = await new Promise((res, rej) => {
        const r = new FileReader()
        r.onload = () => res(r.result.split(',')[1])
        r.onerror = () => rej(new Error('ファイル読み込み失敗'))
        r.readAsDataURL(file)
      })
      const data = await callGemini(apiKey.trim(), b64, file.type)
      setReceipts(p => p.map(r => r.id===id ? {...r, status:'done', data} : r))
    } catch(err) {
      setReceipts(p => p.map(r => r.id===id ? {...r, status:'error', error:err.message} : r))
    }
  }, [apiKey])

  const handleFiles = (files) => [...files].forEach(process)

  const downloadCSV = () => {
    const done = receipts.filter(r => r.status==='done' && r.data).map(r => r.data)
    if (!done.length) return
    const blob = new Blob(['\uFEFF' + toCSV(done)], { type:'text/csv;charset=utf-8;' })
    const a = Object.assign(document.createElement('a'), {
      href: URL.createObjectURL(blob),
      download: `経費精算_${new Date().toISOString().slice(0,10)}.csv`
    })
    a.click()
  }

  const doneCount  = receipts.filter(r => r.status==='done').length
  const errCount   = receipts.filter(r => r.status==='error').length
  const loadCount  = receipts.filter(r => r.status==='loading').length

  return (
    <div style={S.root}>
      {/* ── Header ── */}
      <header style={S.header}>
        <div style={S.brand}>
          <div style={S.brandMark}>R</div>
          <div>
            <div style={S.brandName}>Receipt Scan</div>
            <div style={S.brandSub}>Gemini AI · 経費精算CSV</div>
          </div>
        </div>
        {doneCount > 0 && (
          <button style={S.csvBtn} onClick={downloadCSV}>
            ⬇ CSVダウンロード <span style={S.badge}>{doneCount}件</span>
          </button>
        )}
      </header>

      <div style={S.inner}>
        {/* ── API Key ── */}
        <section style={S.keyBox}>
          <div style={S.keyLabel}>🔑 Gemini APIキー</div>
          <div style={S.keyRow}>
            <input
              type={showKey ? 'text' : 'password'}
              placeholder="AIza... で始まるキーを入力"
              value={apiKey}
              onChange={e => saveKey(e.target.value)}
              style={S.keyInput}
            />
            <button style={S.eyeBtn} onClick={() => setShowKey(v => !v)}>
              {showKey ? '🙈' : '👁'}
            </button>
          </div>
          <div style={S.keyHint}>
            キーはこのデバイスにのみ保存されます。取得先：
            <a href="https://aistudio.google.com" target="_blank" rel="noreferrer" style={S.link}>
              aistudio.google.com
            </a>
            （無料）
          </div>
        </section>

        {/* ── Drop zone ── */}
        <div
          style={{...S.drop, ...(dragging ? S.dropActive : {})}}
          onDragOver={e => { e.preventDefault(); setDragging(true) }}
          onDragLeave={() => setDragging(false)}
          onDrop={e => { e.preventDefault(); setDragging(false); handleFiles(e.dataTransfer.files) }}
          onClick={() => fileRef.current?.click()}
        >
          <input ref={fileRef} type="file" accept="image/*" multiple style={{display:'none'}}
            onChange={e => handleFiles(e.target.files)} />
          <div style={S.dropEmoji}>📷</div>
          <div style={S.dropTitle}>レシートをドロップ・タップして選択</div>
          <div style={S.dropSub}>JPG · PNG · HEIC · 複数枚同時OK</div>
        </div>

        {/* ── Stats ── */}
        {receipts.length > 0 && (
          <div style={S.stats}>
            <Chip label={`合計 ${receipts.length}枚`} />
            {loadCount > 0 && <Chip label={`⏳ 解析中 ${loadCount}`} color="#d97706" />}
            {doneCount > 0 && <Chip label={`✓ 完了 ${doneCount}`} color="#16a34a" />}
            {errCount  > 0 && <Chip label={`✗ エラー ${errCount}`}  color="#dc2626" />}
          </div>
        )}

        {/* ── Cards ── */}
        <div style={S.grid}>
          {receipts.map(r => (
            <ReceiptCard
              key={r.id}
              receipt={r}
              onRemove={() => setReceipts(p => p.filter(x => x.id !== r.id))}
            />
          ))}
        </div>
      </div>
    </div>
  )
}

// ── Receipt Card ──────────────────────────────────────────────────
function ReceiptCard({ receipt, onRemove }) {
  const { preview, status, data, error } = receipt
  const [showErr, setShowErr] = useState(false)

  return (
    <div style={S.card}>
      <div style={S.imgWrap}>
        <img src={preview} alt="" style={S.img} />
        <button style={S.closeBtn} onClick={onRemove} title="削除">✕</button>
        {status === 'loading' && (
          <div style={S.overlay}>
            <div style={S.spinner} />
            <div style={S.overlayTxt}>Geminiが解析中…</div>
          </div>
        )}
        {status === 'done'  && <div style={{...S.statusTag, background:'#16a34a'}}>✓ 読取完了</div>}
        {status === 'error' && <div style={{...S.statusTag, background:'#dc2626'}}>✗ エラー</div>}
      </div>

      <div style={S.cardBody}>
        {status === 'loading' && <p style={S.muted}>解析中です…</p>}

        {status === 'error' && (
          <div style={S.errBox}>
            <div style={S.errTitle}>⚠ 解析に失敗しました</div>
            <button style={S.errToggle} onClick={() => setShowErr(v => !v)}>
              {showErr ? '詳細を隠す ▲' : '詳細を見る ▼'}
            </button>
            {showErr && <pre style={S.errPre}>{error}</pre>}
          </div>
        )}

        {status === 'done' && data && (
          <>
            <Row label="店名"       value={data.store_name} />
            <Row label="日付"       value={data.date} />
            <Row label="インボイス" value={data.invoice_number || '—'} />
            <hr style={S.hr} />
            {data.items?.map((item, i) => (
              <div key={i} style={S.itemRow}>
                <span style={S.itemName}>{item.name || '—'}</span>
                <span style={S.itemRight}>
                  <span style={S.taxTag}>{item.tax_category}</span>
                  <span style={S.itemAmt}>¥{Number(item.amount||0).toLocaleString()}</span>
                </span>
              </div>
            ))}
            <hr style={S.hr} />
            {data.tax_8  != null && <Row label="消費税 8%"  value={`¥${data.tax_8.toLocaleString()}`} />}
            {data.tax_10 != null && <Row label="消費税 10%" value={`¥${data.tax_10.toLocaleString()}`} />}
            <Row label="合計" value={data.total != null ? `¥${data.total.toLocaleString()}` : '—'} bold />
            {data.notes && <p style={S.notes}>{data.notes}</p>}
          </>
        )}
      </div>
    </div>
  )
}

function Row({ label, value, bold }) {
  return (
    <div style={S.row}>
      <span style={S.rowL}>{label}</span>
      <span style={{...S.rowV, ...(bold ? S.rowBold : {})}}>{value || '—'}</span>
    </div>
  )
}

function Chip({ label, color = '#78716c' }) {
  return <span style={{...S.chip, color, borderColor: color + '55'}}>{label}</span>
}

// ── Styles ────────────────────────────────────────────────────────
const S = {
  root:      { minHeight:'100vh', background:'#f5f3ee' },
  header:    { display:'flex', alignItems:'center', justifyContent:'space-between', padding:'16px 20px', background:'#fffdf8', borderBottom:'2px solid #1c1917', position:'sticky', top:0, zIndex:10 },
  brand:     { display:'flex', alignItems:'center', gap:12 },
  brandMark: { width:38, height:38, background:'#1c1917', color:'#fffdf8', display:'flex', alignItems:'center', justifyContent:'center', fontSize:20, fontWeight:900, borderRadius:4 },
  brandName: { fontSize:17, fontWeight:700, letterSpacing:'0.04em' },
  brandSub:  { fontSize:11, color:'#78716c', marginTop:1 },
  csvBtn:    { background:'#16a34a', color:'#fff', border:'none', borderRadius:8, padding:'9px 16px', fontSize:13, fontWeight:600, cursor:'pointer', display:'flex', alignItems:'center', gap:8 },
  badge:     { background:'rgba(255,255,255,0.25)', borderRadius:10, padding:'1px 8px', fontSize:12 },

  inner:  { maxWidth:960, margin:'0 auto', padding:'20px 16px 60px' },

  keyBox:   { background:'#fffdf8', border:'1px solid #e7e5e0', borderRadius:12, padding:'16px 18px', marginBottom:16 },
  keyLabel: { fontSize:13, fontWeight:700, color:'#57534e', marginBottom:10 },
  keyRow:   { display:'flex', gap:8 },
  keyInput: { flex:1, background:'#f5f3ee', border:'1px solid #d6d3d1', borderRadius:8, padding:'10px 12px', fontSize:14, color:'#1c1917', fontFamily:'monospace', outline:'none' },
  eyeBtn:   { background:'none', border:'1px solid #d6d3d1', borderRadius:8, padding:'0 14px', cursor:'pointer', fontSize:16 },
  keyHint:  { fontSize:11, color:'#a8a29e', marginTop:8, lineHeight:1.6 },
  link:     { color:'#16a34a', textDecoration:'none', marginLeft:4 },

  drop:       { background:'#fffdf8', border:'2px dashed #d6d3d1', borderRadius:12, padding:'40px 20px', textAlign:'center', cursor:'pointer', transition:'all 0.2s', marginBottom:16 },
  dropActive: { borderColor:'#16a34a', background:'#f0fdf4' },
  dropEmoji:  { fontSize:40, marginBottom:10 },
  dropTitle:  { fontSize:16, fontWeight:700, marginBottom:6 },
  dropSub:    { fontSize:13, color:'#78716c' },

  stats: { display:'flex', gap:8, flexWrap:'wrap', marginBottom:16 },
  chip:  { border:'1px solid', borderRadius:20, padding:'4px 12px', fontSize:12 },

  grid: { display:'grid', gridTemplateColumns:'repeat(auto-fill,minmax(280px,1fr))', gap:16 },

  card:      { background:'#fffdf8', border:'1px solid #e7e5e0', borderRadius:12, overflow:'hidden', boxShadow:'0 1px 4px rgba(0,0,0,0.06)' },
  imgWrap:   { position:'relative', height:180, background:'#e8e5de', overflow:'hidden' },
  img:       { width:'100%', height:'100%', objectFit:'cover' },
  closeBtn:  { position:'absolute', top:8, right:8, background:'rgba(0,0,0,0.5)', color:'#fff', border:'none', borderRadius:'50%', width:28, height:28, cursor:'pointer', fontSize:13, lineHeight:'28px', textAlign:'center', padding:0 },
  overlay:   { position:'absolute', inset:0, background:'rgba(245,243,238,0.88)', display:'flex', flexDirection:'column', alignItems:'center', justifyContent:'center', gap:12 },
  overlayTxt:{ fontSize:13, color:'#57534e', fontWeight:600 },
  spinner:   { width:32, height:32, border:'3px solid #e7e5e0', borderTop:'3px solid #16a34a', borderRadius:'50%', animation:'spin 0.8s linear infinite' },
  statusTag: { position:'absolute', bottom:8, left:8, color:'#fff', fontSize:11, padding:'3px 10px', borderRadius:4, fontWeight:700 },

  cardBody: { padding:'14px 16px 16px' },
  muted:    { color:'#a8a29e', fontSize:13, textAlign:'center', padding:'8px 0' },

  errBox:    { background:'#fef2f2', border:'1px solid #fca5a5', borderRadius:8, padding:'12px 14px' },
  errTitle:  { color:'#dc2626', fontWeight:700, fontSize:13, marginBottom:6 },
  errToggle: { background:'none', border:'none', color:'#dc2626', cursor:'pointer', fontSize:12, padding:0 },
  errPre:    { fontSize:11, color:'#b91c1c', whiteSpace:'pre-wrap', wordBreak:'break-all', marginTop:8, background:'rgba(0,0,0,0.04)', padding:10, borderRadius:6, maxHeight:160, overflowY:'auto', lineHeight:1.5 },

  row:     { display:'flex', justifyContent:'space-between', alignItems:'center', padding:'4px 0', gap:8 },
  rowL:    { fontSize:12, color:'#78716c', flexShrink:0 },
  rowV:    { fontSize:13, color:'#1c1917', textAlign:'right', wordBreak:'break-all' },
  rowBold: { fontWeight:800, color:'#16a34a', fontSize:15 },
  hr:      { border:'none', borderTop:'1px solid #e7e5e0', margin:'8px 0' },

  itemRow:   { display:'flex', justifyContent:'space-between', alignItems:'center', padding:'3px 0', gap:6 },
  itemName:  { fontSize:12, color:'#78716c', flex:1, overflow:'hidden', textOverflow:'ellipsis', whiteSpace:'nowrap' },
  itemRight: { display:'flex', alignItems:'center', gap:6, flexShrink:0 },
  taxTag:    { fontSize:10, background:'#f0fdf4', color:'#16a34a', border:'1px solid #bbf7d0', borderRadius:4, padding:'1px 6px' },
  itemAmt:   { fontSize:13, fontVariantNumeric:'tabular-nums' },
  notes:     { fontSize:11, color:'#a8a29e', marginTop:8, fontStyle:'italic', lineHeight:1.5 },
}
