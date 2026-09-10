# Coleta Core

Aplicação web single-page para controle de coletas dos Correios (devolução Diesel — Bosch Automotive).

## O que faz

- **Nova Captura**: importa o PDF do "Resumo da Solicitação" de coleta, extrai automaticamente empresa, CNPJ, NFs, volumes e itens (via `pdf.js`), cruza o CNPJ com uma base de cadastro em Excel (via `xlsx.js`) e permite registrar o número da coleta e o código de rastreio de cada volume.
- **Consulta e Histórico**: pesquisa nos registros já salvos (empresa, CNPJ, NF, coleta, rastreio).

## Armazenamento

- Sem backend: roda inteiramente no navegador.
- Em navegadores com suporte a File System Access API (Edge/Chrome), grava diretamente em um arquivo CSV local (`historico_coletas.csv`) e mantém backups diários completos em uma pasta escolhida pelo usuário.
- Em navegadores sem suporte, usa `localStorage` como fallback e permite exportar o CSV manualmente.
- A base de cadastro (Excel) é carregada localmente e indexada por CNPJ; o mapeamento de colunas é detectado automaticamente ou confirmado pelo usuário na primeira carga.

## Como rodar

Basta abrir `index.html` em um navegador (Edge ou Chrome recomendados, para gravação direta em arquivo).

## Estrutura

- `index.html` — aplicação completa (HTML + CSS + JS em um único arquivo).
