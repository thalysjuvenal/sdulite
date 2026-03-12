# SDULite

Editor SQLite no browser para consultores TOTVS Protheus. Abra, edite e salve bases `.sqlite` / `.db` sem instalar nada.

**Acesse online:** [thalysjuvenal.github.io/sdulite](https://thalysjuvenal.github.io/sdulite/)

---

## Como usar

### Abrindo um arquivo

1. Clique em **Abrir arquivo** no canto superior direito
2. Selecione um arquivo `.sqlite`, `.db` ou `.sqlite3`
3. As tabelas aparecem no painel lateral esquerdo — clique para navegar entre elas

Tambem e possivel **arrastar e soltar** o arquivo diretamente na area central.

### Editando dados

- **Clique duplo** em qualquer celula para editar o valor
- Celulas alteradas ficam destacadas em roxo ate serem salvas
- Um tooltip mostra o valor original ao passar o mouse sobre celulas editadas

### Salvando

Clique em **Salvar** para baixar o arquivo `.sqlite` com todas as alteracoes aplicadas.

---

## Funcionalidades

### Linhas

| Acao | Descricao |
|------|-----------|
| **+ Nova Linha** | Adiciona uma linha vazia no final da tabela |
| **Replicar** | Duplica as linhas selecionadas com incremento automatico do `R_E_C_N_O_` e campos configuráveis |
| **Deletar** | Remove as linhas selecionadas (com confirmacao) |

Para selecionar linhas, use os checkboxes na primeira coluna. `Shift+click` seleciona um intervalo.

### Busca e Navegacao

| Acao | Descricao |
|------|-----------|
| **Filtrar linhas** | Campo de texto acima da grid — filtra todas as colunas em tempo real |
| **Filtro por coluna** | Clique no icone de filtro no header de cada coluna para filtros avancados (=, !=, contem, comeca com, entre, nulo) |
| **Busca Global** | Pesquisa um valor em todas as tabelas do banco, mostrando onde ele aparece (tabela, coluna, linha) |
| **Favoritar** | Marca as linhas selecionadas como favoritas para acesso rapido |
| **Favoritos** | Exibe apenas as linhas marcadas como favoritas |

### Edicao em Massa

| Acao | Descricao |
|------|-----------|
| **Find & Replace** | Busca e substitui valores na coluna selecionada, com opcao de case-sensitive e regex |
| **Formula** | Aplica transformacoes em uma coluna inteira: `UPPER`, `LOWER`, `TRIM`, `REPLACE`, operacoes matematicas |
| **Templates** | Salva operacoes frequentes como templates reutilizaveis (ex: "Limpar campo X onde Y = Z") |

### Importacao e Exportacao

| Acao | Descricao |
|------|-----------|
| **Exportar CSV** | Exporta a tabela atual para arquivo `.csv` |
| **Importar CSV** | Importa dados de um arquivo `.csv` para a tabela atual, com mapeamento automatico de colunas |

### Seguranca

| Acao | Descricao |
|------|-----------|
| **Restaurar Original** | Reverte o banco inteiro para o estado original (backup feito automaticamente ao abrir) |
| **Historico** | Painel lateral com todas as alteracoes da sessao (tabela, coluna, valor antigo → novo), com opcao de reverter individualmente |
| **Somente Leitura** | Ativa/desativa o modo somente leitura para evitar edicoes acidentais |
| **Undo/Redo** | Desfazer e refazer alteracoes — contador visivel no rodape |

### Console SQL

Clique em **SQL** para abrir o console na parte inferior. Digite queries SQL livres e execute com **Ctrl+Enter** ou clicando em **Executar**.

O resultado aparece em uma grid temporaria abaixo do editor.

---

## Atalhos de teclado

| Atalho | Acao |
|--------|------|
| `Ctrl+Z` | Desfazer |
| `Ctrl+Y` | Refazer |
| `Ctrl+Enter` | Executar query SQL (quando o console esta aberto) |

---

## Stack tecnica

- **sql.js** v1.10.2 — SQLite compilado para WebAssembly
- **AG Grid Community** v31.3.2 — Grid de dados
- **PO UI** — Design system TOTVS (tema claro)
- HTML single-page, sem framework, sem build

---

## Executar localmente

Basta abrir o `index.html` no navegador ou servir com qualquer servidor HTTP:

```bash
python3 -m http.server 8080
```

Acesse `http://localhost:8080`.
