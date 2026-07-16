# Análisis: Gramática ANTLR y Driver.py

## 1. Estructura general de un archivo `.g4`

Un archivo `.g4` de ANTLR se compone típicamente de estas secciones:

1. **Declaración de la gramática** (`grammar MiniLang;`): define el nombre de la gramática. Este nombre debe coincidir con el nombre del archivo (`MiniLang.g4`), y ANTLR lo usa como prefijo para las clases generadas (`MiniLangLexer`, `MiniLangParser`, `MiniLangListener`, etc.).
2. **Reglas del parser** (en minúscula, ej. `prog`, `stat`, `expr`): definen la estructura sintáctica del lenguaje, es decir, cómo se combinan los tokens para formar construcciones válidas.
3. **Reglas del lexer** (en MAYÚSCULA, ej. `MUL`, `ID`, `INT`): definen cómo se agrupan los caracteres de entrada en *tokens* (unidades léxicas). ANTLR combina automáticamente todas las reglas en mayúscula para generar el lexer.

En `MiniLang.g4` esto se ve así:

```antlr
grammar MiniLang;

prog:   stat+ ;
...
MUL : '*' ;
...
```

## 2. Reglas del parser

```antlr
prog:   stat+ ;
```
`prog` es la **regla de entrada** (start rule): un programa es una o más `stat` (`+` significa "una o más veces", igual que en expresiones regulares).

```antlr
stat:   expr NEWLINE                 # printExpr
    |   ID '=' expr NEWLINE          # assign
    |   NEWLINE                      # blank
    ;
```
Una `stat` (sentencia) puede ser:
- Una expresión seguida de salto de línea (se interpreta como "imprimir expresión").
- Una asignación `ID = expr`.
- Una línea en blanco (solo un `NEWLINE`).

Cada alternativa (separada por `|`) tiene una **etiqueta** después de `#` (`printExpr`, `assign`, `blank`). Esto le indica a ANTLR que genere un método distinto en el `Visitor`/`Listener` para cada alternativa (por ejemplo `visitPrintExpr`, `visitAssign`, `visitBlank`) en lugar de un único método genérico `visitStat`. Esto es muy útil cuando una misma regla agrupa casos semánticamente distintos, ya que evita tener que hacer `instanceof`/casting manual dentro de un solo método para diferenciar los casos.

```antlr
expr:   expr ('*'|'/') expr          # MulDiv
    |   expr ('+'|'-') expr          # AddSub
    |   INT                          # int
    |   ID                           # id
    |   '(' expr ')'                 # parens
    ;
```
`expr` es una regla **recursiva a la izquierda** (`expr` se referencia a sí misma al inicio de la alternativa). ANTLR soporta esto de forma nativa y lo resuelve automáticamente aplicando las reglas de **precedencia por orden de declaración**: las alternativas declaradas primero tienen mayor precedencia. Por eso `MulDiv` (multiplicación/división) se evalúa antes que `AddSub` (suma/resta), replicando la precedencia matemática estándar, sin necesidad de escribir la gramática en la forma clásica no-recurrente-a-la-izquierda (`expr -> term (('+'|'-') term)*`).

Las etiquetas (`# MulDiv`, `# AddSub`, `# int`, `# id`, `# parens`) cumplen el mismo propósito que en `stat`: permiten generar un manejador específico por cada forma que puede tomar una expresión.

## 3. Reglas del lexer

```antlr
MUL : '*' ;
DIV : '/' ;
ADD : '+' ;
SUB : '-' ;
ID  : [a-zA-Z]+ ;
INT : [0-9]+ ;
NEWLINE:'\r'? '\n' ;
WS  : [ \t]+ -> skip ;
```

- `MUL`, `DIV`, `ADD`, `SUB`: tokens literales de un solo carácter, usados como operadores.
- `ID`: uno o más caracteres alfabéticos (`[a-zA-Z]+`); representa nombres de variables.
- `INT`: uno o más dígitos (`[0-9]+`); representa literales numéricos.
- `NEWLINE`: un salto de línea opcionalmente precedido de retorno de carro (soporta finales de línea estilo Windows `\r\n` y Unix `\n`). Este token es significativo para el parser porque actúa como **terminador de sentencia** (equivalente al `;` en otros lenguajes).
- `WS`: espacios y tabulaciones. La acción `-> skip` le indica al lexer que **descarte** estos tokens en vez de enviarlos al parser; de lo contrario tendríamos que contemplar espacios en blanco explícitamente en cada regla del parser.

El **orden** de las reglas del lexer importa: cuando dos reglas podrían coincidir con el mismo texto, ANTLR usa la que fue declarada primero. En este caso no hay ambigüedad real entre ellas, pero es una regla general a tener en cuenta al extender la gramática.

## 4. ¿Qué genera el comando `antlr`?

```bash
antlr -Dlanguage=Python3 MiniLang.g4
```

Este comando (que en el contenedor es un wrapper de `java -jar antlr-4.13.2-complete.jar ...`) toma `MiniLang.g4` y genera, en Python3:

- `MiniLangLexer.py`: el analizador léxico, que convierte el texto de entrada en una secuencia de tokens.
- `MiniLangParser.py`: el analizador sintáctico, que consume esos tokens y construye un árbol de sintaxis (parse tree) siguiendo las reglas gramaticales.
- `MiniLangListener.py`: una clase base con un método `enterX`/`exitX` (y `enter<Etiqueta>`/`exit<Etiqueta>` para las alternativas etiquetadas) por cada regla, pensada para el patrón *Listener* (recorrido pasivo del árbol).
- Archivos auxiliares `.tokens` / `.interp`: metadata usada internamente por ANTLR (mapeo de tokens, tablas de estado del ATN, etc.), no se editan a mano.

No se genera un `Visitor` porque esa opción no fue habilitada explícitamente (`-visitor`); por defecto ANTLR en Python3 solo genera el `Listener`.

## 5. Análisis de `Driver.py`

```python
import sys
from antlr4 import *
from MiniLangLexer import MiniLangLexer
from MiniLangParser import MiniLangParser

def main(argv):
    input_stream = FileStream(argv[1])
    lexer = MiniLangLexer(input_stream)
    stream = CommonTokenStream(lexer)
    parser = MiniLangParser(stream)
    tree = parser.prog()

if __name__ == '__main__':
    main(sys.argv)
```

Este es el flujo estándar para invocar un analizador generado por ANTLR:

1. `FileStream(argv[1])`: lee el archivo de entrada (pasado como argumento de línea de comandos) como un stream de caracteres.
2. `MiniLangLexer(input_stream)`: instancia el lexer generado, que consumirá el stream de caracteres y producirá tokens.
3. `CommonTokenStream(lexer)`: envuelve al lexer en un stream de tokens; el parser consume tokens de aquí (y esta capa es la que permite, por ejemplo, "mirar hacia adelante" varios tokens sin recorrer directamente el lexer).
4. `MiniLangParser(stream)`: instancia el parser generado sobre ese stream de tokens.
5. `parser.prog()`: invoca la **regla de entrada** de la gramática (la primera regla del parser, `prog`). Esto dispara el proceso de parsing completo y retorna el árbol de sintaxis (`ParseTree`) resultante.

El driver no hace nada con `tree` (no imprime, no recorre, no valida semántica); su único propósito en este laboratorio es forzar la ejecución del parser para observar si ANTLR reporta errores sintácticos. Por eso:
- Si la entrada es válida, no hay salida (el árbol se construye en silencio).
- Si hay errores léxicos o sintácticos, el `ErrorListener` por defecto de ANTLR los imprime en `stderr` con el formato `line X:Y <mensaje>`, sin detener la ejecución (ANTLR intenta recuperarse del error y seguir parseando el resto del archivo, un enfoque conocido como *error recovery*).

## 6. Pruebas realizadas

- **Entrada válida** (`program_test.txt`): sin salida, código de salida `0`.
  ```
  5 * 5
  a = 4
  b = 6
  c = a + b
  ```
- **Entrada inválida** (`program_test_invalid.txt`), con una expresión incompleta y un paréntesis sin cerrar:
  ```
  5 * 5
  a = 4
  b = 6
  c = a +
  d = (1 + 2
  ```
  Salida de ANTLR:
  ```
  line 4:7 extraneous input '\n' expecting {'(', ID, INT}
  line 5:2 mismatched input '=' expecting NEWLINE
  line 5:10 missing ')' at '\n'
  ```
  Se observa el mecanismo de recuperación de errores de ANTLR: en vez de detenerse en el primer error (línea 4), continúa intentando parsear e identifica un segundo error en la línea 5, e incluso reporta un paréntesis faltante al llegar al final de esa sentencia.
