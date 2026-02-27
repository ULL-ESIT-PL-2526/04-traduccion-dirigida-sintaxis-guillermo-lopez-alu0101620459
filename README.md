# Procesadores de Lenguajes — Práctica de Laboratorio #4

**Universidad de La Laguna**  
**Nombre:** Guillermo López Concepción  
**Alu:** alu0101620459  

---

## 1. Configuración del proyecto

### 1.1. Instalar dependencias

```bash
npm i
```

### 1.2. Generar el parser con Jison

```bash
npx jison src/grammar.jison -o src/parser.js
```

### 1.3. Ejecutar las pruebas

```bash
npm test
```

Las pruebas están implementadas con **Jest** y configuradas en `package.json` mediante el atributo `"jest"` dentro de las dependencias de desarrollo.

---

## 2. Análisis del lexer

El bloque `%lex` del fichero `grammar.jison` define las reglas del analizador léxico:

```
%lex
%%
\s+        { /* skip whitespace */; }
[0-9]+     { return 'NUMBER'; }
"**"       { return 'OP'; }
[-+*/]     { return 'OP'; }
<<EOF>>    { return 'EOF'; }
.          { return 'INVALID'; }
/lex
```

### 3.1. Diferencia entre `/* skip whitespace */` y devolver un token

Cuando una regla ejecuta `/* skip whitespace */` (es decir, no hace `return`), el lexer consume los caracteres coincidentes y continúa leyendo la entrada sin notificar al parser. El espacio en blanco se ignora silenciosamente.

En cambio, cuando una regla ejecuta `return 'TOKEN'`, el lexer interrumpe el análisis léxico y entrega ese token al parser para que lo procese en la gramática. El parser necesita recibir tokens para avanzar en el análisis sintáctico.

**Resumen:** sin `return` → el lexer sigue sin informar al parser; con `return` → se entrega un token al parser.

### 3.2. Secuencia de tokens para la entrada `123**45+@`

| Texto consumido | Token devuelto |
|-----------------|----------------|
| `123`           | `NUMBER`       |
| `**`            | `OP`           |
| `45`            | `NUMBER`       |
| `+`             | `OP`           |
| `@`             | `INVALID`      |
| (fin de entrada)| `EOF`          |

Secuencia: `NUMBER`, `OP`, `NUMBER`, `OP`, `INVALID`, `EOF`.

### 3.3. Por qué `"**"` debe aparecer antes que `[-+*/]`

Los analizadores léxicos basados en Jison aplican las reglas **en orden de aparición** cuando hay más de una que podría coincidir con el mismo prefijo de la entrada. Si `[-+*/]` apareciese primero, al leer `**` coincidiría con el primer `*` y devolvería `OP`, dejando el segundo `*` como otro `OP` por separado. Al poner `"**"` antes, el lexer lo reconoce como una única cadena de dos caracteres y devuelve un solo token `OP` para el operador de potenciación.

### 3.4. Cuándo se devuelve `EOF`

La regla `<<EOF>>` se activa cuando el lexer ha consumido **toda la entrada** y no quedan más caracteres por leer. En ese momento devuelve el token `EOF`, que la gramática usa (en la producción `L → E eof`) para señalar que la expresión ha terminado y puede calcularse su valor final.

### 3.5. Por qué existe la regla `.` que devuelve `INVALID`

El patrón `.` coincide con **cualquier carácter** que no haya sido reconocido por ninguna regla anterior. Su propósito es el manejo de errores: sin esta regla, un carácter inesperado (como `@`, `#`, `$`, etc.) provocaría un fallo silencioso o un comportamiento indefinido. Devolviendo `INVALID`, el lexer informa al parser de que ha encontrado un carácter ilegal, lo que permite generar un mensaje de error controlado en lugar de un fallo abrupto del programa.

---

## 3. Modificación: ignorar comentarios de una línea (`//`)

Para que el lexer salte los comentarios que empiezan por `//` y se extienden hasta el final de la línea, se añade la siguiente regla **antes** de las demás (para que tenga mayor prioridad):

```
%lex
%%
\/\/[^\n]*  { /* skip line comment */; }
\s+         { /* skip whitespace */; }
[0-9]+      { return 'NUMBER'; }
"**"        { return 'OP'; }
[-+*/]      { return 'OP'; }
<<EOF>>     { return 'EOF'; }
.           { return 'INVALID'; }
/lex
```

La expresión regular `\/\/[^\n]*` reconoce `//` seguido de cualquier número de caracteres que no sean salto de línea, consumiendo así el comentario completo sin devolver ningún token.

---

## 4. Modificación: reconocer números en punto flotante

Para reconocer formatos como `2.35e-3`, `2.35e+3`, `2.35E-3`, `2.35` y `23`, se sustituye la regla `[0-9]+` por una expresión regular más completa:

```
%lex
%%
\/\/[^\n]*                      { /* skip line comment */; }
\s+                             { /* skip whitespace */; }
[0-9]+(\.[0-9]+)?([eE][+-]?[0-9]+)?  { return 'NUMBER'; }
"**"                            { return 'OP'; }
[-+*/]                          { return 'OP'; }
<<EOF>>                         { return 'EOF'; }
.                               { return 'INVALID'; }
/lex
```

La expresión `[0-9]+(\.[0-9]+)?([eE][+-]?[0-9]+)?` cubre:

- `23` → entero básico.
- `2.35` → número con parte decimal.
- `2.35e-3`, `2.35e+3`, `2.35E-3` → notación científica con mantisa decimal y exponente con signo opcional.

También habrá que actualizar la función `convert` en la SDD para usar `parseFloat` en lugar de `parseInt`, de modo que los valores en punto flotante se procesen correctamente:

```javascript
function convert(str) {
  return parseFloat(str);
}
```

---

## 5. Pruebas para las modificaciones del lexer

A continuación se muestran los casos de prueba añadidos en `__tests__/parser.test.js`:

```javascript
// Pruebas para comentarios de línea
describe('Comentarios de línea', () => {
  test('ignora un comentario al final de una expresión', () => {
    expect(parse('3 + 4 // esto es un comentario')).toBe(7);
  });

  test('ignora una línea completa de comentario antes de la expresión', () => {
    expect(parse('// comentario\n5 * 2')).toBe(10);
  });

  test('ignora múltiples comentarios', () => {
    expect(parse('1 + 2 // primer comentario\n// segundo comentario\n+ 3')).toBe(6);
  });
});

// Pruebas para números en punto flotante
describe('Números en punto flotante', () => {
  test('reconoce número decimal simple', () => {
    expect(parse('2.35')).toBeCloseTo(2.35);
  });

  test('reconoce notación científica con exponente negativo', () => {
    expect(parse('2.35e-3')).toBeCloseTo(0.00235);
  });

  test('reconoce notación científica con exponente positivo', () => {
    expect(parse('2.35e+3')).toBeCloseTo(2350);
  });

  test('reconoce notación científica con E mayúscula', () => {
    expect(parse('2.35E-3')).toBeCloseTo(0.00235);
  });

  test('opera correctamente con floats', () => {
    expect(parse('1.5 + 2.5')).toBeCloseTo(4.0);
  });

  test('opera con notación científica combinada', () => {
    expect(parse('1e2 + 50')).toBeCloseTo(150);
  });
});

// Prueba de carácter inválido
describe('Carácter inválido', () => {
  test('lanza error ante un carácter no reconocido', () => {
    expect(() => parse('3 + @')).toThrow();
  });
});
```