# Apêndice B — Especificação da VM Hack

Referência compacta, extraída do capítulo 8, para consulta rápida durante os capítulos 9–12.

## Segmentos

| Segmento | Propósito |
|---|---|
| `local` | Variáveis locais da subroutine em execução |
| `argument` | Argumentos recebidos pela subroutine |
| `static` | Variáveis de classe, compartilhadas por todas as instâncias |
| `this` | Campos do objeto atual |
| `that` | Segundo "objeto atual" — usado por arrays e por métodos de outros objetos |
| `constant` | Pseudo-segmento somente leitura, para literais |
| `pointer` | 2 células: `pointer 0` = `this`, `pointer 1` = `that` |
| `temp` | 8 células de uso livre |

## Comandos de acesso à memória

```
push segmento índice
pop segmento índice
```

## Comandos aritméticos e lógicos

| Comando | Aridade | Semântica |
|---|---|---|
| `add` | binário | soma |
| `sub` | binário | subtração |
| `neg` | unário | negação |
| `eq` | binário | -1 se iguais, 0 caso contrário |
| `gt` | binário | -1 se o primeiro é maior |
| `lt` | binário | -1 se o primeiro é menor |
| `and` | binário | E bit a bit |
| `or` | binário | OU bit a bit |
| `not` | unário | complemento bit a bit |

## Controle de fluxo

```
label rótulo
goto rótulo
if-goto rótulo     // desempilha o topo; salta se ≠ 0
```

## Funções

```
function nome nLocais
call nome nArgs
return
```

Convenção: toda subroutine — `void` ou não — deixa exatamente um valor na pilha antes do `return` (por convenção, `push constant 0` para as que não têm valor útil).
