# Laboratorio 1: Introducción a ANTLR

Repositorio del laboratorio individual de introducción a ANTLR (curso Compiladores 2026).

## Contenido

- `program/MiniLang.g4` — Gramática ANTLR de MiniLang (expresiones aritméticas y asignaciones).
- `program/Driver.py` — Punto de entrada que invoca al parser generado sobre un archivo de entrada.
- `program/program_test.txt` — Entrada de prueba **válida**.
- `program/program_test_invalid.txt` — Entrada de prueba **inválida**, usada para mostrar el reporte de errores de ANTLR.
- `Dockerfile`, `commands/`, `antlr-4.13.2-complete.jar`, `python-venv.sh`, `requirements.txt` — Entorno de ejecución (provisto por la cátedra).
- `ANALISIS.md` — Análisis de la gramática y del driver (entregable).

## Cómo ejecutar

1. Construir y levantar el contenedor:

   ```bash
   docker build --rm . -t lab1-image && docker run --rm -ti -v "$(pwd)/program":/program lab1-image
   ```

2. Dentro del contenedor, generar el lexer/parser:

   ```bash
   antlr -Dlanguage=Python3 MiniLang.g4
   ```

3. Ejecutar el analizador contra la entrada válida (no debería imprimir nada):

   ```bash
   python3 Driver.py program_test.txt
   ```

4. Ejecutar el analizador contra la entrada inválida (debería imprimir errores de sintaxis):

   ```bash
   python3 Driver.py program_test_invalid.txt
   ```

Ver `ANALISIS.md` para la explicación de la gramática, el driver, y los resultados de estas pruebas.

## Video

Video de YouTube (no listado) con las pruebas y comentarios: _pendiente, agregar enlace aquí_.
