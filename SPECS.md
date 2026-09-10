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

- O backend deixa de ser "arquivo/repositório simulando banco de dados" e passa a ser um
  **backend de verdade: Microsoft Lists (SharePoint List)**, acessado via API REST/Graph a partir
  do navegador (ver 2.1). Isso substitui a proposta anterior baseada em repositório GitHub.
- O app continua sem servidor de aplicação próprio (nenhum backend escrito por nós) — o "servidor"
  agora é a plataforma SharePoint/Microsoft 365 que a empresa já assina.
- O template do PDF dos Correios é **fixo** — o parser atual (regex) é a estratégia correta; o
  trabalho aqui é robustez (detectar quando não bateu e avisar claramente), não flexibilização.
- Múltiplos operadores usam o app **independentemente, ao mesmo tempo** — como o backend agora é
  uma Lista real, a concorrência é resolvida nativamente pelo próprio SharePoint (ETag por item),
  sem precisarmos de um mecanismo próprio de fila/compilação.
- Identificação do operador é o **usuário corporativo** (até 7 caracteres, ex. `RTI1CA`) — ver 2.6.
- Status vindo da planilha dos Correios é **texto livre**, copiado como veio, sem validação contra
  lista fixa.
- Qualquer operador pode excluir qualquer registro, sob demanda.
- Toda edição/exclusão fica registrada automaticamente pelo **histórico de versões nativo do
  SharePoint** por item — não precisamos construir nosso próprio log de eventos para isso.
- A regra de retenção "60 dias diário, depois 1 por mês" (definida na rodada anterior) foi pensada
  para podar *snapshots* de arquivo — não existe mais esse conceito com uma Lista viva. Fica como
  pergunta em aberto se algo equivalente é necessário para o *histórico de versões* dos itens
  (ver 2.11).

### 2.1 Armazenamento compartilhado: Microsoft Lists como backend

Confirmado com um teste real: dá para criar uma Lista (Microsoft Lists/SharePoint) e **compartilhá-la
diretamente com pessoas específicas**, com granularidade de permissão:
- **Can edit items** — cria, edita e exclui itens, mas não mexe em colunas/views. É o nível certo
  para os operadores.
- **Can edit list** — inclui o anterior + estrutura da lista (colunas, views). Fica com quem
  administra o app (você, por ora).
- **Can view** — só leitura (útil para quem só precisa consultar, ex. um gestor).

Isso já é suficiente para o cenário de vocês, **sem precisar de um Team Site formal** — a Lista foi
criada no seu espaço pessoal ("My Lists") e compartilhada com edição para quem for operar. Uma
Lista tem uma API REST própria (e é acessível via Microsoft Graph), com:
- **Concorrência real por item**: cada item tem um `ETag`; uma atualização (`PATCH`/`MERGE`)
  enviando um `ETag` desatualizado é recusada pelo SharePoint (HTTP 412) — não precisamos reinventar
  trava otimista, o backend já garante isso por registro.
- **Histórico de versão nativo**: com o versionamento da lista ativado (configuração padrão em
  Microsoft Lists), toda edição/exclusão de item já fica registrada — quem alterou, quando, valores
  anteriores — sem precisarmos manter nosso próprio log de eventos.

Isso elimina toda a camada de "eventos + compilação diária" desenhada na rodada anterior (ela
existia especificamente para contornar a ausência de um backend com escrita concorrente segura).
Com Lists, o app faz **CRUD direto**: cada operador cria/edita/exclui itens na hora, e o próprio
SharePoint arbitra concorrência e mantém auditoria.

**Estrutura de dados proposta: duas Listas relacionadas** (em vez de um único arquivo/JSON):

| Lista | Grão | Campos principais |
|---|---|---|
| **Coletas** | 1 item por coleta | RazaoSocial, CNPJ, Cidade, UF, Categoria, Consultor, PessoaContato, TipoEnvio, NumeroColeta, ArquivoOrigemPDF, CriadoPor, CriadoEm |
| **Volumes** | 1 item por volume | ColetaId (coluna de pesquisa/lookup para Coletas), NF, VolIndex, VolTotal, Itens (texto-resumo dos PNs daquela NF, como hoje), CodigoRastreio, Status, StatusAtualizadoEm |

Essa divisão evita repetir os dados da empresa em cada volume (o CSV atual repete), mantém o grão
de "1 rastreio por volume" que já existe hoje, e permite localizar um volume pelo `CodigoRastreio`
diretamente (importante para 2.4 — atualização de status em lote).

> Nota técnica em aberto (ver 2.11): como a Lista foi criada em "My Lists" (espaço pessoal, não um
> site de equipe), preciso confirmar o endereço/endpoint real da API REST para essa lista e como a
> autenticação vai funcionar quando o app é aberto fora do próprio SharePoint (arquivo local, ou
> hospedado em outro lugar) — é a peça que falta para fechar este desenho por completo.

### 2.2 Modelo de dados — registro de Coleta

O mesmo modelo de dados da rodada anterior continua válido conceitualmente (coleta → NFs → itens +
volumes); a diferença é que ele deixa de ser "um JSON grande por coleta" e passa a ser distribuído
entre as duas Listas (2.1): os campos de `empresa` e dados gerais viram 1 item em **Coletas**; cada
combinação NF+volume vira 1 item em **Volumes**, referenciando a coleta via lookup. Os `itens`
(PN/descrição/qtd) de cada NF continuam guardados como texto-resumo no item de Volumes correspondente
(mesmo formato que a função `itensTextoPorNf` já produz hoje no protótipo) — normalização completa
de itens em uma terceira Lista fica como possível evolução futura, não necessária agora.

### 2.3 CRUD direto na Lista (sem etapa de compilação)

- **Create**: ao salvar uma Nova Captura, o app faz 1 `POST` para criar o item em **Coletas** e N
  `POST`s (1 por volume) em **Volumes**, já com o `ColetaId` retornado pela primeira chamada.
- **Read**: Consulta/Histórico e Dashboard leem via `GET` nas duas Listas (com `$expand`/lookup para
  juntar volume↔coleta), com paginação (`$top`/`$skiptoken`) se o volume de dados crescer.
- **Update**: edição de um registro existente faz `PATCH`/`MERGE` no item correspondente, enviando
  o `ETag` lido — se alguém alterou o mesmo item entre a leitura e a gravação, o SharePoint recusa
  (412) e o app pede para o operador recarregar e tentar de novo (conflito real, tratado na hora,
  não é preciso reprocessar nada depois).
- **Delete**: exclusão lógica (campo `Excluido` = true) preservando o item para auditoria via
  histórico de versão, com opção de UI "mostrar excluídos" — mesma decisão da rodada anterior,
  só que sem precisar de um evento próprio: é só mais um campo do item.
- **Auditoria**: em vez de um `historicoDeAlteracoes` construído por nós, a tela de detalhe do
  registro pode consumir o **histórico de versões nativo do item** (`/versions` na API do
  SharePoint) para mostrar quem alterou o quê e quando.

### 2.4 Atualização de status via planilha dos Correios

- Tela dedicada: "Atualizar status de rastreio".
- Usuário carrega o Excel exportado dos Correios; app pede/confere o mapeamento de colunas
  (mesmo padrão já existente em `CadastroStore`: autodetecta `codigoRastreio` e `status` —
  e opcionalmente `dataStatus` —, confirma com o usuário se não detectar).
- Para cada linha da planilha, o app busca o item correspondente na Lista **Volumes** por
  `CodigoRastreio` (`GET` filtrado) e faz `PATCH` direto no item (`Status`, `StatusAtualizadoEm`) —
  **sem etapa de compilação**: a atualização já vale assim que a planilha é processada.
- Resumo pós-processamento: quantos códigos da planilha deram match vs. não encontrados no
  histórico (para o usuário perceber planilhas erradas/atrasadas).

### 2.5 CRUD completo — resumo

- **Create/Read/Update/Delete**: ver 2.3 — tudo é chamada direta às Listas, sem etapa intermediária.
- **Auditoria**: histórico de versão nativo por item (2.1/2.3), em vez de log próprio.

### 2.6 Identificação do operador

Autoria (`CriadoPor`, e quem aparece no histórico de versão do SharePoint) usa o **usuário
corporativo** do operador — até 7 caracteres alfanuméricos, ex. `RTI1CA`:
- Na primeira abertura do app (por navegador/máquina), pede o usuário corporativo, valida o
  formato (alfanumérico, máx. 7 caracteres, normalizado para maiúsculas) e guarda em
  `localStorage` — reaproveitado em todas as gravações daquela máquina/perfil, sem pedir de novo.
- Tela de configuração permite trocar o usuário salvo (ex. máquina compartilhada entre turnos).
- Vale notar: como a autenticação real na Lista já é feita pelo login corporativo do Microsoft 365
  (ver 2.11 — depende de como resolvermos o acesso à API), o SharePoint também sabe "quem" gravou
  cada versão pelo seu próprio login — este campo (`CriadoPor`) é redundante com isso, mas mantido
  para simplificar consultas/relatórios dentro do próprio app sem precisar cruzar com a API de
  versões a cada consulta.

### 2.7 Dashboard

Tela nova, alimentada por leitura direta das Listas **Coletas** e **Volumes** (2.1):
- **KPIs**: total de coletas no período, volumes sem rastreio preenchido, volumes sem status
  atualizado, distribuição por status, distribuição por categoria de cliente/consultor.
- **Filtros**: período (data de criação), cliente/CNPJ, categoria/canal de cliente, consultor,
  status.
- Como não há mais etapa de compilação, os dados no Dashboard são sempre o estado atual real das
  Listas — não existe mais o conceito de "dados pendentes de compilar".

### 2.8 Exportação para Excel

- Botão de exportação em Consulta/Dashboard, respeitando os filtros ativos na tela.
- Duas granularidades de exportação (usuário escolhe):
  - **Por volume** (equivalente ao CSV atual — uma linha por volume/rastreio, join Coletas+Volumes).
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
| 1 | Onde os dados ficam compartilhados | **Microsoft Lists (SharePoint)** — 2 Listas (Coletas, Volumes), compartilhadas com os operadores no nível "Can edit items" (ver 2.1). Substitui a proposta anterior de repositório GitHub. |
| 2 | Identificação do operador | Usuário corporativo, até 7 caracteres (ex. `RTI1CA`) — ver 2.6. |
| 3 | Duplicidade / concorrência | Resolvida nativamente pelo `ETag` por item do SharePoint — sem lock ou compilação caseira (ver 2.1/2.3). |
| 4 | Quem pode excluir | Qualquer operador, por ora. |
| 5 | Auditoria de alterações | Histórico de versão nativo do SharePoint por item, em vez de log de eventos próprio (ver 2.1/2.3). |

> A pergunta de retenção de snapshots (fechada na rodada anterior como "60 dias diário, depois 1
> por mês") deixou de se aplicar da forma como foi pensada — não existem mais snapshots de arquivo.
> Se fizer sentido um equivalente para o histórico de versões dos itens, é um novo item em 2.11.

### 2.11 Perguntas ainda em aberto

1. **Excel de status dos Correios** *(bloqueado — aguardando template)*: quais colunas exatamente
   vêm nessa planilha (nomes reais das colunas) — assim que houver um exemplo, desenho o
   autodetect de colunas com precisão (hoje é só a suposição `codigoRastreio` + `status` +
   `dataStatus`).
2. **Autenticação do app contra a Lista** — **rota (a) testada e descartada**: confirmamos, com um
   teste real, que o OneDrive/SharePoint renderiza arquivos `.html` enviados por usuário dentro de
   um `blob:` isolado (sandbox), nunca como página real na origem do site — é bloqueio proposital
   de segurança do próprio Microsoft 365 (evita HTML malicioso sequestrar a sessão do site), não uma
   configuração que dá para contornar por fora do app. Ver evidência: ao abrir o arquivo de teste
   pela web, a URL ficou `blob:https://bosch-my.sharepoint.com/...` e o `fetch` relativo falhou com
   `is not a valid URL` — o navegador nem considera aquilo uma origem HTTPS de verdade.
   - **Rota (b) é o caminho**: autenticar via **Microsoft Entra ID + MSAL.js**, chamando a Lista via
     Microsoft Graph. Testamos se você tem permissão de self-service para criar o "App registration"
     necessário — **não tem** (tela abre, mas sem permissão para registrar). Isso não bloqueia o
     projeto, só significa que é preciso um pedido pontual ao TI/administrador do Entra ID — feito
     uma vez, sem dependência contínua depois disso. O pedido está pronto para envio (ver 2.12).

3. **Site de equipe vs. espaço pessoal**: mesmo com o compartilhamento direto funcionando, faz
   sentido migrar a Lista para um site de equipe formal mais adiante (mais robusto a longo prazo —
   ex. se sua conta pessoal for desativada um dia, o site de equipe não depende dela)? Não bloqueia
   o desenvolvimento agora, mas é bom já ter isso mapeado como possível ação futura de TI.
4. **Volume esperado**: quantas coletas por dia, em média (e nos picos)? Ajuda a dimensionar se
   listagens simples (`GET` com filtro) bastam ou se algum ponto (ex. dashboard com muitos filtros
   cruzados) precisa de views/índices dedicados na Lista desde já.
5. **Onde o app vai rodar no dia a dia**: isso define o "Redirect URI" do pedido de 2.12 — uma URL
   pública (ex. GitHub Pages deste repositório, mais prático — todo operador só abre um link), rodar
   localmente na máquina de cada um (`http://localhost:...`, exige um passo extra por operador), ou
   outra hospedagem que você já tenha em mente?

### 2.12 Pedido para o TI/administrador do Entra ID (Azure AD)

Configuração única, para viabilizar o app de Controle de Coletas conversar com a Lista do
SharePoint usando o próprio login corporativo de cada operador (sem senhas/tokens avulsos).

**O que pedir, literalmente:**

> Preciso que seja criado um **App registration** no Entra ID (Azure AD) da empresa, tipo
> **"Single-page application (SPA)"**, para um app interno de controle de coletas dos Correios.
>
> - **Redirect URI**: `<A DEFINIR — ver 2.11, item 5>`
> - **Permissão de API solicitada**: Microsoft Graph → `Sites.Selected` (delegada).
> - Depois de criado, preciso que o administrador **conceda esse app acesso apenas ao site do
>   SharePoint onde está a Lista "Teste"** (usando a permissão `Sites.Selected` — dá acesso só a
>   esse site específico, não a todos os sites da empresa).
> - Ao final, preciso receber o **Application (client) ID** e o **Directory (tenant) ID** gerados.

`Sites.Selected` é a opção recomendada porque restringe o acesso do app a **apenas este site**
(o administrador concede explicitamente); a alternativa mais simples de configurar seria
`Sites.ReadWrite.All`, mas essa dá acesso a **todos os sites do SharePoint da empresa** — não
recomendo pedir isso só para o nosso caso de uso.
