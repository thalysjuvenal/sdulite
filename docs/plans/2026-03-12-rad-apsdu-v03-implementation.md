# RAD APSDU v0.03 — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Adicionar 17 funcionalidades ao editor SQLite browser-based, cobrindo busca, seguranca e produtividade em massa para consultores Protheus.

**Architecture:** Single HTML file com sql.js (WebAssembly) + AG Grid Community. Todas as funcionalidades sao implementadas como secoes de JS/CSS/HTML dentro do mesmo arquivo. O arquivo sera copiado como `RAD APSDU 0.03.html` para preservar a versao anterior.

**Tech Stack:** HTML5, CSS3 (custom properties), JavaScript vanilla, sql.js v1.10.2, AG Grid Community v31.3.2

---

## Mapa do Arquivo Atual (RAD APSDU 0.02.html — 975 linhas)

| Secao | Linhas | Descricao |
|---|---|---|
| CSS/Style | 10-143 | Variaveis, layout, grid, modais, toasts |
| HTML Body | 146-244 | Header, sidebar, content, footer, modal F&R |
| JS — STATE | 247-252 | Variaveis globais |
| JS — INIT | 254-257 | Inicializacao sql.js |
| JS — FILE INPUT | 259-274 | Upload e drag/drop |
| JS — TOOLBAR | 276-286 | Event listeners botoes |
| JS — LOAD FILE/TABLES/TABLE | 288-362 | Carregamento de dados |
| JS — RENDER GRID | 364-384 | AG Grid setup |
| JS — CELL CHANGED | 386-405 | Tracking de alteracoes |
| JS — SAVE FILE | 407-437 | Salvar banco como download |
| JS — ADD/DELETE/REPLICATE | 439-548 | Operacoes CRUD |
| JS — FIND & REPLACE | 550-596 | Modal busca/substituicao |
| JS — TOAST | 598-603 | Notificacoes |
| JS — UNDO/REDO | 605-648 | Desfazer/refazer |
| JS — FILL HANDLE | 650-756 | Preenchimento arrastar |
| JS — COPY/PASTE | 758-972 | Selecao, copiar, colar |

---

## Ordem de Implementacao

As tasks estao ordenadas por dependencia: funcionalidades de fundacao primeiro, depois as que dependem delas.

---

### Task 1: Setup — Copiar arquivo base

**Files:**
- Create: `RAD APSDU 0.03.html` (copia de `RAD APSDU 0.02.html`)

**Step 1: Copiar o arquivo**

```bash
cp "RAD APSDU 0.02.html" "RAD APSDU 0.03.html"
```

**Step 2: Atualizar o title**

No arquivo `RAD APSDU 0.03.html`, linha 6, alterar:
```html
<title>RAD APSDU v0.03</title>
```

**Step 3: Verificar no browser**

Abrir `RAD APSDU 0.03.html` no navegador. Deve funcionar identico ao 0.02.

**Step 4: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: create v0.03 base from v0.02"
```

---

### Task 2: Backup automatico (Funcionalidade 6)

**Fundacao de seguranca — deve ser implementada antes de qualquer outra edicao.**

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar variavel de backup no STATE (linha ~248)**

Apos `let fileName = '';` adicionar:

```javascript
let originalDbBytes = null; // backup do arquivo original
```

**Step 2: Salvar backup na funcao loadFile (linha ~294)**

Dentro do `reader.onload`, logo apos `db = new SQL.Database(...)`, salvar os bytes originais:

```javascript
originalDbBytes = new Uint8Array(e.target.result).slice(); // copia independente
```

**Step 3: Adicionar botao "Restaurar Original" no header**

No HTML, apos o botao `btn-save`, adicionar:

```html
<button class="btn btn-danger" id="btn-restore" disabled>↩ Restaurar Original</button>
```

**Step 4: Implementar funcao de restauracao**

Na secao TOOLBAR BUTTONS, adicionar listener:

```javascript
document.getElementById('btn-restore').addEventListener('click', restoreOriginal);
```

E a funcao:

```javascript
function restoreOriginal() {
  if (!originalDbBytes || !SQL) return;
  if (!confirm('Restaurar o banco ao estado original? Todas as alteracoes serao perdidas.')) return;
  db = new SQL.Database(originalDbBytes.slice());
  pendingChanges.clear();
  undoStack = []; redoStack = [];
  updateChangesPill();
  loadTables();
  toast('Banco restaurado ao estado original', 'success');
}
```

**Step 5: Habilitar o botao ao carregar arquivo**

Na funcao `loadFile`, dentro do bloco try, adicionar junto aos outros enables:

```javascript
document.getElementById('btn-restore').removeAttribute('disabled');
```

**Step 6: Testar no browser**

1. Abrir um arquivo .sqlite
2. Editar algumas celulas
3. Clicar "Restaurar Original" — deve voltar ao estado original
4. Confirmar que o dialog de confirmacao aparece

**Step 7: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add automatic backup and restore original"
```

---

### Task 3: Modo somente leitura (Funcionalidade 11)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar variavel de estado**

No STATE:

```javascript
let readOnlyMode = false;
```

**Step 2: Adicionar toggle no header**

No HTML header-actions, antes do label "Abrir arquivo":

```html
<button class="btn btn-ghost" id="btn-readonly" disabled>🔒 Somente Leitura</button>
```

**Step 3: Implementar toggle**

```javascript
document.getElementById('btn-readonly').addEventListener('click', toggleReadOnly);

function toggleReadOnly() {
  readOnlyMode = !readOnlyMode;
  const btn = document.getElementById('btn-readonly');
  btn.textContent = readOnlyMode ? '🔓 Modo Edicao' : '🔒 Somente Leitura';
  btn.classList.toggle('btn-primary', readOnlyMode);
  btn.classList.toggle('btn-ghost', !readOnlyMode);

  // Disable/enable action buttons
  const actionBtns = ['btn-save','btn-add-row','btn-replicate','btn-delete-rows','btn-find-replace'];
  actionBtns.forEach(id => {
    const el = document.getElementById(id);
    if (readOnlyMode) el.setAttribute('disabled', '');
    else el.removeAttribute('disabled');
  });

  // Update grid editability
  if (gridApi) {
    const newColDefs = columnDefs.map(c => ({
      ...c,
      editable: c.editable === undefined ? undefined : (readOnlyMode ? false : true)
    }));
    gridApi.setGridOption('columnDefs', newColDefs);
  }

  toast(readOnlyMode ? 'Modo somente leitura ativado' : 'Modo edicao ativado', 'info');
}
```

**Step 4: Habilitar o botao ao carregar**

Na funcao `loadTable`, adicionar `'btn-readonly'` ao array de botoes que sao enabled.

**Step 5: Testar no browser**

1. Abrir arquivo, ativar modo leitura
2. Tentar editar celula — nao deve permitir
3. Botoes de acao devem estar desabilitados
4. Desativar modo leitura — edicao volta a funcionar

**Step 6: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add read-only mode toggle"
```

---

### Task 4: Diff visual nas celulas (Funcionalidade 10)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar CSS para celulas modificadas**

Na secao de estilos:

```css
.cell-modified {
  background: rgba(200, 180, 50, 0.12) !important;
  border-left: 2px solid rgba(200, 180, 50, 0.6) !important;
}
```

**Step 2: Adicionar mapa de valores originais no STATE**

```javascript
let originalValues = new Map(); // Map<"rowIndex:colField", originalValue>
```

**Step 3: Capturar valor original no onCellChanged**

No inicio da funcao `onCellChanged`, antes do tracking:

```javascript
const key = `${idx}:${params.colDef.field}`;
if (!originalValues.has(key)) {
  originalValues.set(key, params.oldValue);
}
```

**Step 4: Implementar cellStyle no AG Grid**

Na funcao `renderGrid`, adicionar ao `defaultColDef`:

```javascript
cellStyle: params => {
  if (!params.data || params.data.__rowIndex === undefined) return null;
  const key = `${params.data.__rowIndex}:${params.colDef.field}`;
  if (originalValues.has(key) && originalValues.get(key) !== params.value) {
    return { background: 'rgba(200, 180, 50, 0.12)', borderLeft: '2px solid rgba(200, 180, 50, 0.6)' };
  }
  return null;
},
tooltipValueGetter: params => {
  if (!params.data || params.data.__rowIndex === undefined) return null;
  const key = `${params.data.__rowIndex}:${params.colDef.field}`;
  if (originalValues.has(key) && originalValues.get(key) !== params.value) {
    return `Original: ${originalValues.get(key) ?? '(vazio)'}`;
  }
  return null;
}
```

E adicionar `enableBrowserTooltips: true` nas opcoes do grid.

**Step 5: Limpar originalValues ao salvar e ao trocar tabela**

Na funcao `saveFile`, apos `pendingChanges.clear()`:

```javascript
originalValues.clear();
```

Na funcao `loadTable`, junto com `pendingChanges.clear()`:

```javascript
originalValues.clear();
```

**Step 6: Testar no browser**

1. Abrir arquivo, editar uma celula
2. Celula deve ficar com fundo amarelado e borda esquerda
3. Hover deve mostrar tooltip com valor original
4. Salvar — destaque deve sumir

**Step 7: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add visual diff highlighting for modified cells"
```

---

### Task 5: Historico de alteracoes com reversao (Funcionalidade 7)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar CSS do painel de historico**

```css
.history-panel {
  width: 300px; flex-shrink: 0; border-left: 1px solid var(--border);
  background: var(--surface); display: none; flex-direction: column; overflow: hidden;
}
.history-panel.visible { display: flex; }
.history-header {
  padding: 14px 16px 10px; font-size: 10px; font-weight: 700;
  letter-spacing: 0.15em; text-transform: uppercase; color: var(--text-muted);
  border-bottom: 1px solid var(--border); display: flex; justify-content: space-between; align-items: center;
}
.history-list { flex: 1; overflow-y: auto; padding: 8px; }
.history-list::-webkit-scrollbar { width: 4px; }
.history-list::-webkit-scrollbar-thumb { background: var(--border); border-radius: 2px; }
.history-item {
  padding: 8px 10px; border-radius: 7px; font-size: 11px;
  font-family: 'JetBrains Mono', monospace; color: var(--text-muted);
  border: 1px solid transparent; margin-bottom: 4px; cursor: default;
  display: flex; flex-direction: column; gap: 4px;
}
.history-item:hover { background: var(--surface2); border-color: var(--border); }
.history-item .h-table { font-weight: 600; color: var(--text); }
.history-item .h-change { font-size: 10px; }
.history-item .h-old { color: #aa6666; text-decoration: line-through; }
.history-item .h-new { color: #66aa66; }
.history-item .h-time { font-size: 9px; color: var(--text-muted); }
.history-item .h-revert {
  font-size: 10px; color: var(--accent); cursor: pointer;
  background: none; border: 1px solid var(--border); border-radius: 4px;
  padding: 2px 8px; align-self: flex-end; font-family: 'JetBrains Mono', monospace;
}
.history-item .h-revert:hover { background: var(--surface2); }
```

**Step 2: Adicionar HTML do painel**

Dentro de `.main`, apos o `</div>` do `.content`, adicionar:

```html
<div class="history-panel" id="history-panel">
  <div class="history-header">
    <span>Historico</span>
    <button class="btn btn-ghost" style="padding:4px 8px;font-size:11px;" id="btn-close-history">✕</button>
  </div>
  <div class="history-list" id="history-list">
    <div style="padding:16px 10px;font-size:12px;color:var(--text-muted);text-align:center;">
      Nenhuma alteracao ainda
    </div>
  </div>
</div>
```

**Step 3: Adicionar botao de toggle no header**

No header-actions, adicionar:

```html
<button class="btn btn-ghost" id="btn-history" disabled>📋 Historico</button>
```

**Step 4: Adicionar variavel de historico no STATE**

```javascript
let changeHistory = []; // Array de { table, col, rowIndex, oldValue, newValue, timestamp }
```

**Step 5: Implementar logica**

```javascript
document.getElementById('btn-history').addEventListener('click', toggleHistory);
document.getElementById('btn-close-history').addEventListener('click', () => {
  document.getElementById('history-panel').classList.remove('visible');
});

function toggleHistory() {
  document.getElementById('history-panel').classList.toggle('visible');
}

function addToHistory(table, col, rowIndex, oldValue, newValue) {
  changeHistory.unshift({
    table, col, rowIndex, oldValue, newValue,
    timestamp: new Date().toLocaleTimeString('pt-BR')
  });
  renderHistory();
}

function renderHistory() {
  const list = document.getElementById('history-list');
  if (!changeHistory.length) {
    list.innerHTML = '<div style="padding:16px 10px;font-size:12px;color:var(--text-muted);text-align:center;">Nenhuma alteracao ainda</div>';
    return;
  }
  list.innerHTML = changeHistory.map((h, i) => `
    <div class="history-item">
      <span class="h-table">${h.table}.${h.col} [linha ${h.rowIndex}]</span>
      <span class="h-change"><span class="h-old">${h.oldValue ?? '(vazio)'}</span> → <span class="h-new">${h.newValue ?? '(vazio)'}</span></span>
      <span class="h-time">${h.timestamp}</span>
      <button class="h-revert" onclick="revertHistoryItem(${i})">↩ Reverter</button>
    </div>
  `).join('');
}

function revertHistoryItem(index) {
  const h = changeHistory[index];
  if (!h || currentTable !== h.table) {
    toast('Mude para a tabela ' + h.table + ' primeiro', 'info');
    return;
  }
  const toUpdate = [];
  gridApi.forEachNode(node => {
    if (node.data.__rowIndex === h.rowIndex) {
      node.data[h.col] = h.oldValue;
      toUpdate.push(node.data);
      if (pendingChanges.has(h.rowIndex)) {
        pendingChanges.get(h.rowIndex)[h.col] = h.oldValue;
      }
    }
  });
  if (toUpdate.length) {
    gridApi.applyTransaction({ update: toUpdate });
    changeHistory.splice(index, 1);
    renderHistory();
    updateChangesPill();
    toast('Alteracao revertida', 'success');
  }
}
```

**Step 6: Integrar com onCellChanged**

Na funcao `onCellChanged`, apos o tracking de pendingChanges, adicionar:

```javascript
addToHistory(currentTable, params.colDef.field, idx, params.oldValue, params.newValue);
```

**Step 7: Habilitar botao ao carregar**

Adicionar `'btn-history'` ao array de botoes habilitados em `loadTable`.

**Step 8: Testar no browser**

1. Abrir arquivo, editar celulas
2. Abrir painel de historico — deve mostrar alteracoes
3. Clicar reverter — celula deve voltar ao valor original
4. Historico deve atualizar

**Step 9: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add change history panel with individual revert"
```

---

### Task 6: Confirmacao antes de salvar (Funcionalidade 9)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar HTML do modal de revisao**

Apos o modal de Find & Replace:

```html
<div class="modal-overlay hidden" id="modal-review">
  <div class="modal" style="width:600px;max-height:80vh;display:flex;flex-direction:column;">
    <h3>Revisar Alteracoes</h3>
    <div id="review-list" style="flex:1;overflow-y:auto;margin-bottom:16px;"></div>
    <div class="modal-actions">
      <button class="btn btn-ghost" id="review-cancel">Cancelar</button>
      <button class="btn btn-primary" id="review-confirm">Confirmar e Salvar</button>
    </div>
  </div>
</div>
```

**Step 2: Modificar funcao saveFile**

Renomear `saveFile` atual para `executeSave`. Criar nova `saveFile`:

```javascript
function saveFile() {
  if (!db || !currentTable || !gridApi) return;
  if (pendingChanges.size === 0) { toast('Nenhuma alteracao pendente', 'info'); return; }

  const list = document.getElementById('review-list');
  let html = '<div style="font-size:11px;font-family:\'JetBrains Mono\',monospace;">';
  pendingChanges.forEach((changes, rowIdx) => {
    Object.entries(changes).forEach(([col, newVal]) => {
      const origKey = `${rowIdx}:${col}`;
      const oldVal = originalValues.get(origKey) ?? '?';
      html += `<div style="padding:6px 10px;border-bottom:1px solid var(--border);display:flex;gap:8px;align-items:center;">
        <input type="checkbox" checked data-row="${rowIdx}" data-col="${col}" style="accent-color:var(--accent);">
        <span style="color:var(--text-muted);min-width:60px;">Linha ${rowIdx}</span>
        <span style="color:var(--accent);min-width:100px;">${col}</span>
        <span style="color:#aa6666;text-decoration:line-through;">${oldVal ?? '(vazio)'}</span>
        <span style="color:var(--text-muted);">→</span>
        <span style="color:#66aa66;">${newVal ?? '(vazio)'}</span>
      </div>`;
    });
  });
  html += '</div>';
  list.innerHTML = html;
  document.getElementById('modal-review').classList.remove('hidden');
}
```

**Step 3: Implementar listeners do modal**

```javascript
document.getElementById('review-cancel').addEventListener('click', () => {
  document.getElementById('modal-review').classList.add('hidden');
});

document.getElementById('review-confirm').addEventListener('click', () => {
  // Remove unchecked changes from pendingChanges
  const unchecked = document.querySelectorAll('#review-list input[type="checkbox"]:not(:checked)');
  unchecked.forEach(cb => {
    const row = parseInt(cb.dataset.row);
    const col = cb.dataset.col;
    if (pendingChanges.has(row)) {
      delete pendingChanges.get(row)[col];
      if (Object.keys(pendingChanges.get(row)).length === 0) pendingChanges.delete(row);
      // Revert the unchecked change in grid
      const origKey = `${row}:${col}`;
      if (originalValues.has(origKey)) {
        gridApi.forEachNode(node => {
          if (node.data.__rowIndex === row) {
            node.data[col] = originalValues.get(origKey);
            gridApi.applyTransaction({ update: [node.data] });
          }
        });
        originalValues.delete(origKey);
      }
    }
  });
  updateChangesPill();
  document.getElementById('modal-review').classList.add('hidden');
  executeSave();
});
```

**Step 4: Testar no browser**

1. Abrir arquivo, editar varias celulas
2. Clicar Salvar — modal de revisao deve aparecer
3. Desmarcar uma alteracao, confirmar — alteracao desmarcada deve ser revertida
4. Arquivo deve ser baixado com apenas alteracoes confirmadas

**Step 5: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add save review modal with selective discard"
```

---

### Task 7: Undo/Redo ilimitado com timeline (Funcionalidade 8)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: O undo/redo ja existe — expandir para ilimitado**

Remover qualquer limite de tamanho no undoStack/redoStack (atualmente ja ilimitado, apenas verificar).

**Step 2: Adicionar indicador visual na toolbar**

Na toolbar, apos o changes-pill, adicionar:

```html
<span class="changes-pill" id="undo-pill" style="display:none;">
  ↩ <span id="undo-count">0</span> | ↪ <span id="redo-count">0</span>
</span>
```

**Step 3: Implementar atualizacao do indicador**

```javascript
function updateUndoPill() {
  const pill = document.getElementById('undo-pill');
  const hasItems = undoStack.length > 0 || redoStack.length > 0;
  pill.style.display = hasItems ? 'flex' : 'none';
  document.getElementById('undo-count').textContent = undoStack.length;
  document.getElementById('redo-count').textContent = redoStack.length;
}
```

Chamar `updateUndoPill()` ao final de: `undoLast`, `redoLast`, `onCellChanged`, `loadTable`.

**Step 4: Testar no browser**

1. Editar celulas — pill deve mostrar contadores
2. Ctrl+Z — undo count diminui, redo count aumenta
3. Ctrl+Y — inverso

**Step 5: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add undo/redo counter indicator in toolbar"
```

---

### Task 8: Paginacao virtual (Funcionalidade 4)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Ativar row model infinito do AG Grid**

Substituir a estrategia de carregamento na funcao `loadTable`. Ao inves de `SELECT * LIMIT 5000`, usar o **Server-Side Row Model** do AG Grid (ou implementar paginacao manual).

Como estamos usando AG Grid Community (que nao tem server-side model), implementar **paginacao client-side** com carregamento em blocos:

Na funcao `loadTable`, substituir o `LIMIT 5000` por carregamento total com paginacao AG Grid:

```javascript
// Remover o LIMIT 5000
const result = db.exec(`SELECT * FROM "${tableName}"`);
```

E no `renderGrid`, adicionar opcoes de paginacao:

```javascript
pagination: true,
paginationPageSize: 500,
paginationPageSizeSelector: [100, 250, 500, 1000, 5000],
```

**Step 2: Testar no browser**

1. Abrir banco com tabela grande
2. Deve mostrar paginacao na parte inferior do grid
3. Navegar entre paginas deve ser fluido

**Step 3: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add pagination replacing LIMIT 5000"
```

---

### Task 9: Filtro avancado por coluna (Funcionalidade 1)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Ativar filtros AG Grid nativos**

AG Grid Community ja suporta filtros por coluna. Na funcao `loadTable`, os campos ja tem `filter: true`. Basta configurar o tipo de filtro adequado por tipo de dado.

Modificar a geracao de `columnDefs` em `loadTable`:

```javascript
const colInfo = db.exec(`PRAGMA table_info("${tableName}")`)[0]?.values ?? [];

...cols.map(col => {
  const info = colInfo.find(r => r[1] === col);
  const type = (info?.[2] ?? '').toUpperCase();
  const isNumeric = type.includes('INT') || type.includes('REAL') || type.includes('NUM') || type.includes('FLOAT');
  return {
    field: col, headerName: col, editable: !readOnlyMode, resizable: true, sortable: true,
    filter: isNumeric ? 'agNumberColumnFilter' : 'agTextColumnFilter',
    filterParams: isNumeric ? {} : { filterOptions: ['contains','notContains','equals','notEqual','startsWith','endsWith','blank','notBlank'] },
    minWidth: 100, flex: 1
  };
})
```

**Step 2: Adicionar CSS para o filter popup**

```css
.ag-theme-alpine-dark .ag-filter { background: var(--surface) !important; border: 1px solid var(--border) !important; }
```

**Step 3: Testar no browser**

1. Abrir arquivo, clicar no menu do header de uma coluna
2. Deve aparecer opcoes de filtro (contem, igual, comeca com, etc.)
3. Filtrar e verificar que as linhas sao filtradas corretamente

**Step 4: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add advanced column filters with type detection"
```

---

### Task 10: Console SQL (Funcionalidade 2)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar CSS do console**

```css
.sql-console {
  display: none; flex-direction: column; border-top: 1px solid var(--border);
  background: var(--surface); flex-shrink: 0; max-height: 40vh;
}
.sql-console.visible { display: flex; }
.sql-console-header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 8px 16px; border-bottom: 1px solid var(--border);
  font-size: 10px; font-weight: 700; letter-spacing: 0.12em;
  text-transform: uppercase; color: var(--text-muted);
}
.sql-editor {
  width: 100%; min-height: 60px; max-height: 120px; resize: vertical;
  background: var(--bg); border: none; border-bottom: 1px solid var(--border);
  color: var(--text); font-family: 'JetBrains Mono', monospace; font-size: 12px;
  padding: 12px 16px; outline: none;
}
.sql-result {
  flex: 1; overflow: auto; padding: 8px 16px; font-family: 'JetBrains Mono', monospace;
  font-size: 11px; color: var(--text-muted);
}
.sql-result table { border-collapse: collapse; width: 100%; }
.sql-result th { text-align: left; padding: 4px 8px; border-bottom: 1px solid var(--border); color: var(--accent); font-size: 10px; text-transform: uppercase; }
.sql-result td { padding: 4px 8px; border-bottom: 1px solid rgba(51,51,51,0.5); }
.sql-result .sql-error { color: #cc6666; }
```

**Step 2: Adicionar HTML do console**

Dentro de `.content`, antes do `</div>` final, apos o `.grid-wrapper`:

```html
<div class="sql-console" id="sql-console">
  <div class="sql-console-header">
    <span>Console SQL</span>
    <div style="display:flex;gap:8px;">
      <button class="btn btn-primary" style="padding:4px 12px;font-size:11px;" id="sql-run">▶ Executar (Ctrl+Enter)</button>
      <button class="btn btn-ghost" style="padding:4px 8px;font-size:11px;" id="sql-close">✕</button>
    </div>
  </div>
  <textarea class="sql-editor" id="sql-editor" placeholder="SELECT * FROM tabela WHERE ..."></textarea>
  <div class="sql-result" id="sql-result">Resultado aparecera aqui</div>
</div>
```

**Step 3: Adicionar botao toggle no header**

```html
<button class="btn btn-ghost" id="btn-sql" disabled>⌘ SQL</button>
```

**Step 4: Implementar logica**

```javascript
document.getElementById('btn-sql').addEventListener('click', toggleSqlConsole);
document.getElementById('sql-close').addEventListener('click', () => document.getElementById('sql-console').classList.remove('visible'));
document.getElementById('sql-run').addEventListener('click', executeSql);
document.getElementById('sql-editor').addEventListener('keydown', e => {
  if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') { e.preventDefault(); executeSql(); }
});

function toggleSqlConsole() {
  const console = document.getElementById('sql-console');
  console.classList.toggle('visible');
  if (console.classList.contains('visible')) {
    document.getElementById('sql-editor').focus();
  }
}

function executeSql() {
  const query = document.getElementById('sql-editor').value.trim();
  if (!query || !db) return;
  const resultDiv = document.getElementById('sql-result');
  try {
    const results = db.exec(query);
    if (!results.length) {
      resultDiv.innerHTML = '<span style="color:#66aa66;">Query executada com sucesso (0 resultados)</span>';
      // If it was a write query, refresh
      if (/^\s*(INSERT|UPDATE|DELETE|ALTER|DROP|CREATE)/i.test(query)) {
        loadTables();
        if (currentTable) loadTable(currentTable);
      }
      return;
    }
    const r = results[0];
    let html = `<div style="margin-bottom:8px;color:var(--accent);">${r.values.length} resultado(s)</div>`;
    html += '<table><thead><tr>' + r.columns.map(c => `<th>${c}</th>`).join('') + '</tr></thead><tbody>';
    r.values.forEach(row => {
      html += '<tr>' + row.map(v => `<td>${v ?? '<em style="opacity:.4">NULL</em>'}</td>`).join('') + '</tr>';
    });
    html += '</tbody></table>';
    resultDiv.innerHTML = html;
  } catch(e) {
    resultDiv.innerHTML = `<span class="sql-error">Erro: ${e.message}</span>`;
  }
}
```

**Step 5: Habilitar botao ao carregar**

Adicionar `'btn-sql'` ao array em `loadTable`.

**Step 6: Testar no browser**

1. Abrir arquivo, clicar em SQL
2. Digitar `SELECT * FROM tabela WHERE coluna = 'valor'`
3. Ctrl+Enter — resultado deve aparecer como tabela
4. Testar query invalida — deve mostrar erro em vermelho
5. Testar INSERT/UPDATE — deve recarregar tabela

**Step 7: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add SQL console with query execution"
```

---

### Task 11: Busca global cross-tabela (Funcionalidade 3)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar modal de busca global**

```html
<div class="modal-overlay hidden" id="modal-search">
  <div class="modal" style="width:700px;max-height:80vh;display:flex;flex-direction:column;">
    <h3>Busca Global</h3>
    <div class="form-group">
      <input type="text" id="global-search-input" placeholder="Digite o valor para buscar em todas as tabelas..." />
    </div>
    <div id="global-search-results" style="flex:1;overflow-y:auto;font-family:'JetBrains Mono',monospace;font-size:11px;"></div>
    <div class="modal-actions">
      <button class="btn btn-ghost" id="global-search-close">Fechar</button>
    </div>
  </div>
</div>
```

**Step 2: Adicionar botao no header**

```html
<button class="btn btn-ghost" id="btn-global-search" disabled>🔎 Busca Global</button>
```

**Step 3: Implementar logica**

```javascript
document.getElementById('btn-global-search').addEventListener('click', openGlobalSearch);
document.getElementById('global-search-close').addEventListener('click', () => document.getElementById('modal-search').classList.add('hidden'));
document.getElementById('global-search-input').addEventListener('keydown', e => {
  if (e.key === 'Enter') runGlobalSearch();
});

function openGlobalSearch() {
  document.getElementById('modal-search').classList.remove('hidden');
  document.getElementById('global-search-input').value = '';
  document.getElementById('global-search-results').innerHTML = '';
  document.getElementById('global-search-input').focus();
}

function runGlobalSearch() {
  if (!db) return;
  const term = document.getElementById('global-search-input').value.trim();
  if (!term) return;
  const resultsDiv = document.getElementById('global-search-results');
  resultsDiv.innerHTML = '<div style="color:var(--text-muted);padding:8px;">Buscando...</div>';

  const tables = db.exec("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name");
  if (!tables.length) { resultsDiv.innerHTML = 'Nenhuma tabela'; return; }

  let html = '';
  let totalFound = 0;

  tables[0].values.forEach(([tableName]) => {
    const colsResult = db.exec(`PRAGMA table_info("${tableName}")`);
    if (!colsResult.length) return;
    const cols = colsResult[0].values.map(r => r[1]);
    const whereClauses = cols.map(c => `CAST("${c}" AS TEXT) LIKE ?`).join(' OR ');
    const params = cols.map(() => `%${term}%`);
    try {
      const found = db.exec(`SELECT *, rowid as __rid FROM "${tableName}" WHERE ${whereClauses} LIMIT 50`, params);
      if (found.length && found[0].values.length) {
        const count = found[0].values.length;
        totalFound += count;
        html += `<div style="margin:12px 0 6px;font-weight:700;color:var(--accent);">${tableName} (${count} resultado${count>1?'s':''})</div>`;
        // Show which columns matched
        found[0].values.forEach(row => {
          const matchCols = [];
          cols.forEach((col, i) => {
            if (String(row[i] ?? '').toLowerCase().includes(term.toLowerCase())) {
              matchCols.push(`<span style="color:var(--text);">${col}</span>=<span style="color:#66aa66;">${row[i]}</span>`);
            }
          });
          html += `<div style="padding:4px 10px;border-left:2px solid var(--border);margin:2px 0;cursor:pointer;" onclick="jumpToTable('${tableName}')">${matchCols.join(' | ')}</div>`;
        });
      }
    } catch(e) { /* skip problematic tables */ }
  });

  if (!totalFound) html = '<div style="color:var(--text-muted);padding:16px;text-align:center;">Nenhum resultado encontrado</div>';
  else html = `<div style="margin-bottom:8px;color:var(--text-muted);">${totalFound} resultado(s) encontrado(s)</div>` + html;
  resultsDiv.innerHTML = html;
}

function jumpToTable(tableName) {
  document.getElementById('modal-search').classList.add('hidden');
  loadTable(tableName);
}
```

**Step 4: Habilitar botao ao carregar**

Adicionar `'btn-global-search'` ao array em `loadTable`.

**Step 5: Testar no browser**

1. Abrir arquivo, clicar Busca Global
2. Digitar termo que existe em varias tabelas
3. Deve listar resultados agrupados por tabela
4. Clicar em resultado deve navegar para a tabela

**Step 6: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add cross-table global search"
```

---

### Task 12: Favoritos/Marcadores (Funcionalidade 5)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar STATE**

```javascript
let bookmarks = []; // Array de { table, rowIndex, label, cols }
```

**Step 2: Adicionar botao de favoritar no header**

```html
<button class="btn btn-ghost" id="btn-bookmark" disabled>★ Favoritar</button>
<button class="btn btn-ghost" id="btn-bookmarks" disabled>☆ Favoritos</button>
```

**Step 3: Implementar logica de bookmarks**

```javascript
document.getElementById('btn-bookmark').addEventListener('click', bookmarkSelected);
document.getElementById('btn-bookmarks').addEventListener('click', showBookmarks);

function bookmarkSelected() {
  if (!gridApi || !currentTable) return;
  const selected = gridApi.getSelectedRows();
  if (!selected.length) { toast('Selecione linhas para favoritar', 'info'); return; }
  selected.forEach(row => {
    const exists = bookmarks.find(b => b.table === currentTable && b.rowIndex === row.__rowIndex);
    if (!exists) {
      const cols = columnDefs.slice(1, 4).map(c => c.field);
      const preview = cols.map(c => `${c}=${row[c] ?? ''}`).join(', ');
      bookmarks.push({ table: currentTable, rowIndex: row.__rowIndex, label: preview, data: row });
    }
  });
  toast(`${selected.length} linha(s) favoritada(s)`, 'success');
}

function showBookmarks() {
  if (!bookmarks.length) { toast('Nenhum favorito salvo', 'info'); return; }
  // Reusar modal de busca global
  const modal = document.getElementById('modal-search');
  modal.querySelector('h3').textContent = 'Favoritos';
  const resultsDiv = document.getElementById('global-search-results');
  const input = document.getElementById('global-search-input');
  input.style.display = 'none';

  let html = '';
  bookmarks.forEach((b, i) => {
    html += `<div style="padding:8px 10px;border:1px solid var(--border);border-radius:7px;margin:4px 0;display:flex;justify-content:space-between;align-items:center;cursor:pointer;" onclick="jumpToBookmark(${i})">
      <div><span style="color:var(--accent);font-weight:600;">${b.table}</span> <span style="color:var(--text-muted);font-size:10px;">linha ${b.rowIndex}</span><br><span style="font-size:10px;color:var(--text-muted);">${b.label}</span></div>
      <button class="btn btn-ghost" style="padding:2px 8px;font-size:10px;" onclick="event.stopPropagation();removeBookmark(${i})">✕</button>
    </div>`;
  });
  resultsDiv.innerHTML = html;
  modal.classList.remove('hidden');

  // Restore on close
  document.getElementById('global-search-close').onclick = () => {
    modal.classList.add('hidden');
    modal.querySelector('h3').textContent = 'Busca Global';
    input.style.display = '';
    document.getElementById('global-search-close').onclick = () => modal.classList.add('hidden');
  };
}

function jumpToBookmark(index) {
  const b = bookmarks[index];
  document.getElementById('modal-search').classList.add('hidden');
  document.getElementById('modal-search').querySelector('h3').textContent = 'Busca Global';
  document.getElementById('global-search-input').style.display = '';
  if (currentTable !== b.table) loadTable(b.table);
  // Scroll to row
  if (gridApi) {
    gridApi.forEachNode(node => {
      if (node.data.__rowIndex === b.rowIndex) {
        gridApi.ensureNodeVisible(node, 'middle');
        node.setSelected(true);
      }
    });
  }
}

function removeBookmark(index) {
  bookmarks.splice(index, 1);
  showBookmarks();
}
```

**Step 4: Habilitar botoes ao carregar**

Adicionar `'btn-bookmark'` e `'btn-bookmarks'` ao array em `loadTable`.

**Step 5: Testar no browser**

1. Selecionar linhas, clicar Favoritar
2. Clicar Favoritos — deve listar as linhas favoritadas
3. Clicar em um favorito — deve navegar para a linha
4. Remover favorito — deve sumir da lista

**Step 6: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add row bookmarks for quick navigation"
```

---

### Task 13: Edicao em lote por selecao (Funcionalidade 12)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Detectar selecao de coluna unica**

Adicionar funcao que verifica se a selecao atual e de uma unica coluna:

```javascript
function getSelectionSingleColumn() {
  const bounds = getSelBounds();
  if (!bounds || bounds.minCol !== bounds.maxCol) return null;
  return { colIndex: bounds.minCol, minRow: bounds.minRow, maxRow: bounds.maxRow };
}
```

**Step 2: Adicionar menu de contexto simples**

Ao clicar com botao direito em celula selecionada, mostrar opcao de editar em lote:

```javascript
document.getElementById('grid').addEventListener('contextmenu', e => {
  e.preventDefault();
  const sel = getSelectionSingleColumn();
  if (!sel || sel.minRow === sel.maxRow) return;
  const cols = getDataCols();
  const colName = cols[sel.colIndex];
  const count = sel.maxRow - sel.minRow + 1;
  const newValue = prompt(`Novo valor para ${count} celulas na coluna "${colName}":`);
  if (newValue === null) return;
  applyBulkEdit(sel, colName, newValue);
});

function applyBulkEdit(sel, colName, newValue) {
  const batch = [], toUpdate = [];
  gridApi.forEachNode(node => {
    if (node.rowIndex >= sel.minRow && node.rowIndex <= sel.maxRow) {
      const oldValue = node.data[colName];
      batch.push({ rowIndex: node.rowIndex, colField: colName, oldValue, newValue, dataIndex: node.data.__rowIndex });
      node.data[colName] = newValue;
      toUpdate.push(node.data);
      if (node.data.__rowIndex !== undefined) {
        if (!pendingChanges.has(node.data.__rowIndex)) pendingChanges.set(node.data.__rowIndex, {});
        pendingChanges.get(node.data.__rowIndex)[colName] = newValue;
      }
      addToHistory(currentTable, colName, node.data.__rowIndex, oldValue, newValue);
    }
  });
  if (batch.length) {
    undoStack.push({ batch }); redoStack = [];
    gridApi.applyTransaction({ update: toUpdate });
    updateChangesPill();
    updateUndoPill();
    toast(`${batch.length} celula(s) editada(s) em lote`, 'success');
  }
}
```

**Step 3: Testar no browser**

1. Selecionar varias celulas de uma coluna (clicar e arrastar)
2. Botao direito — prompt deve aparecer
3. Digitar valor — todas as celulas devem atualizar

**Step 4: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add bulk edit via right-click on column selection"
```

---

### Task 14: Exportar para Excel/CSV (Funcionalidade 14)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar botao no header**

```html
<button class="btn btn-ghost" id="btn-export" disabled>📤 Exportar CSV</button>
```

**Step 2: Implementar exportacao**

```javascript
document.getElementById('btn-export').addEventListener('click', exportCsv);

function exportCsv() {
  if (!gridApi || !currentTable) return;
  const cols = columnDefs.slice(1).map(c => c.field);
  let csv = cols.join(';') + '\n';
  gridApi.forEachNodeAfterFilterAndSort(node => {
    csv += cols.map(c => {
      const val = String(node.data[c] ?? '');
      return val.includes(';') || val.includes('"') || val.includes('\n')
        ? '"' + val.replace(/"/g, '""') + '"'
        : val;
    }).join(';') + '\n';
  });
  const blob = new Blob(['\uFEFF' + csv], { type: 'text/csv;charset=utf-8;' });
  const a = Object.assign(document.createElement('a'), {
    href: URL.createObjectURL(blob),
    download: `${currentTable}.csv`
  });
  a.click();
  URL.revokeObjectURL(a.href);
  toast(`Exportado: ${currentTable}.csv`, 'success');
}
```

**Step 3: Habilitar botao ao carregar**

Adicionar `'btn-export'` ao array em `loadTable`.

**Step 4: Testar no browser**

1. Abrir arquivo, selecionar tabela
2. Clicar Exportar CSV — arquivo .csv deve ser baixado
3. Abrir no Excel — dados devem estar corretos com separador ;

**Step 5: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add CSV export with BOM for Excel compatibility"
```

---

### Task 15: Importar dados de Excel/CSV (Funcionalidade 13)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar botao e input file**

```html
<label for="csv-input" class="btn btn-ghost" id="btn-import-label" style="pointer-events:none;opacity:0.35;">📥 Importar CSV</label>
<input type="file" id="csv-input" accept=".csv,.tsv,.txt" style="display:none;" />
```

**Step 2: Implementar importacao**

```javascript
document.getElementById('csv-input').addEventListener('change', e => {
  const file = e.target.files[0];
  if (file) importCsv(file);
  e.target.value = '';
});

function importCsv(file) {
  if (!db || !currentTable) return;
  const reader = new FileReader();
  reader.onload = e => {
    try {
      const text = e.target.result;
      const sep = text.includes('\t') ? '\t' : ';';
      const lines = text.replace(/\r\n/g, '\n').split('\n').filter(l => l.trim());
      if (lines.length < 2) { toast('Arquivo vazio ou sem dados', 'error'); return; }

      const headers = lines[0].split(sep).map(h => h.replace(/^"|"$/g, '').trim());
      const tableCols = columnDefs.slice(1).map(c => c.field);

      // Map CSV columns to table columns
      const mapping = headers.map(h => tableCols.find(c => c.toLowerCase() === h.toLowerCase()));
      const validMapping = mapping.filter(m => m !== undefined);
      if (!validMapping.length) { toast('Nenhuma coluna correspondente encontrada', 'error'); return; }

      const colInfo = db.exec(`PRAGMA table_info("${currentTable}")`)[0]?.values ?? [];
      let imported = 0;
      db.run('BEGIN TRANSACTION');
      try {
        for (let i = 1; i < lines.length; i++) {
          const vals = lines[i].split(sep).map(v => v.replace(/^"|"$/g, ''));
          const insertCols = [], insertVals = [];
          headers.forEach((h, j) => {
            if (mapping[j] && vals[j] !== undefined) {
              const info = colInfo.find(r => r[1] === mapping[j]);
              const isPk = info?.[5] > 0;
              if (!isPk) {
                insertCols.push(`"${mapping[j]}"`);
                insertVals.push(vals[j] === '' ? null : vals[j]);
              }
            }
          });
          if (insertCols.length) {
            db.run(`INSERT INTO "${currentTable}" (${insertCols.join(',')}) VALUES (${insertCols.map(()=>'?').join(',')})`, insertVals);
            imported++;
          }
        }
        db.run('COMMIT');
      } catch(e) { db.run('ROLLBACK'); throw e; }

      loadTable(currentTable);
      toast(`${imported} linha(s) importada(s) do CSV`, 'success');
    } catch(e) { toast('Erro ao importar: ' + e.message, 'error'); }
  };
  reader.readAsText(file, 'UTF-8');
}
```

**Step 3: Habilitar label ao carregar**

Na funcao `loadTable`, apos habilitar os botoes:

```javascript
const importLabel = document.getElementById('btn-import-label');
importLabel.style.pointerEvents = '';
importLabel.style.opacity = '';
```

**Step 4: Testar no browser**

1. Exportar tabela como CSV (task anterior)
2. Adicionar linhas no CSV manualmente
3. Importar o CSV — novas linhas devem aparecer

**Step 5: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add CSV import with automatic column mapping"
```

---

### Task 16: Edicao via formula (Funcionalidade 17)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar modal de formula**

```html
<div class="modal-overlay hidden" id="modal-formula">
  <div class="modal">
    <h3>Transformar Coluna</h3>
    <div class="form-group">
      <label>Coluna</label>
      <select id="formula-col"></select>
    </div>
    <div class="form-group">
      <label>Transformacao</label>
      <select id="formula-type">
        <option value="upper">MAIUSCULAS (UPPER)</option>
        <option value="lower">minusculas (LOWER)</option>
        <option value="trim">Remover espacos (TRIM)</option>
        <option value="replace">Substituir texto</option>
        <option value="prefix">Adicionar prefixo</option>
        <option value="suffix">Adicionar sufixo</option>
        <option value="math-add">Somar valor</option>
        <option value="math-mul">Multiplicar por</option>
      </select>
    </div>
    <div class="form-group" id="formula-param-group" style="display:none;">
      <label id="formula-param-label">Valor</label>
      <input type="text" id="formula-param" />
    </div>
    <div class="form-group" id="formula-param2-group" style="display:none;">
      <label>Substituir por</label>
      <input type="text" id="formula-param2" />
    </div>
    <div class="modal-actions">
      <button class="btn btn-ghost" id="formula-cancel">Cancelar</button>
      <button class="btn btn-primary" id="formula-apply">Aplicar</button>
    </div>
  </div>
</div>
```

**Step 2: Adicionar botao no header**

```html
<button class="btn btn-ghost" id="btn-formula" disabled>ƒ Formula</button>
```

**Step 3: Implementar logica**

```javascript
document.getElementById('btn-formula').addEventListener('click', openFormula);
document.getElementById('formula-cancel').addEventListener('click', () => document.getElementById('modal-formula').classList.add('hidden'));
document.getElementById('formula-apply').addEventListener('click', applyFormula);
document.getElementById('formula-type').addEventListener('change', updateFormulaParams);

function openFormula() {
  if (!currentTable) return;
  const cols = columnDefs.slice(1).map(c => c.field);
  document.getElementById('formula-col').innerHTML = cols.map(c => `<option value="${c}">${c}</option>`).join('');
  document.getElementById('formula-param').value = '';
  document.getElementById('formula-param2').value = '';
  updateFormulaParams();
  document.getElementById('modal-formula').classList.remove('hidden');
}

function updateFormulaParams() {
  const type = document.getElementById('formula-type').value;
  const p1 = document.getElementById('formula-param-group');
  const p2 = document.getElementById('formula-param2-group');
  const label = document.getElementById('formula-param-label');
  p1.style.display = 'none'; p2.style.display = 'none';
  if (type === 'replace') { p1.style.display = ''; p2.style.display = ''; label.textContent = 'Buscar'; }
  else if (type === 'prefix' || type === 'suffix') { p1.style.display = ''; label.textContent = 'Texto'; }
  else if (type.startsWith('math-')) { p1.style.display = ''; label.textContent = 'Valor numerico'; }
}

function applyFormula() {
  const col = document.getElementById('formula-col').value;
  const type = document.getElementById('formula-type').value;
  const param = document.getElementById('formula-param').value;
  const param2 = document.getElementById('formula-param2').value;

  const batch = [], toUpdate = [];
  gridApi.forEachNode(node => {
    const oldValue = node.data[col];
    let newValue = oldValue;
    const str = String(oldValue ?? '');
    switch(type) {
      case 'upper': newValue = str.toUpperCase(); break;
      case 'lower': newValue = str.toLowerCase(); break;
      case 'trim': newValue = str.trim(); break;
      case 'replace': newValue = str.split(param).join(param2); break;
      case 'prefix': newValue = param + str; break;
      case 'suffix': newValue = str + param; break;
      case 'math-add': newValue = (parseFloat(oldValue) || 0) + (parseFloat(param) || 0); break;
      case 'math-mul': newValue = (parseFloat(oldValue) || 0) * (parseFloat(param) || 1); break;
    }
    if (newValue !== oldValue) {
      batch.push({ rowIndex: node.rowIndex, colField: col, oldValue, newValue, dataIndex: node.data.__rowIndex });
      node.data[col] = newValue;
      toUpdate.push(node.data);
      if (node.data.__rowIndex !== undefined) {
        if (!pendingChanges.has(node.data.__rowIndex)) pendingChanges.set(node.data.__rowIndex, {});
        pendingChanges.get(node.data.__rowIndex)[col] = newValue;
      }
      addToHistory(currentTable, col, node.data.__rowIndex, oldValue, newValue);
    }
  });

  if (batch.length) {
    undoStack.push({ batch }); redoStack = [];
    gridApi.applyTransaction({ update: toUpdate });
    updateChangesPill();
    updateUndoPill();
  }
  document.getElementById('modal-formula').classList.add('hidden');
  toast(`Formula aplicada: ${batch.length} celula(s) alterada(s)`, batch.length > 0 ? 'success' : 'info');
}
```

**Step 4: Habilitar botao ao carregar**

Adicionar `'btn-formula'` ao array em `loadTable`.

**Step 5: Testar no browser**

1. Abrir arquivo, clicar Formula
2. Selecionar coluna, escolher UPPER, aplicar — deve converter para maiusculas
3. Testar TRIM, REPLACE, prefixo, operacoes matematicas
4. Ctrl+Z deve desfazer tudo

**Step 6: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add formula-based column transformations"
```

---

### Task 17: Templates de operacao (Funcionalidade 15)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Adicionar STATE**

```javascript
let operationTemplates = JSON.parse(localStorage.getItem('rad-apsdu-templates') || '[]');
```

**Step 2: Adicionar modal de templates**

```html
<div class="modal-overlay hidden" id="modal-templates">
  <div class="modal" style="width:550px;max-height:80vh;display:flex;flex-direction:column;">
    <h3>Templates de Operacao</h3>
    <div id="templates-list" style="flex:1;overflow-y:auto;margin-bottom:16px;"></div>
    <div class="modal-actions">
      <button class="btn btn-ghost" id="templates-close">Fechar</button>
    </div>
  </div>
</div>
```

**Step 3: Adicionar botao de salvar template no Find & Replace**

No modal Find & Replace, antes do botao "Aplicar":

```html
<button class="btn btn-ghost" id="fr-save-template">💾 Salvar Template</button>
```

**Step 4: Adicionar botao de templates no header**

```html
<button class="btn btn-ghost" id="btn-templates" disabled>📑 Templates</button>
```

**Step 5: Implementar logica**

```javascript
document.getElementById('btn-templates').addEventListener('click', showTemplates);
document.getElementById('templates-close').addEventListener('click', () => document.getElementById('modal-templates').classList.add('hidden'));
document.getElementById('fr-save-template').addEventListener('click', saveTemplate);

function saveTemplate() {
  const name = prompt('Nome do template:');
  if (!name) return;
  operationTemplates.push({
    name,
    filterCol: document.getElementById('fr-filter-col').value,
    filterVal: document.getElementById('fr-filter-val').value,
    replaceCol: document.getElementById('fr-column').value,
    replaceVal: document.getElementById('fr-replace').value,
  });
  localStorage.setItem('rad-apsdu-templates', JSON.stringify(operationTemplates));
  toast('Template salvo: ' + name, 'success');
}

function showTemplates() {
  const list = document.getElementById('templates-list');
  if (!operationTemplates.length) {
    list.innerHTML = '<div style="padding:16px;font-size:12px;color:var(--text-muted);text-align:center;">Nenhum template salvo.<br>Crie templates pelo Find & Replace.</div>';
  } else {
    list.innerHTML = operationTemplates.map((t, i) => `
      <div style="padding:10px;border:1px solid var(--border);border-radius:7px;margin:4px 0;font-family:'JetBrains Mono',monospace;font-size:11px;">
        <div style="font-weight:700;color:var(--accent);margin-bottom:4px;">${t.name}</div>
        <div style="color:var(--text-muted);">
          ${t.filterVal ? `Onde ${t.filterCol} = "${t.filterVal}" → ` : 'Todas as linhas → '}
          ${t.replaceCol} = "${t.replaceVal}"
        </div>
        <div style="margin-top:8px;display:flex;gap:6px;justify-content:flex-end;">
          <button class="btn btn-primary" style="padding:3px 10px;font-size:10px;" onclick="executeTemplate(${i})">▶ Executar</button>
          <button class="btn btn-danger" style="padding:3px 10px;font-size:10px;" onclick="deleteTemplate(${i})">✕</button>
        </div>
      </div>
    `).join('');
  }
  document.getElementById('modal-templates').classList.remove('hidden');
}

function executeTemplate(index) {
  const t = operationTemplates[index];
  document.getElementById('fr-filter-col').value = t.filterCol;
  document.getElementById('fr-filter-val').value = t.filterVal;
  document.getElementById('fr-column').value = t.replaceCol;
  document.getElementById('fr-replace').value = t.replaceVal;
  document.getElementById('modal-templates').classList.add('hidden');
  applyFindReplace();
}

function deleteTemplate(index) {
  operationTemplates.splice(index, 1);
  localStorage.setItem('rad-apsdu-templates', JSON.stringify(operationTemplates));
  showTemplates();
}
```

**Step 6: Habilitar botao ao carregar**

Adicionar `'btn-templates'` ao array em `loadTable`.

**Step 7: Testar no browser**

1. Abrir Find & Replace, definir operacao, clicar "Salvar Template"
2. Ir em Templates — deve listar o template
3. Clicar Executar — deve aplicar a operacao
4. Fechar e reabrir browser — template deve persistir (localStorage)

**Step 8: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add reusable operation templates with localStorage persistence"
```

---

### Task 18: Auto-incremento inteligente (Funcionalidade 16)

**Files:**
- Modify: `RAD APSDU 0.03.html`

**Step 1: Expandir modal de replicacao**

Modificar a funcao `replicateRows` para mostrar um modal com opcoes de incremento antes de replicar:

```html
<div class="modal-overlay hidden" id="modal-replicate">
  <div class="modal">
    <h3>Replicar Linhas</h3>
    <div class="form-group">
      <label>Quantidade de copias por linha</label>
      <input type="number" id="replicate-count" value="1" min="1" max="100" />
    </div>
    <div class="modal-section">
      <div class="modal-section-title">Regras de Incremento</div>
      <div id="increment-rules"></div>
      <button class="btn btn-ghost" style="margin-top:8px;font-size:11px;" id="add-increment-rule">+ Adicionar regra</button>
    </div>
    <div class="modal-actions">
      <button class="btn btn-ghost" id="replicate-cancel">Cancelar</button>
      <button class="btn btn-primary" id="replicate-confirm">Replicar</button>
    </div>
  </div>
</div>
```

**Step 2: Implementar logica**

```javascript
document.getElementById('replicate-cancel').addEventListener('click', () => document.getElementById('modal-replicate').classList.add('hidden'));
document.getElementById('replicate-confirm').addEventListener('click', executeReplication);
document.getElementById('add-increment-rule').addEventListener('click', addIncrementRule);

// Override the original replicateRows to show modal
function replicateRows() {
  if (!gridApi || !currentTable) return;
  const selected = gridApi.getSelectedRows();
  if (!selected.length) { toast('Selecione linhas para replicar', 'info'); return; }

  const cols = columnDefs.slice(1).map(c => c.field);
  document.getElementById('increment-rules').innerHTML = '';
  document.getElementById('replicate-count').value = '1';

  // Auto-add R_E_C_N_O_ rule if exists
  if (cols.includes('R_E_C_N_O_')) addIncrementRule('R_E_C_N_O_', 1);

  document.getElementById('modal-replicate').classList.remove('hidden');
}

function addIncrementRule(presetCol, presetStep) {
  const cols = columnDefs.slice(1).map(c => c.field);
  const div = document.createElement('div');
  div.className = 'modal-row';
  div.style.marginBottom = '6px';
  div.innerHTML = `
    <select class="inline-select rule-col">${cols.map(c => `<option value="${c}" ${c===presetCol?'selected':''}>${c}</option>`).join('')}</select>
    <span style="color:var(--text-muted);font-size:11px;flex-shrink:0;">+</span>
    <input type="number" class="inline-input rule-step" value="${presetStep||1}" style="width:80px;" />
    <button class="btn btn-ghost" style="padding:4px 8px;" onclick="this.parentElement.remove()">✕</button>
  `;
  document.getElementById('increment-rules').appendChild(div);
}

function executeReplication() {
  if (!gridApi || !currentTable) return;
  const selected = gridApi.getSelectedRows();
  const copyCount = parseInt(document.getElementById('replicate-count').value) || 1;

  // Parse rules
  const rules = [];
  document.querySelectorAll('#increment-rules .modal-row').forEach(row => {
    rules.push({
      col: row.querySelector('.rule-col').value,
      step: parseFloat(row.querySelector('.rule-step').value) || 1
    });
  });

  try {
    const cols = columnDefs.slice(1).map(c => c.field);
    const colInfo = db.exec(`PRAGMA table_info("${currentTable}")`)[0]?.values ?? [];

    // Get max values for increment cols
    const maxVals = {};
    rules.forEach(rule => {
      const res = db.exec(`SELECT MAX(CAST("${rule.col}" AS REAL)) FROM "${currentTable}"`);
      maxVals[rule.col] = res[0]?.values[0][0] ?? 0;
    });

    db.run('BEGIN TRANSACTION');
    try {
      selected.forEach(row => {
        for (let copy = 0; copy < copyCount; copy++) {
          const insertCols = [], insertVals = [];
          cols.forEach(col => {
            const info = colInfo.find(r => r[1] === col);
            const isPk = info?.[5] > 0;
            if (isPk) return;
            insertCols.push(`"${col}"`);
            const rule = rules.find(r => r.col === col);
            if (rule) {
              maxVals[col] += rule.step;
              insertVals.push(maxVals[col]);
            } else {
              insertVals.push(row[col] ?? null);
            }
          });
          db.run(
            `INSERT INTO "${currentTable}" (${insertCols.join(',')}) VALUES (${insertCols.map(()=>'?').join(',')})`,
            insertVals
          );
        }
      });
      db.run('COMMIT');
    } catch(e) { db.run('ROLLBACK'); throw e; }

    document.getElementById('modal-replicate').classList.add('hidden');
    loadTable(currentTable);
    toast(`${selected.length * copyCount} linha(s) replicada(s)`, 'success');
  } catch(e) { toast('Erro ao replicar: ' + e.message, 'error'); }
}
```

**Step 3: Testar no browser**

1. Selecionar linhas, clicar Replicar
2. Modal deve aparecer com R_E_C_N_O_ ja pre-configurado
3. Adicionar regra para outro campo, definir step
4. Definir 3 copias, confirmar — deve criar 3 copias por linha com incrementos

**Step 4: Commit**

```bash
git add "RAD APSDU 0.03.html"
git commit -m "feat: add smart replication with custom increment rules"
```

---

## Resumo da Ordem de Execucao

| Task | Funcionalidade | Eixo | Dependencia |
|------|---------------|------|-------------|
| 1 | Setup v0.03 | — | — |
| 2 | Backup automatico | Seguranca | Task 1 |
| 3 | Modo somente leitura | Seguranca | Task 1 |
| 4 | Diff visual | Seguranca | Task 1 |
| 5 | Historico de alteracoes | Seguranca | Task 4 (usa originalValues) |
| 6 | Confirmacao ao salvar | Seguranca | Task 4 (usa originalValues) |
| 7 | Undo/Redo indicator | Seguranca | Task 1 |
| 8 | Paginacao virtual | Busca | Task 1 |
| 9 | Filtro avancado | Busca | Task 1 |
| 10 | Console SQL | Busca | Task 1 |
| 11 | Busca global | Busca | Task 1 |
| 12 | Favoritos | Busca | Task 11 (reutiliza modal) |
| 13 | Edicao em lote | Produtividade | Task 5 (usa addToHistory) |
| 14 | Exportar CSV | Produtividade | Task 1 |
| 15 | Importar CSV | Produtividade | Task 1 |
| 16 | Edicao via formula | Produtividade | Task 5 (usa addToHistory) |
| 17 | Templates | Produtividade | Task 1 |
| 18 | Auto-incremento inteligente | Produtividade | Task 1 |
