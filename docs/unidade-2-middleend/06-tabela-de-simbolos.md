# 6. Tabela de símbolos

Antes de gerar qualquer código, o compilador precisa responder a uma pergunta simples toda vez que encontra um identificador: *o que é isso?* Uma variável local? Um campo do objeto? Um parâmetro? De que tipo? Em que posição, dentro do seu segmento, ela vive? Este capítulo constrói a estrutura de dados que responde a essas perguntas — a **tabela de símbolos** — e a integra ao `CompilationEngine` da Unidade 1, fechando a lacuna de análise semântica que faltava para começar a gerar código de verdade nos capítulos 9 e 10.

## Por que uma tabela de símbolos

Um programa de alto nível introduz e manipula muitos identificadores. Toda vez que o compilador encontra um identificador `xxx`, ele precisa saber: é um nome de variável, de classe, de função? Se é variável, é campo de um objeto ou argumento de uma subroutine? Qual seu tipo — `int`, `boolean`, `char`, ou uma classe? Essas perguntas precisam ser respondidas **toda vez** que `xxx` aparece no código-fonte — não apenas na primeira. A tabela de símbolos existe exatamente para isso: sempre que um identificador é declarado, o compilador registra sua descrição; sempre que ele é referenciado, o compilador consulta a tabela e obtém tudo que precisa para gerar o código correspondente.

## As quatro categorias de variável em Jack

Jack tem exatamente quatro categorias (`kind`) de variável, cada uma com um tempo de vida e um segmento de memória VM associado (o capítulo 8 detalha a VM Hack; por ora, note apenas a categoria):

| Categoria (`kind`) | Onde é declarada | Tempo de vida | Segmento VM |
|---|---|---|---|
| `static` | Início da classe, com `static` | Todo o programa; compartilhada por todas as instâncias | `static` |
| `field` | Início da classe, com `field` | Todo o objeto; uma cópia por instância | `this` |
| `arg` | Lista de parâmetros da subroutine | Duração da chamada | `argument` |
| `var` | Início da subroutine, com `var` | Duração da chamada | `local` |

Como `static`/`field` vivem no nível da **classe** e `arg`/`var` vivem no nível da **subroutine**, o compilador precisa manter, a qualquer momento, exatamente **duas** tabelas de símbolos ativas: uma de classe (que persiste durante toda a compilação de uma classe) e uma de subroutine (que é **limpa** toda vez que uma nova subroutine começa a ser compilada).

## API da tabela de símbolos

A tabela expõe uma API pequena e direta:

| Método | Papel |
|---|---|
| `startSubroutine()` | Limpa a tabela de subroutine (chamado ao entrar em cada `constructor`/`function`/`method`) |
| `define(name, type, kind)` | Registra um novo identificador na tabela apropriada (classe se `kind` for `static`/`field`; subroutine se for `arg`/`var`) |
| `varCount(kind)` | Quantas variáveis de uma dada categoria já foram definidas no escopo atual |
| `resolve(name)` | Busca um identificador, devolvendo categoria, tipo e índice |

O índice devolvido por `resolve` (e atribuído automaticamente por `define`) é sempre relativo à sua própria categoria: a primeira `field` é índice 0, a segunda `field` é índice 1, mesmo que entre elas haja uma `static` (que tem sua própria contagem independente). É exatamente esse índice — categoria + posição — que vira o segmento e o deslocamento de um comando `push`/`pop` da VM.

```java
public class SymbolTable {
    public enum Kind { STATIC, FIELD, ARG, VAR }

    public record Symbol(String name, String type, Kind kind, int index) {}

    private Map<String, Symbol> classScope = new HashMap<>();
    private Map<String, Symbol> subroutineScope = new HashMap<>();
    private Map<Kind, Integer> counts = new HashMap<>();

    public SymbolTable() {
        for (Kind k : Kind.values()) counts.put(k, 0);
    }

    public void startSubroutine() {
        subroutineScope.clear();
        counts.put(Kind.ARG, 0);
        counts.put(Kind.VAR, 0);
    }

    private Map<String, Symbol> scopeFor(Kind kind) {
        return (kind == Kind.STATIC || kind == Kind.FIELD) ? classScope : subroutineScope;
    }

    public void define(String name, String type, Kind kind) {
        var scope = scopeFor(kind);
        int index = varCount(kind);
        scope.put(name, new Symbol(name, type, kind, index));
        counts.put(kind, index + 1);
    }

    public int varCount(Kind kind) {
        return counts.get(kind);
    }

    public Symbol resolve(String name) {
        Symbol s = subroutineScope.get(name);
        return s != null ? s : classScope.get(name);
    }
}
```

Repare em `resolve`: a busca acontece **primeiro** no escopo de subroutine e só depois no de classe — exatamente a regra de sombreamento (*shadowing*) que se espera de escopos aninhados: uma variável local com o mesmo nome de um campo da classe deve "ganhar" dentro daquela subroutine.

## Integrando ao parser

A tabela se popula em exatamente três pontos da gramática — todos já reconhecidos pelo `CompilationEngine` da Unidade 1, agora ganhando uma linha de `symbolTable.define(...)` cada:

**Variáveis de classe** (`classVarDec`, dentro de `parseClass`):

```java
void parseClassVarDec() {
    expectPeek(FIELD, STATIC);
    Kind kind = currentTokenIs(FIELD) ? Kind.FIELD : Kind.STATIC;

    expectPeek(INT, CHAR, BOOLEAN, IDENT);
    String type = currentToken.lexeme;

    expectPeek(IDENT);
    symbolTable.define(currentToken.lexeme, type, kind);
    while (peekTokenIs(COMMA)) {
        expectPeek(COMMA);
        expectPeek(IDENT);
        symbolTable.define(currentToken.lexeme, type, kind);
    }
    expectPeek(SEMICOLON);
}
```

**Parâmetros** (`parameterList`, dentro de `parseSubroutineDec`) — sempre `Kind.ARG`:

```java
void parseParameterList() {
    if (!peekTokenIs(RPAREN)) {
        expectPeek(INT, CHAR, BOOLEAN, IDENT);
        String type = currentToken.lexeme;
        expectPeek(IDENT);
        symbolTable.define(currentToken.lexeme, type, Kind.ARG);
        while (peekTokenIs(COMMA)) {
            expectPeek(COMMA);
            expectPeek(INT, CHAR, BOOLEAN, IDENT);
            type = currentToken.lexeme;
            expectPeek(IDENT);
            symbolTable.define(currentToken.lexeme, type, Kind.ARG);
        }
    }
}
```

**Variáveis locais** (`varDec`, dentro de `parseSubroutineBody`) — sempre `Kind.VAR`, seguindo exatamente o mesmo padrão de `classVarDec`, trocando apenas a categoria.

E, no início de cada `subroutineDec`, dois ajustes indispensáveis:

```java
void parseSubroutineDec() {
    symbolTable.startSubroutine();

    expectPeek(CONSTRUCTOR, FUNCTION, METHOD);
    TokenType subroutineType = currentToken.type;

    if (subroutineType == METHOD) {
        symbolTable.define("this", className, Kind.ARG);
    }
    // ...
}
```

O truque de `if (subroutineType == METHOD) symbolTable.define("this", className, Kind.ARG)` merece destaque: em Jack, um método é sempre chamado com o objeto receptor como **primeiro argumento implícito** (o compilador insere isso na chamada, como veremos no capítulo 10) — logo, registrá-lo como `arg` de índice 0 *antes* de processar a lista de parâmetros declarados garante que o primeiro parâmetro *de fato* declarado pelo programador já saia com índice 1, exatamente onde ele estará em tempo de execução.

## O que a tabela ainda não faz

Esta tabela responde "que categoria, que tipo, que índice" — o suficiente para gerar `push`/`pop` corretos nos capítulos 9 e 10. Ela **não** faz verificação de tipos (não impede que você some um `int` com um `boolean`), nem detecta redeclaração além do básico, nem resolve sobrecarga (que nem existe em Jack). Esse é o escopo deliberadamente mínimo de análise semântica deste curso — o suficiente para gerar código correto, sem se tornar um verificador de tipos completo.

## O que vem a seguir

Com a tabela de símbolos pronta, o próximo capítulo dá um passo atrás para revisar, em teoria geral, o papel de uma representação intermediária — preparando o terreno para a especificação da VM Hack (capítulo 8) e, finalmente, para a geração de código de verdade (capítulos 9 e 10), agora que sabemos exatamente onde cada identificador vive.
