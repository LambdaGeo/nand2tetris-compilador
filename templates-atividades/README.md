# Templates de atividades práticas

Enunciados das atividades práticas da disciplina, mantidos **fora do livro** (`docs/`) de
propósito: são material de logística de turma, que muda a cada oferta, enquanto o livro é
conteúdo técnico estável e reutilizável por qualquer professor.

O fluxo de uso é: copiar o arquivo para o Notion (ou para o AVA da oferta), substituir os
placeholders, ajustar o que for específico do semestre e publicar para a turma.

## Placeholders

Todo campo que muda a cada oferta está marcado com `{{CHAVE}}`. Antes de publicar, substitua:

| Placeholder | Exemplo | Onde aparece |
|---|---|---|
| `{{SEMESTRE}}` | `2026.2` | cabeçalho de todos os enunciados |
| `{{DATA_ENTREGA}}` | `07/abril` | cabeçalho de todos os enunciados |
| `{{PLATAFORMA_ENTREGA}}` | `SIGAA` | seção de entrega |
| `{{TAMANHO_GRUPO}}` | `2 integrantes` | seção de equipe |
| `{{MIN_COMMITS}}` | `10` | entregáveis |
| `{{LINK_TURMA}}` | link do Notion/AVA da oferta | materiais de apoio |

Grep rápido para conferir se sobrou algum: `grep -rn '{{' .`

## Arquivos

| # | Arquivo | Projeto nand2tetris | Capítulos do livro |
|---|---|---|---|
| 1 | `01-analisador-lexico.md` | Project 10 (parte 1) | 1–4 |
| 2 | `02-analisador-sintatico.md` | Project 10 (parte 2) | 3, 5 |
| 3 | `03-gerador-codigo-vm.md` | Project 11 | 6, 8, 9, 10 |
| 4 | `04-vm-translator.md` | Projects 7 e 8 | 8, 11, 12 |
| 5 | `05-assembler.md` | Project 6 | 14, 15 |

A ordem acima segue a ordem do **livro** (front-end → middle-end → back-end), que é
deliberadamente diferente da numeração dos projetos do nand2tetris (que vai bottom-up,
do hardware para a linguagem). Se a sua oferta seguir a ordem original do nand2tetris,
basta reordenar as entregas — os enunciados são independentes entre si, exceto pelo
encadeamento 1 → 2 → 3, que reutiliza o mesmo repositório.

## Convenções adotadas nestes templates

- **Links para o livro**, e não para páginas do Notion: os enunciados apontam para os
  capítulos publicados em `docs/`, que são a fonte canônica do conteúdo técnico. Isso
  evita os links relativos quebrados que existiam nas versões exportadas do Notion.
- **Arquivos de teste sempre da fonte oficial** (`nand2tetris.org/software`), em vez de
  links de Google Drive, que expiram e não são reproduzíveis por outras turmas.
- **Critérios de avaliação como sugestão**, não como regra fixa — cada oferta calibra
  pesos, penalidades e obrigatoriedade de vídeo conforme a carga horária disponível.

## Nota sobre o VM Translator

O template `04-vm-translator.md` cobre as **duas partes** (Projects 7 e 8) num enunciado
só, com a Parte 2 marcada como seção separada. A versão original em Notion só tinha a
Parte 1; se a sua oferta dividir em duas entregas, corte o arquivo na marca indicada.
