// ==UserScript==
// @name         rehab_ time add V062
// @namespace    https://digikar.jp/
// @version      0.62
// @description  受付一覧の各行で「受付メモ」列の鉛筆だけを開き、PTがあれば予約HH:MMを挿入→保存（途中で開かなくなる問題に対処）
// @match        https://digikar.jp/*
// @run-at       document-end
// @grant        none
// ==/UserScript==

(function () {
  'use strict';

  const DEBUG = true;
  const log = (...a) => DEBUG && console.log('[DK-AUTO-PT]', ...a);
  const warn = (...a) => DEBUG && console.warn('[DK-AUTO-PT]', ...a);

  // ===== 調整パラメータ =====
  const LOOP_DELAY_MS = 350;
  const OPEN_TIMEOUT_MS = 2500;
  const SAVE_TIMEOUT_MS = 3500;
  const MAX_ROWS = 500;

  const CLICK_RETRY = 3;           // 「受付メモが開かない」時のクリック再試行回数
  const CLICK_RETRY_DELAY = 120;

  const norm = (s) =>
    (s ?? '')
      .toString()
      .replace(/\u00A0/g, ' ')
      .replace(/\s+/g, ' ')
      .trim();

  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

  // ====== UI（Start/Stop） ======
  let isRunning = false;
  let stopRequested = false;

  function createControlPanel() {
  if (document.getElementById('dkAutoPtPanel')) return;

  const POS_KEY = 'dkAutoPtPanelPos';
  const MIN_KEY = 'dkAutoPtPanelMin';

  const panel = document.createElement('div');
  panel.id = 'dkAutoPtPanel';
  panel.style.position = 'fixed';
  panel.style.left = 'auto';
  panel.style.top = 'auto';
  panel.style.right = '12px';
  panel.style.bottom = '12px';
  panel.style.zIndex = '2147483647';         // 可能な限り最前面
  panel.style.background = 'white';
  panel.style.border = '1px solid #ccc';
  panel.style.borderRadius = '10px';
  panel.style.padding = '10px 12px';
  panel.style.boxShadow = '0 6px 18px rgba(0,0,0,0.20)';
  panel.style.fontSize = '12px';
  panel.style.lineHeight = '1.4';
  panel.style.minWidth = '240px';
  panel.style.userSelect = 'none';
  panel.style.pointerEvents = 'auto';        // クリックを確実に拾う

  // 位置復元
  try {
    const saved = JSON.parse(localStorage.getItem(POS_KEY) || 'null');
    if (saved && typeof saved.left === 'number' && typeof saved.top === 'number') {
      panel.style.left = saved.left + 'px';
      panel.style.top = saved.top + 'px';
      panel.style.right = 'auto';
      panel.style.bottom = 'auto';
    }
  } catch {}

  // ヘッダ（ドラッグハンドル）
  const header = document.createElement('div');
  header.style.display = 'flex';
  header.style.alignItems = 'center';
  header.style.justifyContent = 'space-between';
  header.style.cursor = 'grab';
  header.style.gap = '8px';
  header.style.marginBottom = '8px';

  const title = document.createElement('div');
  title.textContent = '受付メモ：PT時刻 自動保存';
  title.style.fontWeight = '700';
  title.style.flex = '1';

  const miniBtn = document.createElement('button');
  miniBtn.type = 'button';
  miniBtn.textContent = '−';
  miniBtn.style.width = '28px';
  miniBtn.style.height = '22px';
  miniBtn.style.lineHeight = '18px';
  miniBtn.style.padding = '0';
  miniBtn.style.cursor = 'pointer';

  header.appendChild(title);
  header.appendChild(miniBtn);

  // 本体（最小化対象）
  const body = document.createElement('div');
  body.id = 'dkAutoPtBody';

  const status = document.createElement('div');
  status.id = 'dkAutoPtStatus';
  status.textContent = '待機中';
  status.style.marginBottom = '8px';

  const btnStart = document.createElement('button');
  btnStart.type = 'button';
  btnStart.textContent = 'Start';
  btnStart.style.marginRight = '6px';

  const btnStop = document.createElement('button');
  btnStop.type = 'button';
  btnStop.textContent = 'Stop';

  const hint = document.createElement('div');
  hint.textContent = '※実行中は手操作を控える';
  hint.style.marginTop = '8px';
  hint.style.color = '#555';

  btnStart.onclick = () => {
    if (isRunning) return;
    stopRequested = false;
    runAllVisibleRows().catch((e) => warn('run error', e));
  };
  btnStop.onclick = () => {
    stopRequested = true;
    status.textContent = '停止要求…（現在の1件処理後に停止）';
  };

  body.appendChild(status);
  body.appendChild(btnStart);
  body.appendChild(btnStop);
  body.appendChild(hint);

  panel.appendChild(header);
  panel.appendChild(body);
  document.body.appendChild(panel);

  // 最小化状態復元
  let minimized = false;
  try { minimized = localStorage.getItem(MIN_KEY) === '1'; } catch {}
  if (minimized) {
    body.hidden = true;
    miniBtn.textContent = '+';
    panel.style.minWidth = '140px';
  }

  // 最小化トグル（display:noneよりhiddenが安定することがある）
  miniBtn.onclick = (e) => {
    e.stopPropagation();
    minimized = !minimized;
    body.hidden = minimized;
    miniBtn.textContent = minimized ? '+' : '−';
    panel.style.minWidth = minimized ? '140px' : '240px';
    try { localStorage.setItem(MIN_KEY, minimized ? '1' : '0'); } catch {}
  };

  // --- ドラッグ移動（captureで確実に拾う） ---
  let dragging = false;
  let startX = 0, startY = 0, startLeft = 0, startTop = 0;

  const clamp = (v, min, max) => Math.max(min, Math.min(max, v));

  const onMove = (e) => {
    if (!dragging) return;
    const dx = e.clientX - startX;
    const dy = e.clientY - startY;

    const newLeft = clamp(startLeft + dx, 0, window.innerWidth - 80);
    const newTop  = clamp(startTop  + dy, 0, window.innerHeight - 40);

    panel.style.left = newLeft + 'px';
    panel.style.top  = newTop + 'px';
  };

  const onUp = () => {
    if (!dragging) return;
    dragging = false;
    header.style.cursor = 'grab';
    panel.style.cursor = 'default';
    document.removeEventListener('mousemove', onMove, true);
    document.removeEventListener('mouseup', onUp, true);

    try {
      const rect = panel.getBoundingClientRect();
      localStorage.setItem(POS_KEY, JSON.stringify({ left: rect.left, top: rect.top }));
    } catch {}
  };

  header.addEventListener('mousedown', (e) => {
    // 最小化ボタン押下はドラッグ開始しない
    if (e.target === miniBtn) return;

    dragging = true;
    startX = e.clientX;
    startY = e.clientY;

    const rect = panel.getBoundingClientRect();
    startLeft = rect.left;
    startTop = rect.top;

    panel.style.left = startLeft + 'px';
    panel.style.top = startTop + 'px';
    panel.style.right = 'auto';
    panel.style.bottom = 'auto';

    header.style.cursor = 'grabbing';
    panel.style.cursor = 'grabbing';

    // captureでページ側のハンドラに邪魔されにくくする
    document.addEventListener('mousemove', onMove, true);
    document.addEventListener('mouseup', onUp, true);

    e.preventDefault();
  }, true);
}



  function setStatus(text) {
    const el = document.getElementById('dkAutoPtStatus');
    if (el) el.textContent = text;
  }

  // ====== “毎回” 行を取り直す（古い参照を捨てる） ======
  function getAllRowsFresh() {
    return Array.from(document.querySelectorAll('tbody tr')).slice(0, MAX_ROWS);
  }

  // ====== ヘッダindex（tableは差し替わるので、table参照ごとに取り直す方が安全） ======
  function findColIndexByHeaderCandidates(table, candidates) {
    if (!table) return -1;
    const ths = Array.from(table.querySelectorAll('thead th'));
    const headers = ths.map((th) => norm(th.textContent));
    for (const c of candidates) {
      const idx = headers.indexOf(c);
      if (idx >= 0) return idx;
    }
    return -1;
  }

  function getTdByIndex(tr, idx) {
    if (!tr || idx < 0) return null;
    const tds = Array.from(tr.querySelectorAll('td'));
    return tds[idx] || null;
  }

  function getCellText(td) {
    if (!td) return '';
    const firstText = Array.from(td.childNodes)
      .filter((n) => n.nodeType === Node.TEXT_NODE)
      .map((n) => norm(n.textContent))
      .find((t) => t);
    return firstText || norm(td.textContent);
  }

  // ====== 予約HH:MM ======
  function extractHHMM(s) {
    const m = norm(s).match(/\b([01]?\d|2[0-3]):([0-5]\d)\b/);
    return m ? `${m[1].padStart(2, '0')}:${m[2]}` : '';
  }

  function getReserveHHMM(tr) {
    const table = tr?.closest('table');
    const reserveIdx = findColIndexByHeaderCandidates(table, ['予約', '予約時刻', '予約時間', '予約日時']);
    const td = getTdByIndex(tr, reserveIdx);
    return extractHHMM(getCellText(td));
  }

  // ====== ✅受付メモ列の鉛筆ボタン（ヘッダで列確定→そのtd内） ======
  function findReceptionMemoEditButton(tr) {
    const table = tr?.closest('table');
    const memoIdx = findColIndexByHeaderCandidates(table, ['受付メモ', 'メモ', '受付ﾒﾓ']);
    if (memoIdx < 0) return null;
    const td = getTdByIndex(tr, memoIdx);
    if (!td) return null;
    return td.querySelector('button.edit-icon[type="button"]') || null;
  }

  // ====== 受付メモ textarea ======
  function findMemoTextarea() {
    const list = Array.from(document.querySelectorAll('textarea'));

    const byAttr = list.find((ta) => {
      const p = norm(ta.getAttribute('placeholder'));
      const a = norm(ta.getAttribute('aria-label'));
      const n = norm(ta.getAttribute('name'));
      return [p, a, n].some((x) => x.includes('メモ') || x.includes('受付メモ'));
    });
    if (byAttr) return byAttr;

    for (const ta of list) {
      const container = ta.closest('div,section,form,dialog') || ta.parentElement;
      const t = norm(container?.textContent);
      if (t.includes('受付メモ')) return ta;
    }

    if (list.length === 1) return list[0];
    return null;
  }

  function waitForMemoTextarea(timeoutMs = OPEN_TIMEOUT_MS) {
    return new Promise((resolve) => {
      const quick = findMemoTextarea();
      if (quick) return resolve(quick);

      const obs = new MutationObserver(() => {
        const ta = findMemoTextarea();
        if (ta) {
          obs.disconnect();
          resolve(ta);
        }
      });
      obs.observe(document.documentElement, { childList: true, subtree: true });

      setTimeout(() => {
        obs.disconnect();
        resolve(findMemoTextarea());
      }, timeoutMs);
    });
  }

  // ====== 保存ボタン ======
  function findSaveButton() {
    const buttons = Array.from(document.querySelectorAll('button[type="button"]'));
    return (
      buttons.find(
        (b) =>
          norm(b.textContent) === '保存' &&
          (b.getAttribute('data-variant') === 'primary' || b.className.includes('primary'))
      ) || null
    );
  }

  async function waitUntil(predicate, timeoutMs = 2000, intervalMs = 80) {
    const start = Date.now();
    while (Date.now() - start < timeoutMs) {
      if (predicate()) return true;
      await sleep(intervalMs);
    }
    return !!predicate();
  }

  // ====== React制御textarea対策 ======
  function setTextareaValueReactSafe(ta, value) {
    const proto = window.HTMLTextAreaElement.prototype;
    const desc = Object.getOwnPropertyDescriptor(proto, 'value');
    const setter = desc && desc.set;

    if (setter) setter.call(ta, value);
    else ta.value = value;

    ta.dispatchEvent(new InputEvent('input', { bubbles: true, inputType: 'insertText' }));
    ta.dispatchEvent(new Event('change', { bubbles: true }));
    ta.dispatchEvent(new Event('blur', { bubbles: true }));
  }

  // ====== PT挿入ロジック（V0.50相当） ======
  function escapeRegExp(s) {
    return (s ?? '').replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  }
  function memoLooksLikePT(memo) {
    return norm(memo).includes('PT');
  }
  function extractPTTokens(memo) {
    const t = memo ?? '';
    const re = /([^\s（）()]{1,12}?PT)/g;
    const out = [];
    let m;
    while ((m = re.exec(t)) !== null) {
      out.push(m[1]);
      if (out.length >= 10) break;
    }
    return Array.from(new Set(out));
  }
  function injectTimeAfterPT(memoText, ptToken, hhmm) {
    const text = memoText ?? '';
    if (!ptToken || !hhmm) return { next: text, changed: false };

    const already = new RegExp(`${escapeRegExp(ptToken)}\\s+([01]?\\d|2[0-3]):[0-5]\\d`);
    if (already.test(text)) return { next: text, changed: false };

    const open = '[（(]';
    const close = '[）)]';

    const inParen = new RegExp(`(${open}[^）)]*?)(${escapeRegExp(ptToken)})(\\s*${close})`);
    if (inParen.test(text)) {
      const next = text.replace(inParen, `$1$2 ${hhmm}$3`);
      return { next, changed: next !== text };
    }

    const plain = new RegExp(`(${escapeRegExp(ptToken)})(?!\\s*([01]?\\d|2[0-3]):[0-5]\\d)`);
    const next2 = text.replace(plain, `$1 ${hhmm}`);
    return { next: next2, changed: next2 !== text };
  }

  function rowLooksProcessed(tr) {
    const t = norm(tr?.textContent);
    return /PT\s+([01]?\d|2[0-3]):[0-5]\d/.test(t);
  }

  // ====== 強いクリック（.click + MouseEvent） ======
  function hardClick(el) {
    if (!el) return;
    el.scrollIntoView({ block: 'center', inline: 'nearest' });
    try { el.focus?.(); } catch {}
    // まず普通にclick
    el.click();
    // 念のためMouseEventも投げる
    const opts = { bubbles: true, cancelable: true, view: window };
    el.dispatchEvent(new MouseEvent('mousedown', opts));
    el.dispatchEvent(new MouseEvent('mouseup', opts));
    el.dispatchEvent(new MouseEvent('click', opts));
  }

  // ====== モーダルが詰まった時：ESCで閉じる ======
  function pressEsc() {
    const ev = new KeyboardEvent('keydown', { key: 'Escape', code: 'Escape', bubbles: true });
    document.dispatchEvent(ev);
  }

  // ====== 1行処理 ======
  async function openMemoEditorWithRetries(tr) {
    // もし前のモーダルが残っていたら先に閉じる
    if (findMemoTextarea()) {
      pressEsc();
      await sleep(200);
    }

    for (let k = 1; k <= CLICK_RETRY; k++) {
      const btn = findReceptionMemoEditButton(tr);
      if (!btn) return null;

      hardClick(btn);

      const ta = await waitForMemoTextarea(OPEN_TIMEOUT_MS);
      if (ta) return ta;

      // 開かなかった → 少し待って再試行（スクロールや再描画を吸収）
      await sleep(CLICK_RETRY_DELAY * k);
    }
    return null;
  }

  async function processRow(tr, index, total) {
    if (!tr) return { ok: false, skipped: true, reason: 'no tr' };

    if (rowLooksProcessed(tr)) return { ok: true, skipped: true, reason: 'already processed' };

    const hhmm = getReserveHHMM(tr);
    if (!hhmm) return { ok: true, skipped: true, reason: 'no reserve time' };

    setStatus(`処理中 ${index + 1}/${total} … 受付メモを開く`);
    const ta = await openMemoEditorWithRetries(tr);
    if (!ta) return { ok: false, skipped: true, reason: 'cannot open memo editor' };

    const cur = ta.value ?? '';
    if (!memoLooksLikePT(cur)) {
      pressEsc();
      return { ok: true, skipped: true, reason: 'memo has no PT' };
    }

    const pts = extractPTTokens(cur);
    if (!pts.length) {
      pressEsc();
      return { ok: true, skipped: true, reason: 'no PT tokens' };
    }

    let next = cur;
    let changedAny = false;
    for (const ptToken of pts) {
      const r = injectTimeAfterPT(next, ptToken, hhmm);
      next = r.next;
      changedAny = changedAny || r.changed;
    }

    if (!changedAny) {
      pressEsc();
      return { ok: true, skipped: true, reason: 'already has time or no match' };
    }

    setStatus(`処理中 ${index + 1}/${total} … 挿入→保存`);
    setTextareaValueReactSafe(ta, next);

    const saveBtn = findSaveButton();
    if (!saveBtn) {
      pressEsc();
      return { ok: false, skipped: true, reason: 'no save button' };
    }

    hardClick(saveBtn);

    // 保存後に閉じるまで待つ（閉じない場合も続行できるよう保険）
    const closed = await waitUntil(() => !findMemoTextarea(), SAVE_TIMEOUT_MS, 100);
    if (!closed) {
      // 詰まっていたらESCで抜けて続行
      pressEsc();
      await sleep(200);
      warn('保存後クローズ判定が取れず、ESCで続行');
    }

    return { ok: true, skipped: false, reason: 'updated & saved' };
  }

  // ====== ループ本体：行は毎回freshに取り直す ======
  async function runAllVisibleRows() {
    createControlPanel();

    isRunning = true;
    stopRequested = false;

    let updated = 0, skipped = 0, failed = 0;

    // “表示範囲の先頭から末尾まで” を、都度取り直しながら進める
    // indexで進める（古い参照のまま進まない）
    let i = 0;

    while (true) {
      if (stopRequested) break;

      const rows = getAllRowsFresh();
      const total = rows.length;

      if (!total) {
        setStatus('行が見つかりません');
        break;
      }
      if (i >= total) break;

      // ここで最新のtrを取得
      const tr = rows[i];

      try {
        const r = await processRow(tr, i, total);
        if (!r.ok && !r.skipped) failed++;
        else if (r.skipped) skipped++;
        else updated++;

        setStatus(`進捗 ${i + 1}/${total} ｜更新:${updated} スキップ:${skipped} 失敗:${failed}`);
        if (DEBUG) log('row', i + 1, r);

      } catch (e) {
        failed++;
        warn('row process error', e);
      }

      i++;
      await sleep(LOOP_DELAY_MS);
    }

    isRunning = false;
    if (stopRequested) setStatus(`停止しました｜更新:${updated} スキップ:${skipped} 失敗:${failed}`);
    else setStatus(`完了｜更新:${updated} スキップ:${skipped} 失敗:${failed}`);
  }

  // 起動
  createControlPanel();
  log('loaded V0.62');
})();
