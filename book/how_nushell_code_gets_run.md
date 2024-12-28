# Cómo se ejecuta el código Nushell

En [Pensando en Nu](./thinking_in_nu.md#think-of-nushell-as-a-compiled-language), te animamos a _"pensar en Nushell como un lenguaje compilado"_ debido a cómo se procesa el código de Nushell. También cubrimos varios ejemplos de código que no funcionan en Nushell debido a ese proceso.

La razón subyacente es una estricta separación de las etapas de **_análisis y evaluación (parsing and evaluation)_**, lo que **_impide la funcionalidad tipo `eval`_**. En esta sección, explicaremos en detalle qué significa esto, por qué lo hacemos y cuáles son las implicaciones. La explicación busca ser lo más sencilla posible, pero podría ayudarte haber programado en otro lenguaje antes.

[[toc]]

## Lenguajes interpretados vs compilados

### Lenguajes interpretados

Nushell, Python y Bash (entre muchos otros) son lenguajes _"interpretados"_.

Comencemos con un simple programa de Nushell que muestra "Hello, World!" (¡Hola, Mundo!):

```nu
# hello.nu

print "Hello, World!"
```

Por supuesto, esto se ejecuta como se espera usando `nu hello.nu`. Un programa similar escrito en Python o Bash se vería (y se comportaría) casi igual.

En lenguajes _"interpretados"_ el código generalmente se maneja de la siguiente manera:

```
Código fuente (Source Code) → Intérprete (Interpreter) → Resultado (Result)
```

Nushell sigue este patrón, y su "intérprete" se divide en dos partes:

1. `Código fuente → Analizador (Parser) → Representación Intermedia (Intermediate Representations - IR)`
2. `IR → Motor de evaluación (Evaluation Engine) → Resultado`

Primero, el código fuente es analizado por el Parser y convertido en una representación intermedia (IR), que en el caso de Nushell es simplemente una colección de estructuras de datos. Luego, estas estructuras se pasan al motor (Engine) para su evaluación y salida de resultados.

Esto, también, es común en los lenguajes interpretados. Por ejemplo, el código fuente de Python generalmente se [convierte en bytecode](https://github.com/python/cpython/blob/main/InternalDocs/interpreter.md) antes de la evaluación.

### Lenguajes compilados

Por otro lado,  están los lenguajes típicamente «compilados», como C, C++ o Rust. Por ejemplo, aquí hay un simple _"Hello, World!"_ en Rust:

```rust
// main.rs

fn main() {
    println!("Hello, World!");
}
```

Para "ejecutar" este código, debe ser:

1. Compilado en [instrucciones de código máquina](https://en.wikipedia.org/wiki/Machine_code)
2. Los resultados de la compilación almacenados como un archivo binario en el disco

Los primeros dos pasos se manejan con `rustc main.rs`.

3. Luego, para producir un resultado, necesitas ejecutar el binario (`./main`), que pasa las instrucciones a la CPU.

Entonces:

1. `Código fuente ⇒ Compilador ⇒ Código máquina` (Source Code ⇒ Compiler ⇒ Machine Code)
2. `Código máquina ⇒ CPU ⇒ Resultado` (Machine Code ⇒ CPU ⇒ Result)

::: important
You can see that the compile-run sequence is not much different from the parse-evaluate sequence of an interpreter. You begin with source code, parse (or compile) it into some state (e.g., bytecode, IR, machine code), then evaluate (or run) the IR to get a result. You could think of machine code as just another type of IR and the CPU as its interpreter.
Puedes ver que la secuencia compilar-ejecutar (compile-run) no es muy diferente de la secuencia analizar-evaluar (parse-evaluate) de un intérprete. Comienzas con código fuente, lo analizas "parseas" (o compilas) en algún estado (p. ej., bytecode, IR, código máquina), luego evalúas (o ejecutas) el IR para obtener un resultado. Podrías pensar en el código máquina como otro tipo de IR y en la CPU como su intérprete.

Una gran diferencia, sin embargo, entre los lenguajes interpretados y compilados es que los lenguajes interpretados generalmente implementan una función _`eval`_, mientras que los lenguajes compilados no. ¿Qué significa esto?
:::

## Lenguajes dinámicos vs estáticos

::: tip Terminología
En general, la diferencia entre un lenguaje dinámico y uno estático es cuánta parte del código fuente se resuelve durante la compilación (o análisis -parsing-) vs. la evaluación/ejecución (Evaluation/Runtime):

- Los lenguajes _"estáticos"_ realizan más análisis de código (p. ej., verificación de tipos (type-checking), [propiedad de los datos (data ownership)](https://doc.rust-lang.org/stable/book/ch04-00-understanding-ownership.html)) durante la compilación/análisis (Compilation/Parsing).

- Los lenguajes _"dinámicos"_ realizan más análisis de código, incluyendo `eval` de código adicional, durante la evaluación/ejecución (Evaluation/Runtime).

Para los propósitos de esta discusión, la principal diferencia entre un lenguaje estático y dinámico es si tiene o no una función `eval`.

:::

### Función Eval

La mayoría de los lenguajes dinámicos e interpretados tienen una función `eval`. Por ejemplo, [Python `eval`](https://docs.python.org/3/library/functions.html#eval) (también, [Python `exec`](https://docs.python.org/3/library/functions.html#exec)) o [Bash `eval`](https://linux.die.net/man/1/bash).

;;;;;;;;;;CONSULTAR;;;;;;;;;;;;;;;;
The argument to an `eval` is _"source code inside of source code"_, typically conditionally or dynamically computed. This means that, when an interpreted language encounters an `eval` in source code during Parse/Eval, it typically interrupts the normal Evaluation process to start a new Parse/Eval on the source code argument to the `eval`.
El argumento de un `eval` es _«código fuente dentro de código fuente»_, normalmente ejecutado de forma condicional o dinámica. Esto significa que, cuando un lenguaje interpretado encuentra un `eval` en el código fuente durante Parse/Eval (analizar/evaluar), normalmente interrumpe el proceso normal de Evaluación para iniciar un nuevo Parse/Eval en el argumento de código fuente del `eval`.
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
Here's a simple Python `eval` example to demonstrate this (potentially confusing!) concept:

```python:line-numbers
# hello_eval.py

print("Hello, World!")
eval("print('Hello, Eval!')")
```

When you run the file (`python hello_eval.py`), you'll see two messages: _"Hello, World!"_ and _"Hello, Eval!"_. Here is what happens:

1. The entire program is Parsed
2. (Line 3) `print("Hello, World!")` is Evaluated
3. (Line 4) In order to evaluate `eval("print('Hello, Eval!')")`:
   1. `print('Hello, Eval!')` is Parsed
   2. `print('Hello, Eval!')` is Evaluated

::: tip More fun
Consider `eval("eval(\"print('Hello, Eval!')\")")` and so on!
:::

Notice how the use of `eval` here adds a new "meta" step into the execution process. Instead of a single Parse/Eval, the `eval` creates additional, "recursive" Parse/Eval steps instead. This means that the bytecode produced by the Python interpreter can be further modified during the evaluation.

Nushell does not allow this.

As mentioned above, without an `eval` function to modify the bytecode during the interpretation process, there's very little difference (at a high level) between the Parse/Eval process of an interpreted language and that of the Compile/Run in compiled languages like C++ and Rust.

::: tip Takeaway
This is why we recommend that you _"think of Nushell as a compiled language"_. Despite being an interpreted language, its lack of `eval` gives it some of the characteristic benefits as well as limitations common in traditional static, compiled languages.
:::

We'll dig deeper into what it means in the next section.

## Implications

Consider this Python example:

```python:line-numbers
exec("def hello(): print('Hello eval!')")
hello()
```

::: note
We're using `exec` in this example instead of `eval` because it can execute any valid Python code rather than being limited to `eval` expressions. The principle is similar in both cases, though.
:::

During interpretation:

1. The entire program is Parsed
2. In order to Evaluate Line 1:
   1. `def hello(): print('Hello eval!')` is Parsed
   2. `def hello(): print('Hello eval!')` is Evaluated
3. (Line 2) `hello()` is evaluated.

Note, that until step 2.2, the interpreter has no idea that a function `hello` even exists! This makes [static analysis](https://en.wikipedia.org/wiki/Static_program_analysis) of dynamic languages challenging. In this example, the existence of the `hello` function cannot be checked just by parsing (compiling) the source code. The interpreter must evaluate (run) the code to discover it.

- In a static, compiled language, a missing function is guaranteed to be caught at compile-time.
- In a dynamic, interpreted language, however, it becomes a _possible_ runtime error. If the `eval`-defined function is conditionally called, the error may not be discovered until that condition is met in production.

::: important
In Nushell, there are **exactly two steps**:

1. Parse the entire source code
2. Evaluate the entire source code

This is the complete Parse/Eval sequence.
:::

::: tip Takeaway
By not allowing `eval`-like functionality, Nushell prevents these types of `eval`-related bugs. Calling a non-existent definition is guaranteed to be caught at parse-time in Nushell.

Furthermore, after parsing completes, we can be certain the bytecode (IR) won't change during evaluation. This gives us a deep insight into the resulting bytecode (IR), allowing for powerful and reliable static analysis and IDE integration which can be challenging to achieve with more dynamic languages.

In general, you have more peace of mind that errors will be caught earlier when scaling Nushell programs.
:::

## The Nushell REPL

As with most any shell, Nushell has a _"Read→Eval→Print Loop"_ ([REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop)) that is started when you run `nu` without any file. This is often thought of, but isn't quite the same, as the _"commandline"_.

::: tip Note
In this section, the `> ` character at the beginning of a line in a code-block is used to represent the commandline **_prompt_**. For instance:

```nu
> some code...
```

Code after the prompt in the following examples is executed by pressing the <kbd>Enter</kbd> key. For example:

```nu
> print "Hello world!"
# => Hello world!

> ls
# => prints files and directories...
```

The above means:

- From inside Nushell (launched with `nu`):
  1. Type `print "Hello world!"`
  1. Press <kbd>Enter</kbd>
  1. Nushell will display the result
  1. Type `ls`
  1. Press <kbd>Enter</kbd>
  1. Nushell will display the result

:::

When you press <kbd>Enter</kbd> after typing a commandline, Nushell:

1. **_(Read):_** Reads the commandline input
1. **_(Evaluate):_** Parses the commandline input
1. **_(Evaluate):_** Evaluates the commandline input
1. **_(Evaluate):_** Merges the environment (such as the current working directory) to the internal Nushell state
1. **_(Print):_** Displays the results (if non-`null`)
1. **_(Loop):_** Waits for another input

In other words, each REPL invocation is its own separate parse-evaluation sequence. By merging the environment back to the Nushell's state, we maintain continuity between the REPL invocations.

Compare a simplified version of the [`cd` example](./thinking_in_nu.md#example-change-to-a-different-directory-cd-and-source-a-file) from _"Thinking in Nu"_:

```nu
cd spam
source-env foo.nu
```

There we saw that this cannot work (as a script or other single expression) because the directory will be changed _after_ the parse-time [`source-env` keyword](/commands/docs/source-env.md) attempts to read the file.

Running these commands as separate REPL entries, however, works:

```nu
> cd spam
> source-env foo.nu
# Yay, works!
```

To see why, let's break down what happens in the example:

1. Read the `cd spam` commandline.
2. Parse the `cd spam` commandline.
3. Evaluate the `cd spam` commandline.
4. Merge environment (including the current directory) into the Nushell state.
5. Read and Parse `source-env foo.nu`.
6. Evaluate `source-env foo.nu`.
7. Merge environment (including any changes from `foo.nu`) into the Nushell state.

When `source-env` tries to open `foo.nu` during the parsing in Step 5, it can do so because the directory change from Step 3 was merged into the Nushell state during Step 4. As a result, it's visible in the following Parse/Eval cycles.

### Multiline REPL Commandlines

Keep in mind that this only works for **_separate_** commandlines.

In Nushell, it's possible to group multiple commands into one commandline using:

- A semicolon:

  ```nu
  cd spam; source-env foo.nu
  ```

- A newline:

  ```
  > cd span
    source-env foo.nu
  ```

  Notice there is no "prompt" before the second line. This type of multiline commandline is usually created with a [keybinding](./line_editor.md#keybindings) to insert a Newline when <kbd>Alt</kbd>+<kbd>Enter</kbd> or <kbd>Shift</kbd>+ <kbd>Enter</kbd> is pressed.

These two examples behave exactly the same in the Nushell REPL. The entire commandline (both statements) are processed a single Read→Eval→Print Loop. As such, they will fail the same way that the earlier script-example did.

::: tip
Multiline commandlines are very useful in Nushell, but watch out for any out-of-order Parser-keywords.
:::

## Parse-time Constant Evaluation

While it is impossible to add parsing into the evaluation stage and yet still maintain our static-language benefits, we can safely add _a little bit_ of evaluation into parsing.

::: tip Terminology
In the text below, we use the term _"constant"_ to refer to:

- A `const` definition
- The result of any command that outputs a constant value when provide constant inputs.
  :::

By their nature, **_constants_** and constant values are known at Parse-time. This, of course, is in sharp contrast to _variable_ declarations and values.

As a result, we can utilize constants as safe, known arguments to parse-time keywords like [`source`](/commands/docs/source.md), [`use`](/commands/docs/use.md), and related commands.

Consider [this example](./thinking_in_nu.md#example-dynamically-creating-a-filename-to-be-sourced) from _"Thinking in Nu"_:

```nu
let my_path = "~/nushell-files"
source $"($my_path)/common.nu"
```

As noted there, we **_can_**, however, do the following instead:

```nu:line-numbers
const my_path = "~/nushell-files"
source $"($my_path)/common.nu"
```

Let's analyze the Parse/Eval process for this version:

1. The entire program is Parsed into IR.

   1. Line 1: The `const` definition is parsed. Because it is a constant assignment (and `const` is also a parser-keyword), that assignment can also be Evaluated at this stage. Its name and value are stored by the Parser.
   2. Line 2: The `source` command is parsed. Because `source` is also a parser-keyword, it is Evaluated at this stage. In this example, however, it can be **_successfully_** parsed since its argument is **_known_** and can be retrieved at this point.
   3. The source-code of `~/nushell-files/common.nu` is parsed. If it is invalid, then an error will be generated, otherwise the IR results will be included in evaluation in the next stage.

2. The entire IR is Evaluated:
   1. Line 1: The `const` definition is Evaluated. The variable is added to the runtime stack.
   2. Line 2: The IR result from parsing `~/nushell-files/common.nu` is Evaluated.

::: important

- An `eval` adds additional parsing during evaluation
- Parse-time constants do the opposite, adding additional evaluation to the parser.
  :::

Also keep in mind that the evaluation allowed during parsing is **_very restricted_**. It is limited to only a small subset of what is allowed during a regular evaluation.

For example, the following is not allowed:

```nu
const foo_contents = (open foo.nu)
```

Put differently, only a small subset of commands and expressions can generate a constant value. For a command to be allowed:

- It must be designed to output a constant value
- All of its inputs must also be constant values, literals, or composite types (e.g., records, lists, tables) of literals.

In general, the commands and resulting expressions will be fairly simple and **_without side effects_**. Otherwise, the parser could all-too-easily enter an unrecoverable state. Imagine, for instance, attempting to assign an infinite stream to a constant. The Parse stage would never complete!

::: tip
You can see which Nushell commands can return constant values using:

```nu
help commands | where is_const
```

:::

For example, the `path join` command can output a constant value. Nushell also defines several useful paths in the `$nu` constant record. These can be combined to create useful parse-time constant evaluations like:

```nu
const my_startup_modules =  $nu.default-config-dir | path join "my-mods"
use $"($my_startup_modules)/my-utils.nu"
```

::: note Additional Notes
Compiled ("static") languages also tend to have a way to convey some logic at compile time. For instance:

- C's preprocessor
- Rust macros
- [Zig's comptime](https://kristoff.it/blog/what-is-zig-comptime), which was an inspiration for Nushell's parse-time constant evaluation.

There are two reasons for this:

1. _Increasing Runtime Performance:_ Logic in the compilation stage doesn't need to be repeated during runtime.

   This isn't currently applicable to Nushell, since the parsed results (IR) are not stored beyond Evaluation. However, this has certainly been considered as a possible future feature.

2. As with Nushell's parse-time constant evaluations, these features help (safely) work around limitations caused by the absence of an `eval` function.
   :::

## Conclusion

Nushell operates in a scripting language space typically dominated by _"dynamic"_, _"interpreted"_ languages, such as Python, Bash, Zsh, Fish, and many others. Nushell is also _"interpreted"_ since code is run immediately (without a separate, manual compilation).

However, is not _"dynamic"_ in that it does not have an `eval` construct. In this respect, it shares more in common with _"static"_, compiled languages like Rust or Zig.

This lack of `eval` is often surprising to many new users and is why it can be helpful to think of Nushell as a compiled, and static, language.
