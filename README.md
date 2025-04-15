# minishell
A program that recreates bash

### Hierarchy

Shells have a natural hierarchy, pipelines redirections etc. Each node can represent a command and each child nodes can represent components, arguments, files for redirection, subcommands.

Recursive descent parser has a function for each type of grammar rule, commands, operators, arguments etc. This makes it easier to extend the shell with more complex features.

< file cmd1
cmd1 < file

file1 cmd1 cmd2 file2

cmd1 < file1  &&  cmd2 > file2

### High-level Design

- Split the string into tokens (commands, operators (`|`, `&&`, `>` etc.))
- Define grammar rules for shell syntax
- Write parsing functions that correspond to each grammar rule.
	- `parse_command()`
	- `parse_pipeline()`
	- `parse_expression()` - `&&` `||`
- Abstract Syntax Tree
	- Leaf nodes are individual commands
	- Internal nodes are operators (e.g. `&&`)
- To execute traverse the tree to execute commands
	- For a pipeline, traverse each command node in the order they need to be executed, chaining outputs
	- For logical operations, evaluate the left and right child nodes based on the operator (e.g. `&&` only evaluates the right side if the left succeeds).

For an input like:  
`cmd1 | cmd2 && cmd3 > output.txt`

Your parser could produce an AST that looks like:  
```
 	       &&  
	     /    \
	   |        >
	  / \      / \
      cmd1 cmd2 cmd3 output.txt
```

- The `|` node means `cmd1` pipes into `cmd2`.
- The `&&` node means `cmd3` runs only if the pipeline `cmd1 | cmd2` succeeds.
- The `>` node means `cmd3` redirects its output to `output.txt`.

#### Grammar for Example Language

`::=` is defined as

```BNF
<sentence> ::= <subj> <verb> <obj>
    <subj> ::= <art> <noun> | the robot
	 <art> ::= the | a
	<noun> ::= dog | cat| man | woman | robot
	<verb> ::= bit | kicked | stroked
	 <obj> ::= <art> <noun> | two furry dice   ; "two furry dice is a globbet that isn't parsed"
```

Each symbol on the left (`<sentence>`) has a stack of prerequisites on the right. So it will search for a `<subj>` then a `<verb>` and finally an `<obj>`.  
Once `<subj>` has been satisfied, it is popped off the top of the stack.

_When parsing `<subj>`, we see that it could be `<art>` and `<noun>` so those will be added to the top of the stack and popped when they are satisfied._

Sentence: _"The robot stroked two furry dice"_

We will use the top-down approach i.e. don't look for leaf nodes in the tree and try and work your way up. Instead always start at the top (the root) of the tree and work your way down to see if a sentence can be parsed (decoded).

### Double Quotes

Allow for variable expansion and command substitution but prevent word splitting and interpretation of special characters (except `$`, and `\`). We only need to worry about `$` (an env variable) in minishell.  
  
Without a space before the quotes, the shell interprets the word before and after the quotes as one token.  
  
E.g. `echo abc"def"` outputs `abcdef`, any spaces inside the quotes would also be preserved.  
  
`set -- abc"def"; echo $#` outputs `1`  
`set -- abc "def"; echo $#` outputs `2`  
  
`set --` reassigns positional parameters `$1 $2` etc. to each token.  
`echo $#` prints the number of assigned positional parameters i.e. tokens  
  
`set -- abc "def ghi"; echo $#` outputs `2`  
  
### Single Quotes  

Single quotes prevent all special character interpretation.

## Implementation

### Grammar

`::=` - is defined as
`<symbol>`
`[foo]` - `foo` is optional
`^` - not
`*` - zero or more
`+` - one or more

`<compound_command>` is run separately to other commands. In the shell any changes compound commands make to environment variables do not affect the other commands (which are run in the parent shell). The subject only mentions priority so subshells I think are beyond the scope of the project.

```CFG
# Top-level structure
; command line input is defined as command_sequence or compound_command
<command_line> ::= <command_sequence> | <compound_command>

<command_sequence> ::= <pipeline> | <command_sequence> "&&" <pipeline> |<command_sequence> "||" <pipeline>

<compound_command> ::= "(" <command_sequence> ")"

# Pipeline
<pipeline> ::= <command> | <pipeline> "|" <command>

# Command structure
<command> ::= <simple_command> [<redirections>]

<simple_command> ::= <command_prefix> [<command_word>] [<command_suffix>] | <command_word> [<command_suffix>]

<command_prefix> ::= <io_redirect> | <command_prefix> <io_redirect>

<command_suffix> ::= <io_redirect> | <word> | <command_suffix> <io_redirect> | <command_suffix> <word>

<command_word> ::= <word>

# Word and quoting
<word> ::= <unquoted_word> | <single_quoted_content> | <double_quoted_content>

<unquoted_word> ::= <char>+ | "$" <env_var_name>

; single_quoted_content starts and ends with single quotes and contains anything else inside. optional ([]) any character (*) not (^) single quote (')
<single_quoted_content> ::= "'" [^']* "'"

<double_quoted_content> ::= '"' (<char> | "$" <env_var_name> | <whitespace>)* '"'

; environment variables must have one or more alphanumeric chars (or underscore)
<env_var_name> ::= [a-zA-Z0-9_]+

<char> ::= any printable character except whitespace and metacharacters

# Redirections
<redirections> ::= <io_redirect> | <redirections> <io_redirect>

<io_redirect> ::= <input_redirect> | <output_redirect> | <heredoc>

<input_redirect> ::= "<" <word>

<output_redirect> ::= ">" <word> | ">>" <word>

<heredoc> ::= "<<" <word>

# Additional rules
<whitespace> ::= [ \t]+
; do we need a newline rule?
```

### Terminal and Non-Terminal Symbols

[wiki](https://en.wikipedia.org/wiki/Terminal_and_nonterminal_symbols)

In the grammar, the `Variables`, (`<Noun>`, `<Verb>`, etc.) are `Non-terminals`.

`Constants` (`Dog`, `is`, etc.) are `Terminals`.

Once a noun or verb is chosen from the available choices, it is fixed and is a terminal symbol.

A grammar is described as four tuples (`N`, `T`, `P`, `S`) where:
`N` ::= a finite non-empty set of Non-terminals.
`T` ::= a finite non-empty set of Terminals.
`N`∩`T` = Φ the intersection of `N` and `T` is phi (an empty set). i.e. they have no elements in common (are two disjoint sets).
`S` ::= a special Non-terminal (i.e. `S` ∈ `N`). It is the Start symbol because from `S` the process of derviation will initiate.
`P` ::= a finite set whose elements are of the form α→β (alpha can be written as beta). Alpha and beta are strings over `N`∪`T` (union of `N` and `T`). The elements that they have in common. Means it may be a mixture of Non-terminals and Terminals or only Terminals or only Non-Terminals. Alpha must have at least one Non-terminal. These elements are called Porduction rules.

## Functions

### `readline()`

`readline(char *prompt)`  
  
Requires `-lreadline` flag to compile.  
`prompt` displays as a prompt.  
Returns the line entered on the terminal. This is mallocated and must be freed by the user.  
If `EOF` is encountered while reading the line and the line is empty `NULL` is returned.

[gnu](https://web.mit.edu/gnu/doc/html/rlman_2.html)
[readline](https://tiswww.case.edu/php/chet/readline/readline.html#index-rl_005fon_005fnew_005fline)

#### `rl_clear_history()`

Clears `readline`'s private history list.

#### `rl_on_newline()`

Call this before redisplaying the input prompt, esp in cases where the previous line might not have ended in a newline (e.g. when a partial command is given or the line editing gets interrupted (`SIGINT`)).

#### `rl_redisplay()`

Redisplays the prompt.

#### `rl_replace_line()`

`int rl_replace_line(const char *text, int clear_undo);.  
`text` is a pointer to the string that will replace the current line.

#### `add_history()`

`void add_history(const char *line);.  
`readline()` doesn't automatically store every input, you have to manually call `add_history()` to store it. Allows you to skip non-meaningful lines.

## Bash/Minishell Parsing Rules

- Special characters do not need whitespace before/around them:
  - `echo$PATH`, `sleep 5|sleep 1`, `sleep 1| sleep1` are fine
- Multiple redirects can occur but redirect takes precedence over the pipe. In this instance if file contains only one word, the output will be 1 and that will come from the second command.
  - `wc -w < file | < file wc -w`
- If `&` is not immediately followed by another `&`, it is an error. In bash it would be fine but we are not handling a single `&`.
- Redirections occur before a command is executed.
  - [GNU Documentation](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)
- Command substitution $(ls) allows the output of a command to be replaced by the command itself.
  - this is not required by the subject

Shell Grammar

- A _blank_ is a space or a tab used to separate items
- A _word_ is a sequences of chars considered to be a single unit/token by the shell
- A _name_ is a word consisting of alphanumerics and underscore
- A _metacharacter_ is a special character that represents something other than the ASCII value of itself
  - `|` - pipe
  - `()` - execute in subshell
  - `<` - redirect in
  - `>` - redirecout out
  - `<<` - here_doc
  - `>>` - redirect append out
- A _control operator_ is a token that performs a control function
  - `&&` executes depending on success of previous command (if previous command has exit code of `0`)
  - `||` executes depending on failure of previous command (if previous command has exit code different to `0`)
- _Commands_
  - Simple command - `ls`, `touch`, `pwd`
- _Pipelines_
  - Simple commands separated by pipes
- Overlapping/mismatched quotes e.g. `"a 'weird" string'`
  - the first single quote would be treated as a character, not a delimiter so the second token would be `string'` and that single quote would have no closing partner. It would be a syntax error.

```BASH
# Top-level structure
; command line input is defined as command_sequence or compound_command
<command_line> ::= <command_sequence>

<command_sequence> ::= <pipeline> | <command_sequence> "&&" <pipeline> |<command_sequence> "||" <pipeline>

<compound_command> ::= "(" <command_sequence> ")"

# Pipeline
<pipeline> ::= <command> | <pipeline> "|" <command>

# Command structure
<command> ::= <simple_command> [<redirections>]

<simple_command> ::= <command_prefix> [<command_word>] [<command_suffix>] | <command_word> [<command_suffix>]

<command_prefix> ::= <io_redirect> | <command_prefix> <io_redirect>

<command_suffix> ::= <io_redirect> | <word> | <command_suffix> <io_redirect> | <command_suffix> <word>

<command_word> ::= <word>

# Word and quoting
<word> ::= <unquoted_word> | <single_quoted_content> | <double_quoted_content>

<unquoted_word> ::= <char>+ | "$" <env_var_name>

; single_quoted_content starts and ends with single quotes and contains anything else inside. optional ([]) any character (*) not (^) single quote (')
<single_quoted_content> ::= "'" [^']* "'"

<double_quoted_content> ::= '"' (<char> | "$" <env_var_name> | <whitespace>)* '"'

; environment variables must have one or more alphanumeric chars (or underscore)
<env_var_name> ::= [a-zA-Z0-9_]+

<char> ::= any printable character except whitespace and metacharacters

# Redirections
<redirections> ::= <io_redirect> | <redirections> <io_redirect>

<io_redirect> ::= <input_redirect> | <output_redirect> | <heredoc>

<input_redirect> ::= "<" <word>

<output_redirect> ::= ">" <word> | ">>" <word>

<heredoc> ::= "<<" <word>

# Additional rules
<whitespace> ::= [ \t]+
; do we need a newline rule?
```

tokenising/lexing (linked list so you don't need to count total tokens like you do with an array - can also save information like the token type into the node).
expansion (for environment vars)
parsing

## `$` Environment Variables

`$`  - is literal when followed by `space`, `|` pipe, `&` AND, `\0` NUL, `"` or `'` quotes, `(` or `)` parentheses, `<` or `>` redirect etc. Basically any metacharacter except `$` itself 
`$?` - last exit code
`$[alnum_]` - followed by alphanumeric or underscore is an environment variable which itself is limited by anything non `[alnum_]`.

## Glob/Wildcard

`echo *` prints all filenames in the current directory.

## Expansion

[Bash Manual: Expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html)

Order of expansion:

- brace expansion
- tilde expansion
- parameter and variable expansion
- command substitution
- arithmetic expansion
- word splitting
- filename expansion

_After expansion, quotes are removed unless they have been quoted themselves or are a result of expansion._

## `getenv()`

We should make our own copy of `envp` and `getenv()`. `envp` is stored in the parent process. This also means we can add to the environment variable list if.

## Abstract Syntax Tree

In an AST each node represents another operation in a shell command (like a pipe, a command, a redirect etc.).

Each AST node has `left` and `right` branches to describe hierarchical relationships and `next` to describe sequential relationships. i.e. `next` will be used to group sequential words together, like the parameters and flags of a command.

They can then be combined into each command's `argv` array.
`[cmd, --flag, etc]`

Example AST for: echo hello world > output.txt | grep foo

```plaintext
             PIPELINE
                |
        ----------------
        |              |
    COMMAND         COMMAND
        |              |
       echo           grep
        |              |
       hello          foo
        |
       world
        |
    REDIRECT(>)
        |
    output.txt
```

In memory, using next pointers:  
Command1's words: echo->hello->world->REDIRECT->output.txt  
Command2's words: grep->foo  

For: ls -l && grep foo || wc -l

```plaintext
         OR
         |
    -------------
    |           |
   AND        COMMAND(wc)
    |           |
---------      -l
|       |
ls     COMMAND(grep)
|       |
-l      foo
```

In memory:  
Command1's words: ls->-l  
Command2's words: grep->foo  
Command3's words: wc->-l  

When `BASH` finds a syntax error, it reports near which symbol the error has ocurred. After consuming the tokens, any symbols that are remaining is where the syntax error has ocurred.

https://craftinginterpreters.com/parsing-expressions.html
https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_10
https://github.com/izenynn/mini-shell#shell-grammar

<< EOF wc -w > > outfile


Todo  
BASH special variables
https://www.gnu.org/software/bash/manual/bash.html#Special-Parameters

[PARSING] `awk ' {count++} END {print count}'

[PARSING] "abc'def'" - adjacent quotes considered one token
l"s"
'l'"s"
"l""s"
"l""""""s"

are all single tokens and should be interpreted as "ls"

bool combine_next field of token list can be used to achieve this

readline()

[tokenising/lexing] hello$ is treated as two words hello & $. Should be one
echo hello$ | wc -w -> 1 bash. would be 2 minishell

Question. Do you think $$ or even $$$ for that matter should be treated as a syntax error since we are not handling $$ (current process id) or should it be treated as a token?

input that ends in pipe should return new prompt > where input can be completed
minishell> echo hello |
> wc -w

results in 1

* wildcards (only for the current directory)

[PARSING] ${USER}_test1 should expand $USER and cat _test1 to become one word token
can use the same bool combine_next in token node to handle this.
also implement { } into variable rules

[Parsing] Empty strings issue

Not sure it's relevant (yet).

When splitting the commands into tokens (words (taking quotations into account)), my function correctly ignores '' and "" but treats " " and '   ' etc. as a token. I think this can be ignore when it is parsed, since the commands will ignore the blank space regardless. Just a heads up

EXECUTION
All commands in a multicommand pipeline are run in their own shell whereas single commands are run in the current shell. This is why `exit | exit` does not quit the current shell, only their subshells. In our program, perhaps confusingly, all multicommand pipelines have the pipeline node whereas single commands do not.

PIPELINE
  |- Left:
    PIPELINE
      |- Left:
        COMMAND
          |- Left:
            CMD_NAME (cmd1)
              |- Right:
                IO_REDIRECT (<)
                  |- Right:
                    FILENAME (infile)
                      |- Right:
                        ARG (arg1)
      |- Right:
        COMMAND
          |- Left:
            CMD_NAME (cmd2)
              |- Right:
                ARG (arg2)
  |- Right:
    COMMAND
      |- Left:
        CMD_NAME (cmd3)
          |- Right:
            ARG (arg3)
              |- Right:
                IO_REDIRECT (>)
                  |- Right:
                    FILENAME (outfile)
<<<<<<< HEAD
