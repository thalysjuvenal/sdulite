# RAD APSDU v0.03 — Design de Melhorias

## Contexto

Editor SQLite no browser (single-page HTML) para **consultores Protheus** que corrigem dados em bases SQLite exportadas.

- **Usuarios:** Consultores TOTVS Protheus
- **Cenario:** Correcao de dados em bases SQLite exportadas
- **Volume:** Variavel, deve suportar o maximo possivel
- **Interface:** Visual para dia a dia + Console SQL para casos avancados

## Dores Identificadas

1. **Busca/Navegacao** — Consultores perdem tempo localizando registros
2. **Seguranca** — Medo de errar sem conseguir reverter, causando retrabalho
3. **Produtividade em Massa** — Operacoes repetitivas consomem a maior parte do tempo

---

## Eixo 1 — Busca e Navegacao

### 1. Filtro avancado por coluna
Clicar no header da coluna abre filtro com operadores: =, !=, contem, comeca com, entre, e nulo/nao nulo.

### 2. Console SQL
Painel inferior retratil onde o consultor digita queries SQL livres e ve o resultado em uma grid temporaria, com botao para "aplicar" os resultados como edicao.

### 3. Busca global cross-tabela
Campo de busca que procura um valor em todas as tabelas do banco, mostrando onde aparece (tabela + coluna + linha).

### 4. Paginacao virtual
Substituir o LIMIT 5000 por scroll infinito com carregamento sob demanda, suportando tabelas grandes.

### 5. Favoritos/Marcadores
Possibilidade de "marcar" linhas especificas para voltar a elas rapidamente durante a sessao.

---

## Eixo 2 — Seguranca e Reversibilidade

### 6. Backup automatico
Ao abrir um arquivo, salvar uma copia original em memoria (ou download automatico de backup) antes de qualquer edicao.

### 7. Historico de alteracoes
Painel lateral que mostra todas as alteracoes feitas na sessao (tabela, coluna, valor antigo -> novo, timestamp), com possibilidade de reverter individualmente.

### 8. Undo/Redo ilimitado
Expandir o undo/redo atual para persistir todo o historico da sessao, com navegacao visual (timeline).

### 9. Confirmacao antes de salvar
Modal de revisao que lista todas as alteracoes pendentes antes de gravar, permitindo descartar alteracoes especificas.

### 10. Diff visual
Celulas alteradas ficam destacadas com cor diferente e tooltip mostrando o valor original, ate que o arquivo seja salvo.

### 11. Modo somente leitura
Toggle para abrir o banco em modo read-only, evitando edicoes acidentais enquanto o consultor esta apenas consultando.

---

## Eixo 3 — Produtividade em Massa

### 12. Edicao em lote por selecao
Selecionar multiplas celulas de uma mesma coluna e atribuir um valor de uma vez (alem do Find & Replace atual, que exige abrir modal).

### 13. Importar dados de Excel/CSV
Colar ou importar dados de planilhas externas diretamente em uma tabela, mapeando colunas automaticamente.

### 14. Exportar para Excel/CSV
Exportar a tabela atual (ou resultado de query SQL) para CSV/Excel para conferencia ou envio.

### 15. Templates de operacao
Salvar operacoes comuns como templates reutilizaveis (ex: "Limpar campo X onde Y = Z") que podem ser re-executados com um clique.

### 16. Auto-incremento inteligente
Ao replicar linhas, permitir definir regras de incremento para campos alem do R_E_C_N_O_ (ex: codigos sequenciais, datas incrementais).

### 17. Edicao via formula
Aplicar transformacoes em coluna inteira (UPPER, LOWER, TRIM, REPLACE, concatenar, operacoes matematicas) sem precisar escrever SQL.

---

## Stack Tecnica

- HTML single-page (sem framework/build)
- sql.js (WebAssembly SQLite)
- AG Grid Community
- CSS custom properties (tema escuro)
