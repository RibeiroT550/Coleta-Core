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

- Continua **100% client-side** no sentido de "sem servidor de aplicação próprio" — mas a
  persistência compartilhada passa a ser **o próprio GitHub**, via chamadas diretas da API REST do
  GitHub feitas pelo navegador (ver 2.1). Não existe um backend nosso: o "servidor" é o GitHub.
- O template do PDF dos Correios é **fixo** — o parser atual (regex) é a estratégia correta; o
  trabalho aqui é robustez (detectar quando não bateu e avisar claramente), não flexibilização.
- Múltiplos operadores usam o app **independentemente, ao mesmo tempo**, cada um no seu navegador
  — não é multiusuário com sessão/login, mas **precisa produzir um único conjunto de dados
  consistente no fim do dia**.
- Identificação do operador é o **usuário corporativo** (até 7 caracteres, ex. `RTI1CA`) — não é
  nome livre nem exige uma segunda camada de autenticação (ver 2.6).
- Status vindo da planilha dos Correios é **texto livre**, copiado como veio, sem validação contra
  lista fixa.
- Qualquer operador pode rodar a compilação e pode excluir qualquer registro, sob demanda.
- Toda edição/exclusão em registro já compilado gera evento auditável (quem, quando, o quê).
- Retenção de snapshots diários: até 60 dias, granularidade diária; acima de 60 dias, mantém-se só
  o último snapshot de cada mês (ver 2.3.2).

### 2.1 Armazenamento compartilhado: o GitHub como "servidor de dados"

Você perguntou se dá para ter uma pasta padrão no GitHub para onde os arquivos sobem
automaticamente — a resposta é **sim, e é o desenho recomendado**, mas com uma ressalva importante
de expectativa: isso não é "uma pasta que sincroniza sozinha" como o OneDrive faz com o disco.
É o **próprio app, de dentro do navegador, fazendo chamadas HTTPS para a API do GitHub**
(`api.github.com`) para ler e gravar arquivos no repositório — não existe um cliente git rodando
na máquina do operador, nem instalação de nada. Do ponto de vista de quem usa, o efeito é o mesmo
("os dados vão parar num lugar comum automaticamente"), mas o mecanismo é o app chamando a API, não
uma sincronização de pasta de disco.

**O que isso exige, na prática:**
- Um repositório privado dedicado a dados (recomendo **separado** deste `Coleta-Core`, que é só
  código — ex. `Coleta-Core-Data` — para não misturar o histórico de commits de dados com o de
  código, e para poder restringir quem acessa cada um). Fica como decisão em aberto no item 2.10.
- Um **token de acesso** (Personal Access Token do GitHub, com permissão restrita a esse único
  repositório) configurado uma vez em cada navegador/máquina (guardado em `localStorage` daquele
  navegador — nunca vai para o repositório de código, nunca é commitado).
- O repositório de dados **precisa ser privado** — ele vai conter CNPJ, razão social, dados de
  contato etc.; isso é assumido como requisito, não like-to-have.

**Isso resolve, de graça, dois problemas que antes exigiam solução caseira:**
1. **Duplicidade/concorrência** (sua pergunta 4): a API do GitHub para criar/atualizar arquivo
   (`PUT /repos/.../contents/{path}`) exige o `sha` da versão atual do arquivo quando ele já
   existe; se dois operadores tentam escrever o mesmo arquivo "ao mesmo tempo", o segundo pedido
   chega com o `sha` antigo e a API **recusa com erro 409/422** — é uma trava otimista de verdade,
   dada pelo próprio GitHub, não uma convenção de arquivo de lock que pode falhar. Isso substitui
   por completo a ideia de `coleta_core.lock` da versão anterior desta spec.
2. **Trilha de auditoria "de graça"**: cada gravação é um commit — o histórico de commits do
   repositório de dados já é, por si só, um log de tudo que mudou e quando (complementar ao
   `historicoDeAlteracoes` por registro, ver 2.5).

**Estrutura de arquivos no repositório de dados** (mesmo desenho de antes, só troca "pasta local"
por "caminho dentro do repo Git", lido/escrito via API):

```
eventos/
├── pendentes/
│   ├── create_<uuid>.json
│   ├── status-batch_<uuid>.json
│   ├── update_<uuid>.json
│   └── delete_<uuid>.json
└── processados/
    └── … (mesmos arquivos, movidos para cá após compilação — só até expirarem, ver 2.3.2)
historico/
├── historico_2026-09-09.json     (snapshot fechado do dia 9)
├── historico_2026-09-10.json     (snapshot do dia 10 = dia 9 + eventos processados do dia 10)
└── historico_atual.json          (cópia do snapshot mais recente — é o que Consulta/Dashboard leem)
```

O nome de cada evento usa o **`id` da coleta (ou um UUID próprio do evento)** como parte do nome —
não mais o timestamp como prefixo — porque criar um arquivo com nome que já existe também falha
na API do GitHub (sem `sha`, a API assume que é criação e recusa se já houver algo lá), o que dá
uma segunda camada de proteção contra duplicar a mesma captura duas vezes (ex. duplo clique em
"Salvar").

Cada tipo de evento é um arquivo JSON pequeno, com um `tipo` e um `payload`. Nenhum operador
escreve em `historico_atual.json` diretamente — apenas o processo de **Compilar** (rodado por
qualquer operador, botão "Compilar agora") lê os eventos pendentes + o snapshot mais recente e
produz o próximo snapshot.

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
   pendentes existem, obtido listando `eventos/pendentes/` via API).
2. App lê `historico_atual.json` via API — a resposta traz o conteúdo **e o `sha`** atual do
   arquivo (isso é o que viabiliza a trava otimista do passo 5).
3. Lista os arquivos em `eventos/pendentes/`, aplica em ordem determinística (ex. por nome/`id`):
   - `create` → adiciona registro (por `id`).
   - `update` → aplica patch no registro existente por `id` (se o `id` não existir mais — ex.
     evento de outra fonte — loga como inconsistência, não quebra a compilação).
   - `delete` → marca `excluido: true` no registro por `id`.
   - `status-batch` → para cada item da lista, localiza o volume pelo `codigoRastreio` em
     qualquer coleta e atualiza `status`/`statusAtualizadoEm`.
4. Escreve o novo `historico/historico_<AAAA-MM-DD>.json` e faz `PUT` em `historico_atual.json`
   **enviando o `sha` lido no passo 2**.
   - Se o `PUT` for aceito → segue para o passo 5.
   - Se o GitHub recusar por `sha` desatualizado (**outra compilação terminou primeiro**) → o app
     **recomeça do passo 2** automaticamnte (relê o snapshot já atualizado por essa outra
     compilação, filtra os eventos que ela já processou — via a lista de `processados/` que ela já
     moveu — e tenta de novo só com o que sobrou). Isso é concorrência real, não best-effort.
5. Move os eventos aplicados de `pendentes/` para `processados/` (um `DELETE` + `PUT` por arquivo,
   ou um único commit multi-arquivo via a API de Git Trees, para não gerar dezenas de commits
   pequenos).
6. Compilação é **idempotente por nome de arquivo de evento**: se o arquivo já não existe mais em
   `pendentes/` (outra compilação já o moveu), é ignorado silenciosamente.

### 2.3.2 Retenção dos snapshots diários

Regra definida: **até 60 dias, mantém-se 1 snapshot por dia; acima de 60 dias, mantém-se apenas o
último snapshot de cada mês** (ex.: só sobra `historico_2026-07-31.json` para julho, os demais dias
de julho são apagados assim que a janela de 60 dias os ultrapassa).

Importante: isso **não apaga nenhuma coleta dos dados vivos** — `historico_atual.json` é sempre
cumulativo e continua tendo todos os registros desde o início, independentemente da idade. O que é
podado são as **cópias intermediárias** em `historico/`, que servem só como pontos de restauração/
auditoria de "como estava o dado neste dia específico". Rotina sugerida: a cada compilação, depois
de escrever o snapshot do dia, o app verifica se há snapshots com mais de 60 dias que não sejam o
último dia do seu mês, e os remove (via API, um `DELETE` por arquivo).
Os eventos em `eventos/processados/` podem seguir a mesma regra de expurgo (60 dias), já que uma
vez compilados, sua informação de auditoria já está preservada em `historicoDeAlteracoes` dentro do
próprio registro (ver 2.5) — o arquivo de evento bruto não é mais a única cópia dessa informação.

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

Autoria dos eventos (`criadoPor`, auditoria de `update`/`delete`) usa o **usuário corporativo** do
operador — até 7 caracteres alfanuméricos, ex. `RTI1CA`:
- Na primeira abertura do app (por navegador/máquina), pede o usuário corporativo, valida o
  formato (alfanumérico, máx. 7 caracteres, normalizado para maiúsculas) e guarda em
  `localStorage` — reaproveitado em todos os eventos daquela máquina/perfil, sem pedir de novo.
- Tela de configuração permite trocar o usuário salvo (ex. máquina compartilhada entre turnos).
- Não é autenticação (não há senha nem verificação contra um diretório corporativo) — é só
  identificação para rastreabilidade; o controle de acesso real de quem pode usar o app é dado
  pelo token do GitHub configurado na máquina (ver 2.1), não por este campo.

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

### 2.10 Decisões já fechadas (rodada 2)

| # | Pergunta | Decisão |
|---|---|---|
| 1 | Pasta compartilhada | **GitHub como storage**, via chamadas de API do navegador a um repositório de dados privado (ver 2.1) — não é sincronização de pasta local. |
| 2 | Identificação do operador | Usuário corporativo, até 7 caracteres (ex. `RTI1CA`) — ver 2.6. |
| 4 | Duplicidade / concorrência na compilação | Resolvida pela trava otimista nativa da API do GitHub (`sha` em `PUT`) — sem lock caseiro (ver 2.3). |
| 5 | Quem pode excluir | Qualquer operador, por ora. |
| 6 | Retenção dos snapshots | 60 dias em granularidade diária; acima disso, só o último snapshot de cada mês (ver 2.3.2). |

### 2.11 Perguntas ainda em aberto

1. **Excel de status dos Correios** *(bloqueado — aguardando template)*: quais colunas exatamente
   vêm nessa planilha (nomes reais das colunas) — assim que houver um exemplo, desenho o
   autodetect de colunas com precisão (hoje é só a suposição `codigoRastreio` + `status` +
   `dataStatus`).
2. **Nome e local do repositório de dados**: crio um repositório novo (ex.
   `RibeiroT550/Coleta-Core-Data`), separado deste `Coleta-Core` (que fica só com o código do app)?
   Confirma o nome, ou prefere outro esquema (ex. uma branch separada dentro do mesmo repo — viável,
   mas mistura menos bem com o histórico de código)?
3. **Token de acesso ao GitHub**: cada operador vai gerar seu próprio Personal Access Token
   (com escopo restrito só ao repositório de dados) e colar uma vez na tela de configuração do app,
   ou existe preferência por outro mecanismo (ex. um único token "de serviço" compartilhado entre
   todos, mais simples de configurar porém com rastreabilidade de autoria só via o campo
   `criadoPor`, não via autor do commit)?
4. **Gatilho de compilação**: além do botão manual "Compilar agora", faz sentido compilar também
   automaticamente ao entrar na tela de Consulta/Dashboard (silenciosamente, sem exigir clique)?
5. **Limite de tamanho do repositório/arquivo**: `historico_atual.json` cresce para sempre (é
   cumulativo). A API do GitHub tem um limite prático de ~100 MB por arquivo — em volume normal de
   uso isso deve levar anos para ser um problema, mas vale confirmar se há uma expectativa de
   volume diário (quantas coletas/dia, em média) para eu estimar quando isso viraria uma
   preocupação real e se precisamos paginar/particionar o histórico antes disso.
