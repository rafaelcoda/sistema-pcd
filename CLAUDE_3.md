# CLAUDE.md — Sistema PCD (A Apoio)

## O que é este projeto
Sistema web para mapeamento de Pessoas com Deficiência (PCD) em empresas, com foco em conformidade com a **Lei 8.213/91** (cota de PCDs). Permite que o RH gerencie colaboradores, dispare pesquisas, colete respostas via formulário e acompanhe indicadores em tempo real.

---

## Stack
- **Frontend:** HTML puro + CSS + JS vanilla (arquivo único `.html`)
- **Backend/Banco:** Supabase (PostgreSQL + Storage)
- **Biblioteca de ícones:** Tabler Icons (CDN)
- **Planilhas:** SheetJS / xlsx (CDN) — importação e download de modelo

---

## Arquivo principal
`sistema_pcd_v4.html` — tudo em um único arquivo HTML autocontido.

---

## Estrutura do banco (Supabase)

### Tabelas
| Tabela | Descrição |
|---|---|
| `colaboradores` | Cadastro de funcionários (cpf, nome, matricula, email, area, funcao, unidade, telefone, gestor, ativo) |
| `pesquisas` | Campanhas de disparo (titulo, data_envio, prazo, publico, area_alvo, observacoes) |
| `respostas` | Respostas do formulário PCD (colaborador_id, pesquisa_id, e_pcd, tipo_deficiencia, cid, usa_recurso, recursos, adaptacao_necessaria, observacoes) |
| `documentos` | Metadados dos arquivos enviados (resposta_id, colaborador_id, nome_arquivo, storage_path, mime_type, tamanho_bytes) |

### Storage
- Bucket: `laudos-pcd` (privado)
- Caminho dos arquivos: `documentos/{colaborador_id}/{timestamp}_{nome_arquivo}`
- Download via URL assinada (1h de validade)

---

## Credenciais
- Salvas no `localStorage` do navegador com a chave `pcd_cfg`
- Estrutura: `{ url, anon, service }`
- **Anon key** — usada para leitura e operações normais
- **Service role key** — usada para importação em massa (upsert com `merge-duplicates`)
- Nunca hardcoded no código — sempre via aba ⚙️ Configuração

---

## Funcionalidades implementadas

### ✅ Dashboard
- KPIs: total de colaboradores, responderam, aguardando, PCD identificados
- Barra de progresso por KPI
- Alerta de cota legal (Lei 8.213/91) com cálculo automático por faixa de funcionários
- Tabela com filtro por status (respondeu / aguardando / PCD) e busca por nome/matrícula
- Botão "Atualizar" que recarrega dados do banco
- Fallback para dados mock se banco não estiver configurado

### ✅ Formulário
- Busca de colaborador por **CPF** (com máscara automática) **ou matrícula** (ilike, case-insensitive)
- Se CPF não encontrar, tenta matrícula automaticamente e vice-versa
- 5 perguntas com lógica condicional:
  1. É PCD? → se Sim, exibe perguntas 2 e 3
  2. Tipo de deficiência + CID
  3. Usa recurso assistivo? → se Sim, exibe pergunta 4
  4. Quais recursos?
  5. Adaptação necessária + observações (sempre visível)
- Upload de documentos (laudos) com drag & drop — múltiplos arquivos, máx 10MB cada
- Envio salva em `respostas` e faz upload para Storage + registra em `documentos`
- Modo demo funcional sem banco configurado

### ✅ Disparar pesquisa
- Registro de campanhas na tabela `pesquisas`
- Público-alvo: Todos / Apenas aguardando / Por área
- Histórico de campanhas carregado do banco

### ✅ Importar colaboradores
- Download de modelo `.xlsx`
- Upload de planilha com mapeamento automático de colunas (aceita variações de nome)
- Preview com validação linha a linha (verde = válido, vermelho = erro)
- Importação em lotes de 20 com upsert (`merge-duplicates`) — atualiza se matrícula/email já existe
- Usa service role key se disponível
- Barra de progresso durante importação

### ✅ Modal "Ver resposta"
- 3 abas: Dados pessoais / Respostas / Documentos
- Busca dados reais do banco ao abrir (paralelo: `respostas` + `documentos`)
- Documentos com link de download autenticado via Storage

### ✅ Configuração
- Campos para URL, Anon Key e Service Role Key
- Salvar no localStorage + Testar conexão com feedback visual
- Script SQL completo para criar tabelas, bucket e políticas RLS

---

## Padrões de código

### Chamadas ao Supabase
```js
// GET
supaGet('/rest/v1/tabela?filtro=eq.valor', useAdmin?)

// POST (com upsert)
supaPost('/rest/v1/tabela', payload, useAdmin?)

// Upload Storage
supaUpload('laudos-pcd', 'caminho/arquivo.pdf', fileObj, useAdmin?)

// URL assinada para download
supaDownloadUrl('laudos-pcd', 'caminho/arquivo.pdf')
```

### Toast de feedback
```js
toast('Mensagem de sucesso')
toast('Mensagem de erro', true) // true = vermelho
```

### Escape HTML
```js
esc(string) // sempre usar ao inserir dados do banco no DOM
```

### Navegação entre abas
```js
showV('dash')  // dashboard
showV('disp')  // disparar pesquisa
showV('form')  // formulário
showV('imp')   // importar
showV('cfg')   // configuração
```

---

## Decisões de design tomadas
- **Arquivo único** — facilita entrega e uso sem servidor
- **Fallback mock** — sistema funciona para demonstração mesmo sem banco
- **Upsert na importação** — evita duplicatas, atualiza registros existentes
- **Service role key opcional** — sistema funciona só com anon key; service role melhora permissões na importação
- **Busca dupla CPF/matrícula** — colaboradores importados sem CPF são encontrados normalmente
- **RLS aberto** — políticas permissivas para facilitar implantação; produção deve restringir conforme necessidade

---

## Próximas melhorias planejadas
- [ ] Exportar relatório em `.xlsx` com dados consolidados do dashboard
- [ ] Página de resposta pública (link enviado por e-mail ao colaborador)
- [ ] Filtro por área no dashboard
- [ ] Edição de colaborador direto na tabela
- [ ] Histórico de respostas por colaborador (mais de uma resposta ao longo do tempo)
- [ ] Gráfico de distribuição por tipo de deficiência

---

## Como retomar o projeto em nova sessão
1. Cole este arquivo no início da conversa
2. Informe qual melhoria quer implementar
3. Ao finalizar, peça para atualizar o `CLAUDE.md` com o que foi feito
