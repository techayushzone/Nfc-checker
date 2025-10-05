# Nfc-checker
<!doctype html>
<html lang="hi">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>लखनऊ मेट्रो कार्ड — NFC बैलेंस चेकर (Demo)</title>
<style>
  :root{--bg:#f6f8fa;--card:#fff;--muted:#6b7280;--accent:#0b84ff}
  body{font-family:system-ui,-apple-system,Segoe UI,Roboto,'Noto Sans',Arial; background:var(--bg); margin:0; padding:20px; display:flex; align-items:center; justify-content:center; min-height:100vh}
  .wrap{width:100%;max-width:520px;background:var(--card);border-radius:12px;box-shadow:0 6px 24px rgba(15,23,42,.08);padding:20px}
  h1{font-size:20px;margin:0 0 8px}
  p.lead{margin:0 0 16px;color:var(--muted)}
  .row{display:flex;gap:8px;flex-wrap:wrap}
  button{background:var(--accent);color:#fff;border:0;padding:12px 14px;border-radius:10px;font-size:15px;cursor:pointer}
  button.secondary{background:#e6eefc;color:#05345a}
  button[disabled]{opacity:.5;cursor:not-allowed}
  #status{margin-top:12px;color:#334155;font-weight:600}
  #result{margin-top:14px;background:#f8fafc;padding:12px;border-radius:8px;border:1px solid #e6eefc;font-size:14px;white-space:pre-wrap;display:none}
  .small{font-size:13px;color:var(--muted)}
  .hint{margin-top:10px;font-size:13px;color:#334155}
  .mono{font-family:ui-monospace,SFMono-Regular,Menlo,Monaco,monospace;background:#0f1724;color:#e6eefc;padding:6px;border-radius:6px;display:inline-block;font-size:13px}
  .success{color:#059669}
  .error{color:#dc2626}
  footer{margin-top:14px;font-size:13px;color:var(--muted)}
</style>
</head>
<body>
  <div class="wrap" role="main">
    <h1>लखनऊ मेट्रो कार्ड बैलेंस चेकर — Demo</h1>
    <p class="lead">Chrome (Android) + NFC required. पेज HTTPS पर होना चाहिए।</p>

    <div class="row">
      <button id="startBtn">🔍 NFC स्कैन शुरू करें</button>
      <button id="stopBtn" class="secondary" disabled>⛔ स्कैन बंद करें</button>
    </div>

    <div id="status" class="small">तैयार — बटन दबाएँ।</div>

    <div id="result" aria-live="polite"></div>

    <p class="hint">नोट: कई मेट्रो कार्ड्स NDEF नहीं रखते — इसलिए यह demo raw टैग डेटा दिखायेगा। वास्तविक बैलेंस निकालने के लिए official app / backend चाहिए।</p>

    <footer>टेस्ट के लिए: NDEF टैग (text) का उपयोग करना सबसे आसान है।</footer>
  </div>

<script>
(async () => {
  const startBtn = document.getElementById('startBtn');
  const stopBtn = document.getElementById('stopBtn');
  const statusEl = document.getElementById('status');
  const resultEl = document.getElementById('result');

  function setStatus(html, cls='') {
    statusEl.innerHTML = html;
    statusEl.className = 'small ' + (cls||'');
  }
  function showResult(html){
    resultEl.style.display = 'block';
    resultEl.innerHTML = html;
  }
  function hideResult(){ resultEl.style.display = 'none'; resultEl.innerHTML=''; }

  // Feature checks
  if (!('NDEFReader' in window)) {
    setStatus('⚠️ आपका ब्राउज़र Web NFC सपोर्ट नहीं करता। Chrome (Android) उपयोग करें।', 'error');
    startBtn.disabled = true;
    return;
  }
  if (location.protocol !== 'https:') {
    setStatus('⚠️ HTTPS आवश्यक है — यह पेज secure context पर होनी चाहिए। (GitHub Pages / ngrok / Netlify उपयोग करें)', 'error');
    // still allow devs to click if they know what they're doing
  }

  let reader = null;
  let abortController = null;
  let autoTimeout = null;

  function toHex(buffer) {
    const bytes = new Uint8Array(buffer);
    return Array.from(bytes).map(b => b.toString(16).padStart(2,'0')).join(' ');
  }

  startBtn.addEventListener('click', async () => {
    hideResult();
    startBtn.disabled = true;
    stopBtn.disabled = false;
    setStatus('स्कैन शुरू किया जा रहा है... क्रोम permission पॉपअप दिख सकता है।');

    try {
      reader = new NDEFReader();
      abortController = new AbortController();

      // Auto-abort after 30s (safety)
      autoTimeout = setTimeout(() => {
        if (abortController) abortController.abort();
      }, 30000);

      await reader.scan({ signal: abortController.signal });
      setStatus('✅ स्कैन active — कार्ड को फोन के पीछे लगाएँ।', 'success');

      reader.onreading = (evt) => {
        clearTimeout(autoTimeout);
        const ndef = evt.message;
        let text = '';
        text += 'Timestamp: ' + new Date().toLocaleString() + '\\n\\n';

        if (ndef.records && ndef.records.length) {
          text += '--- NDEF Records ---\\n';
          ndef.records.forEach((r, i) => {
            text += `Record ${i}: t=${r.recordType}, mt=${r.mediaType||'--'}\\n`;
            // try decode text or URL
            try {
              if (r.recordType === 'text' || r.recordType === 'url' || r.recordType === 'mime') {
                const td = new TextDecoder();
                const decoded = td.decode(r.data);
                text += `  -> Decoded: ${decoded}\\n`;
                text += `  -> Hex: ${toHex(r.data)}\\n`;
              } else {
                // generic hex output
                text += `  -> Raw Hex: ${toHex(r.data)}\\n`;
              }
            } catch(e){
              text += `  -> (decode error) hex: ${toHex(r.data)}\\n`;
            }
          });
        } else {
          text += 'NDEF रिकॉर्ड्स नहीं मिले। यह टैग non-NDEF या protected हो सकता है.\\n';
        }

        // event.serialNumber (if available)
        if (evt.serialNumber) {
          text += '\\nTag serialNumber: ' + evt.serialNumber + '\\n';
        }

        // Try simple balance regex on decoded text
        const balMatch = text.match(/BAL(?:ANCE)?:?\\s*₹?\\s*(\\d+(?:\\.\\d{1,2})?)/i) || text.match(/\\b(\\d+\\.\\d{2})\\b/);
        let balanceStr = balMatch ? '₹' + balMatch[1] : 'बैलेंस not found (demo).';

        showResult(`<strong>स्कैन सफल!</strong>\\n\\n<strong>Detect summary:</strong>\\n${text}\\n<strong>Detected balance (heuristic):</strong> ${balanceStr}\\n\\n<small class="small">नोट: वास्तविक मेट्रो कार्ड्स protected हो सकते हैं — raw hex देखें और official app प्रयोग करें।</small>`);
        setStatus('स्कैन पूरा — रिज़ल्ट ऊपर दिख रहा है।', 'success');

        // automatically stop scan after first read
        try { abortController && abortController.abort(); } catch(e){}
        startBtn.disabled = false;
        stopBtn.disabled = true;
      };

      reader.onreadingerror = (evt) => {
        clearTimeout(autoTimeout);
        setStatus('❌ पढ़ने में त्रुटि हुई — टैग को ठीक से लगाएँ या डिवाइस रीस्टार्ट करें।', 'error');
        startBtn.disabled = false;
        stopBtn.disabled = true;
      };

    } catch (err) {
      clearTimeout(autoTimeout);
      console.error(err);
      let msg = 'स्कैन शुरू नहीं हो सका: ' + (err && err.message ? err.message : String(err));
      if (err.name === 'NotAllowedError') msg = 'Permission denied — Allow in Chrome prompt.';
      if (err.name === 'NotSupportedError') msg = 'यह डिवाइस/ब्राउज़र Web NFC support नहीं करता।';
      if (err.name === 'AbortError') msg = 'स्कैन टाइमआउट या रद्द किया गया।';
      setStatus('❌ ' + msg, 'error');
      startBtn.disabled = false;
      stopBtn.disabled = true;
    }
  });

  stopBtn.addEventListener('click', () => {
    try {
      abortController && abortController.abort();
    } catch(e){}
    clearTimeout(autoTimeout);
    startBtn.disabled = false;
    stopBtn.disabled = true;
    setStatus('स्कैन रोका गया।', '');
  });

})();
</script>
</body>
</html>
