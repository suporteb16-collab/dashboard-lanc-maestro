# Deploy Guide — Dashboard Maestro Lançamentos

## Arquitetura

```
Google Sheets (silver)
       ↓  (atualiza via Apps Script ou upload manual)
  BigQuery — tabela total_lancamentos_silver
       ↓
  Cloudflare Worker  (worker.js)
  · Autentica com Service Account
  · Roda queries SQL no BQ
  · Retorna JSON agregado
  · Cache 30 min no edge
       ↓
  Dashboard HTML (Dashboard_Lancamentos.html)
  · fetch(WORKER_URL + '/data')
  · Renderiza todos os gráficos
       ↓
  GitHub Pages ou Cloudflare Pages
  (hospeda o HTML estático — público)
```

---

## PASSO 1 — Credenciais BigQuery

### 1.1 Criar Service Account

1. Acesse [console.cloud.google.com](https://console.cloud.google.com)
2. IAM & Admin → Service Accounts → **+ Create Service Account**
   - Nome: `maestro-dashboard`
3. Atribua os papéis:
   - `BigQuery Data Viewer`
   - `BigQuery Job User`
4. Clique **Done**

### 1.2 Baixar chave JSON

1. Clique na Service Account criada
2. Aba **Keys** → **Add Key** → **Create new key** → **JSON**
3. Salve o arquivo — você vai precisar do conteúdo completo

### 1.3 Criar Dataset e Tabela no BigQuery

No [BigQuery Studio](https://console.cloud.google.com/bigquery):

```sql
-- Criar dataset
CREATE SCHEMA IF NOT EXISTS `SEU_PROJETO.maestro`
OPTIONS(location='US');

-- Criar tabela (schema inferido do CSV)
-- Use a UI: Dataset → Create Table → Upload CSV
-- Nome da tabela: total_lancamentos_silver
-- Habilite: "Auto-detect schema"
```

Ou via CLI:
```bash
bq load --autodetect \
  --source_format=CSV \
  SEU_PROJETO:maestro.total_lancamentos_silver \
  MAESTRO_LANCAMENTOS_TODOS_silver.csv
```

> **Dica:** Para atualização automática pelo Google Sheets, veja a seção "Automação" no final.

---

## PASSO 2 — Configurar o Cloudflare Worker

### 2.1 Abrir o Worker existente

Você já tem o Worker `polished-bar-17f8`. Acesse:
[dash.cloudflare.com → Workers → polished-bar-17f8](https://dash.cloudflare.com/a9609c3343e6f8a15f26c898c3df8d08/workers/services/view/polished-bar-17f8/production)

### 2.2 Adicionar Secrets (variáveis de ambiente)

Em **Settings → Variables**:

| Secret | Valor |
|--------|-------|
| `BQ_SA_JSON` | Cole o conteúdo **completo** do JSON da service account |
| `BQ_PROJECT_ID` | ID do seu projeto GCP (ex: `meu-projeto-123`) |
| `BQ_DATASET` | `maestro` |
| `BQ_TABLE` | `total_lancamentos_silver` |

> Todos como **Secret** (não texto simples) — clique em "Encrypt".

### 2.3 Deploy do código do Worker

Opção A — Via Editor online:
1. Worker → **Edit Code**
2. Apague todo o conteúdo
3. Cole o conteúdo de `worker.js`
4. Clique **Deploy**

Opção B — Via Wrangler CLI (recomendado para automação):
```bash
npm install -g wrangler
wrangler login
wrangler deploy worker.js --name polished-bar-17f8
```

### 2.4 Testar o Worker

Acesse no browser:
```
https://polished-bar-17f8.workers.dev/data
```

Você deve receber um JSON com as chaves: `cards`, `tags`, `produtos`, `horas`, etc.

Se aparecer erro de autenticação, verifique se o `BQ_SA_JSON` foi salvo corretamente (sem quebras de linha extras).

---

## PASSO 3 — Subir no GitHub e publicar via Pages

### 3.1 Criar repositório

1. Acesse [github.com/new](https://github.com/new)
2. Nome: `maestro-dashboard`
3. Visibilidade: **Public** (necessário para GitHub Pages gratuito) ou **Private** + Cloudflare Pages
4. Clique **Create repository**

### 3.2 Adicionar os arquivos

No browser (sem CLI), pela interface do GitHub:

1. Clique **uploading an existing file**
2. Arraste o arquivo `Dashboard_Lancamentos.html`
3. Commit: `"feat: add dashboard html"`

> Só o HTML vai pro repositório — o CSV/silver fica apenas no BigQuery. ✅

### 3.3 Ativar GitHub Pages

1. Repositório → **Settings → Pages**
2. Source: **Deploy from a branch**
3. Branch: `main` / Folder: `/ (root)`
4. Clique **Save**

URL pública será: `https://SEU_USUARIO.github.io/maestro-dashboard/Dashboard_Lancamentos.html`

### 3.4 Alternativa: Cloudflare Pages (recomendado — CDN global)

1. [dash.cloudflare.com](https://dash.cloudflare.com) → **Pages → Create a project**
2. **Connect to Git** → selecione o repositório `maestro-dashboard`
3. Framework preset: **None**
4. Build output directory: `/`
5. Clique **Save and Deploy**

URL será: `https://maestro-dashboard.pages.dev`

---

## Automação — Google Sheets → BigQuery

Para que o Sheets atualize o BQ automaticamente quando o silver mudar:

### Opção A: Apps Script no Sheets (sem código externo)

No Google Sheets com a base silver:

1. **Extensions → Apps Script**
2. Cole o script abaixo:

```javascript
function exportToBigQuery() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('silver');
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const rows = data.slice(1).map(row =>
    Object.fromEntries(headers.map((h, i) => [h, row[i] ?? null]))
  );

  const projectId = 'SEU_PROJETO_GCP';
  const datasetId = 'maestro';
  const tableId   = 'total_lancamentos_silver';

  // Truncate + insert via BQ REST API
  const token = ScriptApp.getOAuthToken();
  const url = `https://bigquery.googleapis.com/bigquery/v2/projects/${projectId}/datasets/${datasetId}/tables/${tableId}/insertAll`;

  // Enviar em lotes de 500 (limite do streaming insert)
  const batchSize = 500;
  for (let i = 0; i < rows.length; i += batchSize) {
    const batch = rows.slice(i, i + batchSize).map((r, idx) => ({
      insertId: `row_${i + idx}`,
      json: r
    }));
    const res = UrlFetchApp.fetch(url, {
      method: 'post',
      contentType: 'application/json',
      headers: { Authorization: `Bearer ${token}` },
      payload: JSON.stringify({ rows: batch, skipInvalidRows: true }),
      muteHttpExceptions: true
    });
    if (res.getResponseCode() !== 200) {
      console.error(res.getContentText());
    }
  }
  console.log('✅ Exportado para BigQuery');
}

// Trigger: rodar automaticamente ao editar a aba silver
function createTrigger() {
  ScriptApp.newTrigger('exportToBigQuery')
    .forSpreadsheet(SpreadsheetApp.getActive())
    .onChange()
    .create();
}
```

3. Rode `createTrigger()` uma vez para ativar o gatilho automático
4. Conceda as permissões solicitadas (BigQuery API)

> ⚠️ O Apps Script usa o usuário logado para autenticação — certifique-se que ele tem acesso ao projeto GCP.

### Opção B: WRITE_TRUNCATE a cada exportação

Para substituir toda a tabela (mais simples, evita duplicatas), use o endpoint de Load Job em vez do streaming insert. Adapte conforme necessidade.

---

## Invalidar cache do Worker manualmente

O Worker cacheia por 30 minutos. Para forçar atualização imediata após um upload novo:

```
https://polished-bar-17f8.workers.dev/data?bust=1
```

Ou altere `CACHE_TTL` no `worker.js` para `0` durante testes.

---

## Checklist de Deploy

- [ ] Service Account criada com papéis corretos
- [ ] Chave JSON baixada
- [ ] Dataset `maestro` criado no BigQuery
- [ ] Tabela `total_lancamentos_silver` carregada com o CSV
- [ ] Secrets adicionados no Worker (BQ_SA_JSON, BQ_PROJECT_ID, BQ_DATASET, BQ_TABLE)
- [ ] `worker.js` deployado no Worker `polished-bar-17f8`
- [ ] Testado: `https://polished-bar-17f8.workers.dev/data` retorna JSON
- [ ] `Dashboard_Lancamentos.html` commitado no GitHub
- [ ] GitHub Pages ou Cloudflare Pages ativo
- [ ] Dashboard abrindo e carregando dados do Worker
- [ ] (Opcional) Apps Script configurado para atualização automática
