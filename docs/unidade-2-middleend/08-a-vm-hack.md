# 8. A VM Hack (especificação)

Este capítulo é a especificação de referência da VM Hack — a representação intermediária baseada em pilha para a qual o compilador Jack (capítulos 9 e 10) vai gerar código, e que o VM Translator (capítulos 11 e 12) vai traduzir para Assembly Hack. Não há código de compilador aqui; é a "planta" que orienta os próximos quatro capítulos.

## Modelo de memória

Cada célula de memória da máquina Hack armazena 16 bits, usados para representar de forma uniforme três coisas: um inteiro em complemento de dois (-32768 a 32767), um booleano (`0` para falso, `-1` — todos os bits em 1 — para verdadeiro), ou um endereço de memória.

A memória é organizada em **oito segmentos**, cada um endereçado por índice a partir de 0:

| Segmento | Propósito | Mapeado de |
|---|---|---|
| `local` | Variáveis locais da subroutine em execução | `var` (tabela de símbolos, capítulo 6) |
| `argument` | Argumentos recebidos pela subroutine | `arg` |
| `static` | Variáveis de classe (compartilhadas por todas as instâncias) | `static` |
| `this` | Campos do objeto atual | `field` (via `this`) |
| `that` | Um segundo "objeto atual" — usado para arrays e para o objeto de um método invocado sobre outra instância | (uso interno) |
| `constant` | Pseudo-segmento somente leitura para constantes literais | literais na expressão |
| `pointer` | Dois registradores (0 e 1) que apontam para `this` e `that`, respectivamente | uso interno do compilador |
| `temp` | Oito células de uso livre, compartilhadas por toda a VM | uso interno do compilador |

Além dos segmentos, duas áreas adicionais completam o modelo: a **pilha** (*stack*), usada para armazenar valores temporários durante a avaliação de expressões e para implementar o protocolo de chamada de função (capítulo 12); e o **heap**, a região de RAM dedicada ao armazenamento de objetos e arrays, gerenciada dinamicamente pelas funções `Memory.alloc`/`Memory.deAlloc` do sistema operacional Jack.

## Comandos de acesso à memória

Apenas dois comandos movem dados entre a pilha e os segmentos:

```
push segmento índice   // empilha o valor de segmento[índice]
pop segmento índice    // desempilha o topo e armazena em segmento[índice]
```

`segmento` é um dos oito acima (exceto que não existe `pop constant`, já que constantes não são graváveis); `índice` é um inteiro não negativo.

```
push local 2       // empilha local[2]
push constant 10   // empilha o literal 10
pop argument 0      // desempilha o topo para argument[0]
```

## Comandos aritméticos e lógicos

Nove comandos operam sobre a pilha — sete binários (consomem dois valores do topo, empilham um resultado) e dois unários (consomem um, empilham um):

| Comando | Aridade | Semântica |
|---|---|---|
| `add` | binário | soma |
| `sub` | binário | subtração |
| `neg` | unário | negação aritmética |
| `eq` | binário | `-1` se iguais, `0` caso contrário |
| `gt` | binário | `-1` se o primeiro é maior |
| `lt` | binário | `-1` se o primeiro é menor |
| `and` | binário | E bit a bit |
| `or` | binário | OU bit a bit |
| `not` | unário | complemento bit a bit |

Não existem comandos de multiplicação ou divisão na VM — como veremos no capítulo 9, o compilador Jack traduz os operadores `*` e `/` em chamadas às funções `Math.multiply`/`Math.divide` da biblioteca padrão, não em comandos VM nativos.

## Controle de fluxo

Linguagens de alto nível têm `if`, `while`, `for` — a VM reduz tudo isso a três primitivas:

```
label rótulo       // declara uma posição nomeada no código
goto rótulo        // salto incondicional
if-goto rótulo     // desempilha o topo; se for verdadeiro (≠ 0), salta
```

Rótulos têm escopo de função — dois `label LOOP` em funções diferentes não colidem (o capítulo 12 detalha a qualificação `funcao$rótulo` usada pelo VM Translator para garantir isso na tradução final para Assembly). É com apenas essas três primitivas que o capítulo 9 vai mapear `if`/`else` e `while` de Jack.

## Funções

Três comandos governam todo o protocolo de chamada:

```
function nome nLocais   // declara uma função, com N variáveis locais (inicializadas em 0)
call nome nArgs         // chama uma função, passando os N argumentos já empilhados
return                  // desempilha um valor e devolve o controle ao chamador
```

Uma peça importante do design: **mesmo subroutines que não retornam valor útil precisam empilhar algo antes do `return`** — por convenção, `push constant 0`. É simetria pura: todo `call` espera encontrar um valor no topo da pilha ao retornar, então toda função — `void` ou não — precisa deixar exatamente um valor lá.

```
function Main.dobro 0
    push constant 2
    push argument 0
    call Math.multiply 2
    return

function Main.main 1
    push constant 5
    call Main.dobro 1
    pop local 0
    push constant 0
    return
```

O nome de uma função é sempre `Classe.nomeDaSubroutine` — é assim que construtores, métodos e funções (na terminologia Jack) se tornam, todos igualmente, "funções" no nível da VM. O mecanismo completo de `call`/`return` (salvar e restaurar o estado do chamador, o *stack frame*) é o assunto do capítulo 12; por ora, basta saber a interface: argumentos entram empilhados, o resultado sai empilhado.

## Um programa completo

Todo programa VM começa executando `Main.main`. Um exemplo simples — somar de 1 a `n` e imprimir o resultado — ilustra push/pop, aritmética, controle de fluxo e chamada de função juntos:

```
function Soma.soma 2        // variáveis locais: sum (0), i (1)
    push constant 0
    pop local 0              // sum = 0
    push constant 1
    pop local 1              // i = 1
label LOOP
    push local 1
    push argument 0
    gt                        // i > n ?
    if-goto FIM
    push local 0
    push local 1
    add
    pop local 0               // sum = sum + i
    push local 1
    push constant 1
    add
    pop local 1               // i = i + 1
    goto LOOP
label FIM
    push local 0
    return

function Main.main 1
    push constant 10
    pop local 0               // n = 10
    push local 0
    call Soma.soma 1
    call Output.printInt 1
    push constant 0
    return
```

## O que vem a seguir

Com a especificação da VM Hack em mãos, os próximos dois capítulos (9 e 10) evoluem o `CompilationEngine` da Unidade 1 para efetivamente gerar este código, expressão por expressão, comando por comando — fechando o compilador Jack → VM (Project 11) por completo. Os capítulos 11 e 12, na Unidade 3, tratam do outro lado: como traduzir esse código VM para Assembly Hack.

Uma versão tabular desta especificação, para consulta rápida durante esses capítulos, está no [Apêndice B](../apendices/b-especificacao-vm.md).
