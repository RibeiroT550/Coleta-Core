# Especificações — Coleta Core

> Documento vivo. A Parte 1 descreve o que o protótipo (`index.html`) já implementa e como.
> A Parte 2 lista lacunas e decisões em aberto que precisamos fechar antes de evoluir o código.

## Parte 1 — Estado atual (engenharia reversa do protótipo)

### 1.1 Visão geral

Single Page Application (SPA) 100% client-side, arquivo único (`index.html`: HTML + CSS + JS
inline), sem backend e sem build step. Tema visual com identidade Bosch (cores/logo hardcoded).
Duas telas (seções), alternadas via JS, sem roteamento por URL:

1. **Nova Captura** — cadastrar uma nova coleta a partir de um PDF.
2. **Consulta e Histórico** — buscar registros já salvos.

Bibliotecas de terceiros via CDN: `pdf.js` (extração de texto de PDF) e `xlsx.js` (leitura de
planilhas Excel). Não há dependências de build, framework ou package manager.

### 1.2 Persistência de dados

Não há servidor nem banco de dados. Tudo depende de recursos do navegador:

| Mecanismo | Uso | Suporte |
|---|---|---|
| **File System Access API** (`showSaveFilePicker`, `showDirectoryPicker`, `showOpenFilePicker`) | Ler/escrever o CSV de histórico direto em um arquivo local; escolher a pasta de backups | Só Chrome/Edge (Chromium) |
| **IndexedDB** (`controleColetasDB`) | Persistir os `FileSystemHandle` (histórico, cadastro, pasta de backup) entre sessões, para reconectar sem pedir o arquivo de novo | Chromium |
| **localStorage** | Fallback quando File System Access não existe: guarda os registros em `historico_fallback` (JSON) e o mapeamento de colunas do Excel (`cadastroMapping_v1`) | Qualquer navegador |
| Download de arquivo (`<a download>`) | No modo fallback, cada novo registro dispara o download de um CSV atualizado | Qualquer navegador |

**Implicações importantes:**
- O "banco de dados" é um **arquivo CSV local**, editável por fora da aplicação (Excel, editor de texto). Não há transações, locks nem controle de concorrência — se dois usuários abrirem o mesmo arquivo local (ex.: numa pasta de rede) e salvarem ao mesmo tempo, um pode sobrescrever o outro (a escrita é sempre "ler tudo → concatenar → reescrever tudo").
- Não existe multiusuário real nem sincronização entre máquinas, exceto o que uma pasta sincronizada (OneDrive/SharePoint) oferecer por fora da aplicação.
- Sem File System Access API (Firefox, Safari), a "persistência" é local ao navegador (`localStorage`) — perdida se o usuário limpar dados do navegador, e não compartilhada entre dispositivos.

### 1.3 Modelo de dados

**Registro de histórico** (uma linha de CSV = um volume dentro de uma NF de uma coleta):

```
DataHoraRegistro, Empresa, CNPJ, Cidade, UF, CategoriaCliente, Consultor,
PessoaContato, TipoEnvio, NF, VolumeIndice, Itens, QtdVolumesTotal,
NumeroColeta, CodigoRastreio, ArquivoOrigemPDF
```

- Granularidade: **1 linha por volume** (não por NF, não por item). `Itens` é uma string
  concatenada de todos os PN/descrição/qtd daquela NF (não normalizado).
- `VolumeIndice` no formato texto `"i/total"` (ex. `2/3`).
- CSV delimitado por `;`, escaping por aspas duplas (padrão RFC4180-like, à mão), sem biblioteca.
- Cabeçalho fixo (`CSV_HEADER`), sem versionamento de schema.

**Base de cadastro (Excel)**: sem schema fixo — o app detecta/pede o mapeamento de colunas para
5 campos (`cnpj`, `razaoSocial`, `cidade`, `uf`, `categoria`, `consultor`) e indexa em memória por
CNPJ normalizado (só dígitos). Mapeamento persistido em `localStorage`, não faz parte do arquivo
de histórico.

### 1.4 Fluxo — Nova Captura

1. Usuário opcionalmente conecta o **arquivo de histórico** (CSV) e/ou carrega a **base de
   cadastro** (Excel) — indicadores de status ("Conectado"/"Não conectado").
2. Usuário arrasta ou seleciona o **PDF da Solicitação de Coleta**.
3. `PdfExtractor` extrai o texto do PDF por posição (agrupando por linha/Y, inserindo espaço só
   quando há gap real entre runs de texto — evita quebrar PNs/códigos numéricos).
4. `parseResumo` aplica regex sobre o texto para extrair: Empresa, CNPJ, Pessoa de contato, Tipo
   de Envio, Quantidade total de volumes, e a lista de NFs com sua quantidade de volumes
   (`NF <n>: <n> volume(s)`).
5. `parseDetalhamento` varre a seção "Detalhamento:" do PDF linha a linha com uma regex posicional
   fixa (`^(NF) (Descrição) (Código) (Qtd)$`) para montar os itens (PN/descrição/qtd) de cada NF.
6. Campos extraídos aparecem destacados em azul (`.auto-filled`); se o CNPJ existir na base de
   cadastro carregada, cidade/UF/categoria/consultor são preenchidos automaticamente também; caso
   contrário, um aviso pede preenchimento manual.
7. Duas tabelas editáveis são geradas:
   - **Itens da Solicitação**: uma linha por NF+PN (pode ter várias por NF).
   - **Coleta e Rastreio**: uma linha por volume (não por PN) — é aqui que o usuário digita o
     código de rastreio de cada volume, e um único campo de "Nº da Coleta" vale para toda a
     captura.
8. Ambas as tabelas suportam edição inline e adicionar/remover linha manualmente (para PDFs que
   não seguem o padrão esperado ou não vieram de PDF).
9. **Salvar**: valida que existe pelo menos 1 volume preenchido; confirma com o usuário se
   faltarem Nº da coleta ou algum rastreio; grava 1 linha de CSV por volume, disparando também a
   rotina de backup diário (se aplicável) e limpando o formulário.

### 1.5 Fluxo — Consulta e Histórico

- Botão "Conectar" (arquivo de histórico) e "Recarregar" (relê do disco).
- Busca full-text simples (client-side, sem index) sobre todos os campos do registro.
- Tabela somente leitura, mais recente primeiro.
- "Restaurar de um backup…": abre um CSV de backup, confirma com o usuário e **substitui
  integralmente** o arquivo de histórico principal pelo conteúdo do backup escolhido.

### 1.6 Backup

- Na primeira gravação do dia (`lastBackupDate` em `localStorage` != hoje), pede (uma única vez,
  via `showDirectoryPicker`) uma pasta para backups e a lembra via IndexedDB.
- A cada gravação subsequente no mesmo dia, sobrescreve `backup_historico_<AAAA-MM-DD>.csv` com o
  estado **completo** do histórico até aquele momento (não é incremental/diff).
- Falha de backup é apenas logada no console (`console.warn`) — o usuário não é avisado na UI.

### 1.7 O que a ferramenta cobre hoje (resumo funcional)

| Necessidade | Cobertura atual |
|---|---|
| Extrair dados de um PDF de solicitação de coleta | ✅ via regex específicas para um template fixo |
| Evitar redigitar dados de empresas já conhecidas | ✅ via cruzamento com planilha de cadastro (CNPJ) |
| Registrar Nº de coleta + rastreio por volume | ✅ |
| Consultar histórico | ✅ busca simples client-side |
| Persistir sem servidor | ✅ arquivo local (Chromium) ou localStorage (fallback) |
| Proteção contra perda de dados | ⚠️ parcial — backup diário completo, mas sem versionamento granular nem verificação de integridade |
| Multiusuário / concorrência | ❌ não tratado |
| Validação de dados (CNPJ, formatos) | ❌ nenhuma validação de formato, só campos vazios |
| Edição/exclusão de um registro já salvo | ❌ histórico é somente leitura na UI |
| Suporte fora de Chromium | ⚠️ degrada para localStorage/download manual, sem paridade de recursos (sem backup automático, sem restauração) |

### 1.8 Riscos e dívidas técnicas identificadas

- **Parsing frágil**: `parseResumo`/`parseDetalhamento` dependem de um template textual exato do
  PDF exportado pelos Correios; qualquer mudança de layout ou template diferente quebra a
  extração silenciosamente (cai no fluxo manual, mas sem diagnóstico do que não bateu).
- **Sem normalização relacional**: item (PN) e volume (rastreio) são capturados em duas tabelas
  paralelas indexadas por NF, mas não há ligação direta item↔volume — só é possível reconstruir a
  associação porque cada linha salva repete o texto de todos os itens da NF em `Itens`.
- **Escrita não atômica**: cada save relê o CSV inteiro, concatena em memória e reescreve o
  arquivo inteiro — arriscado sob concorrência e ineficiente para históricos grandes.
- **Sem tratamento de erros de usuário**: `alert()`/`confirm()` nativos do navegador para tudo.
- **Sem autenticação/autorização**: qualquer pessoa com o arquivo local tem acesso completo.
- **Tudo em um arquivo**: 1420 linhas de HTML+CSS+JS misturados, sem módulos ES, sem testes.

---

## Parte 2 — Nova arquitetura (decisões e specs para evolução)

### 2.0 Premissas confirmadas

- Continua **100% client-side** — sem servidor/backend/banco de dados.
- O template do PDF dos Correios é **fixo** — o parser atual (regex) é a estratégia correta; o
  trabalho aqui é robustez (detectar quando não bateu e avisar claramente), não flexibilização.
- Múltiplos operadores usam o app **independentemente, ao mesmo tempo**, cada um no seu navegador
  — não é multiusuário com sessão/login, mas **precisa produzir um único conjunto de dados
  consistente no fim do dia**.
- Todos os operadores têm, localmente, acesso a **uma mesma pasta sincronizada** (OneDrive/
  SharePoint/rede) — é essa pasta compartilhada que viabiliza "vários navegadores, um dataset".
- Status vindo da planilha dos Correios é **texto livre**, copiado como veio, sem validação contra
  lista fixa.
- Qualquer operador pode rodar a compilação, sob demanda.
- Toda edição/exclusão em registro já compilado gera evento auditável (quem, quando, o quê).

### 2.1 Modelo mental: "eventos" + "compilação" (sem servidor, sem lock real)

Como não há backend para arbitrar escritas concorrentes, a arquitetura muda de
**"1 arquivo compartilhado, reescrito por todo mundo"** (modelo atual, frágil) para
**"cada ação gera um arquivo novo e imutável; um processo de compilação consolida tudo depois"**.
Isso elimina o cenário de dois navegadores reescrevendo o mesmo CSV ao mesmo tempo: cada operador
só cria arquivos com nome único, nunca edita o de outro nem o arquivo geral diretamente.

```
/coleta-core-data/                          (pasta sincronizada — raiz configurada uma vez por operador)
├── eventos/
│   ├── pendentes/
│   │   ├── 2026-09-10T14-32-01_create_<uuid>.json
│   │   ├── 2026-09-10T15-01-09_status-batch_<uuid>.json
│   │   ├── 2026-09-10T16-40-22_update_<uuid>.json
│   │   └── 2026-09-10T17-05-00_delete_<uuid>.json
│   └── processados/
│       └── … (mesmos arquivos, movidos para cá após entrarem numa compilação)
├── historico/
│   ├── historico_2026-09-09.json     (snapshot fechado do dia 9: base do dia + eventos do dia 9)
│   ├── historico_2026-09-10.json     (snapshot do dia 10 = historico_2026-09-09 + eventos processados do dia 10)
│   └── historico_atual.json          (cópia do snapshot mais recente — é o que Consulta/Dashboard leem)
└── coleta_core.lock                  (arquivo de trava temporária durante uma compilação)
```

Cada tipo de evento é um arquivo JSON pequeno, versionado no nome pelo timestamp + UUID, com um
`tipo` e um `payload`. Nenhum operador escreve no `historico_atual.json` diretamente — apenas o
processo de **Compilar** (rodado por qualquer operador, botão "Compilar agora") lê os eventos
pendentes + o snapshot mais recente e produz o próximo snapshot.

**Tipos de evento:**
| Tipo | Gerado quando | Conteúdo |
|---|---|---|
| `create` | Usuário salva uma nova captura (Nova Captura) | Registro completo da coleta (ver 2.2) |
| `update` | Usuário edita um registro existente (tela de CRUD) | `coletaId`, campos alterados (antes/depois), autor, timestamp |
| `delete` | Usuário exclui um registro | `coletaId`, motivo (opcional), autor, timestamp — exclusão lógica (tombstone), nunca remove do histórico |
| `status-batch` | Usuário importa a planilha de status dos Correios | Lista de `{codigoRastreio, status, dataStatus, ...}` a aplicar sobre os volumes correspondentes |

### 2.2 Modelo de dados — registro de Coleta (normalizado)

Diferente do CSV atual (uma linha "achatada" por volume, item repetido como texto), o registro
passa a ser um objeto único por coleta, com NF → Itens e NF → Volumes como listas próprias —
resolve a lacuna identificada em 1.8 (item↔volume sem ligação real) e permite CRUD granular.

```jsonc
{
  "id": "c-6f2a1e30",                 // UUID gerado na criação, estável para sempre
  "criadoEm": "2026-09-10T14:32:01Z",
  "criadoPor": "nome.operador",       // identificação do operador (ver 2.6)
  "atualizadoEm": "2026-09-10T14:32:01Z",
  "excluido": false,                  // tombstone lógico (evento delete marca true)
  "empresa": {
    "razaoSocial": "Cliente X Ltda",
    "cnpj": "00000000000000",
    "cidade": "Campinas",
    "uf": "SP",
    "categoria": "OEM",
    "consultor": "Fulano",
    "pessoaContato": "Ciclano"
  },
  "tipoEnvio": "Solicitação de Coleta",
  "numeroColeta": "123456789",
  "arquivoOrigemPDF": "resumo_coleta_0001.pdf",
  "nfs": [
    {
      "nf": "8823",
      "itens": [
        { "pn": "1928374", "descricao": "Bico injetor", "qtd": "2" }
      ],
      "volumes": [
        {
          "volIndex": 1, "volTotal": 2,
          "codigoRastreio": "OB123456789BR",
          "status": "Objeto entregue ao destinatário",   // texto livre, vindo do Excel Correios
          "statusAtualizadoEm": "2026-09-15T09:00:00Z"
        }
      ]
    }
  ]
}
```

### 2.3 Fluxo de compilação ("Compilar agora")

1. Operador clica "Compilar agora" (disponível nas duas telas, com indicador de quantos eventos
   pendentes existem na pasta).
2. App tenta criar `coleta_core.lock` (com timestamp + quem está compilando). Se o lock já existir
   e tiver menos de N minutos, avisa "Compilação em andamento por <quem>, tente novamente em
   instantes" e aborta — mitigação best-effort (não é um lock distribuído de verdade, mas cobre o
   caso comum de dois cliques quase simultâneos).
3. Lê `historico_atual.json` (ou o snapshot mais recente em `historico/`) como base.
4. Lista os arquivos em `eventos/pendentes/`, ordena por timestamp do nome, aplica em ordem:
   - `create` → adiciona registro (por `id`).
   - `update` → aplica patch no registro existente por `id` (se o `id` não existir mais — ex.
     evento de outra fonte — loga como inconsistência, não quebra a compilação).
   - `delete` → marca `excluido: true` no registro por `id`.
   - `status-batch` → para cada item da lista, localiza o volume pelo `codigoRastreio` em
     qualquer coleta e atualiza `status`/`statusAtualizadoEm`.
5. Escreve o novo `historico/historico_<AAAA-MM-DD>.json` (snapshot do dia corrente) e atualiza
   `historico_atual.json`.
6. Move os eventos aplicados de `pendentes/` para `processados/` (mantém trilha de auditoria e
   permite reprocessar/depurar se algo parecer errado).
7. Remove o `coleta_core.lock`.
8. Compilação é **idempotente por `id` de evento**: se um evento já está em `processados/`, é
   ignorado mesmo que apareça de novo em `pendentes/` (proteção extra contra reprocessamento).

> Observação de risco a registrar: como não é um lock distribuído real, uma corrida entre duas
> compilações quase simultâneas ainda é teoricamente possível (ex. duas pessoas passam pelo
> "arquivo não existe ainda" ao mesmo tempo antes do primeiro `write` do lock). Como a sincronização
> (OneDrive/SharePoint) tem sua própria latência, aceitar esse risco residual é razoável para o
> volume de uso esperado, mas deve ficar documentado — não é uma garantia forte de exclusão mútua.

### 2.4 Atualização de status via planilha dos Correios

- Tela dedicada (ou seção dentro de Consulta/Dashboard): "Atualizar status de rastreio".
- Usuário carrega o Excel exportado dos Correios; app pede/confere o mapeamento de colunas
  (reaproveita o padrão já existente em `CadastroStore`: autodetecta `codigoRastreio` e `status`
  — e opcionalmente `dataStatus` —, confirma com o usuário se não detectar).
- App **não** grava direto no histórico: gera um evento `status-batch` com todas as linhas da
  planilha (mesmo as que não derem match ainda — o match final é feito na compilação, contra o
  snapshot mais atual disponível naquele momento).
- Após gerar o evento, mostra um resumo pré-compilação: quantos códigos foram encontrados na
  planilha vs. quantos existem hoje no histórico (para o usuário perceber planilhas erradas antes
  de compilar) — a atualização de fato só é refletida no histórico após "Compilar agora".

### 2.5 CRUD completo

- **Create**: fluxo atual de Nova Captura, adaptado para gerar o registro normalizado (2.2) e
  gravar como evento `create` em `eventos/pendentes/` (em vez de linha de CSV).
- **Read**: tela de Consulta/Histórico e Dashboard leem sempre `historico_atual.json` (nunca os
  eventos pendentes diretamente — dado "em trânsito" só aparece após compilar; a UI deixa isso
  explícito, ex. "3 capturas suas ainda não compiladas").
- **Update**: nova tela/modal a partir da Consulta — abre um registro existente, permite editar
  qualquer campo (dados da empresa, NF, itens, volumes, número da coleta, rastreio, status manual).
  Ao salvar, gera evento `update` com diff (campo, valor anterior, valor novo) — nunca edita
  `historico_atual.json` diretamente.
- **Delete**: exclusão lógica (tombstone) via evento `delete`; registros excluídos somem das
  listagens padrão mas continuam no snapshot para auditoria, com opção de UI "mostrar excluídos".
- **Auditoria**: cada registro passa a ter um `historicoDeAlteracoes` (lista de eventos aplicados
  a ele: tipo, autor, timestamp, diff) — visível numa aba de detalhe do registro.

### 2.6 Identificação do operador

Necessário para autoria dos eventos (`criadoPor`, auditoria de `update`/`delete`). Sem login/
autenticação real (fora de escopo, é client-side puro), a proposta mínima:
- Na primeira abertura do app (por navegador/máquina), pedir "Seu nome" uma vez e guardar em
  `localStorage` — reaproveitado em todos os eventos daquela máquina/perfil.
- Sem validação forte de identidade (é um app sem backend); serve para rastreabilidade
  operacional, não para controle de acesso.
- **Em aberto**: definir se isso é só um nome livre ou uma lista fixa de operadores conhecidos
  (select) — impacta consistência dos relatórios "por operador", se vierem a existir.

### 2.7 Dashboard

Tela nova, alimentada por `historico_atual.json` (mesma leitura da Consulta):
- **KPIs**: total de coletas no período, volumes sem rastreio preenchido, volumes sem status
  atualizado, distribuição por status, distribuição por categoria de cliente/consultor.
- **Filtros**: período (data de criação), cliente/CNPJ, categoria/canal de cliente, consultor,
  status.
- **Indicador de "dados pendentes de compilar"**: quantos eventos existem em `pendentes/` agora,
  para deixar claro que o dashboard pode estar levemente atrasado em relação ao que já foi
  capturado por outros operadores.

### 2.8 Exportação para Excel

- Botão de exportação em Consulta/Dashboard, respeitando os filtros ativos na tela.
- Duas granularidades de exportação (usuário escolhe):
  - **Por volume** (equivalente ao CSV atual — uma linha por volume/rastreio).
  - **Por coleta** (uma linha por coleta, itens/volumes agregados em colunas resumo) — útil para
    visão gerencial.
- Exportação "tudo" (sem filtro) e "filtrado" (o que estiver na tela: por canal/categoria de
  cliente, período, status etc.) usando os mesmos filtros do Dashboard/Consulta.
- Formato de saída: `.xlsx` real (via `xlsx.js`, já é dependência do projeto), não apenas CSV.

### 2.9 Robustez do parser de PDF (dado que o template é fixo)

- Quando `parseResumo`/`parseDetalhamento` não encontrarem os campos esperados, a UI deve
  **diagnosticar explicitamente** o que não bateu (ex. "Não foi possível localizar a seção
  'Detalhamento:' neste PDF") em vez do estado genérico atual ("não foi possível identificar
  automaticamente os campos").
- Adicionar validação leve pós-parse: nº de volumes somados por NF bate com `Quantidade total de
  Volumes`; CNPJ tem 14 dígitos; alertar (não bloquear) divergências antes de salvar.

### 2.10 Perguntas ainda em aberto

Estas precisam de resposta antes de detalhar as telas/eventos por completo:

1. **Configuração da pasta compartilhada**: cada operador seleciona a pasta raiz uma vez (via
   `showDirectoryPicker`, guardado no IndexedDB daquele navegador) — confirma esse modelo, ou deve
   haver um caminho fixo sugerido/documentado para toda a equipe usar?
2. **Operador**: nome livre ou lista fixa de operadores (select) para consistência de relatórios?
3. **Excel de status dos Correios**: quais colunas exatamente vêm nessa planilha (nomes de coluna
   reais, mesmo que variem) — preciso de um exemplo ou de uma planilha real para desenhar o
   autodetect de colunas com precisão (hoje só tenho a suposição `codigoRastreio` + `status` +
   `dataStatus`).
4. **Frequência/gatilho de compilação**: além do botão manual "Compilar agora", faz sentido também
   compilar automaticamente ao entrar na tela de Consulta/Dashboard (silenciosamente, sem exigir
   clique), já que qualquer operador pode compilar?
5. **Exclusão**: quem pode excluir um registro — qualquer operador, ou deve existir algum nível de
   permissão (ex. só quem criou, ou um papel "admin") mesmo sem login formal?
6. **Retenção**: os snapshots diários em `historico/` (um arquivo por dia, para sempre) devem ter
   algum limite/expurgo, ou ficam todos indefinidamente como trilha de auditoria?
