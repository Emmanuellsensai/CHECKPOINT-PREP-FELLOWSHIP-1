# go-reloaded

A command-line text tool written in Go. It reads a text file, applies a set of formatting and transformation rules, and writes the corrected text to an output file. It uses only the Go standard library (no external dependencies).

## Features

The program scans the input text and applies these rules:

| Rule | Marker / trigger | Input | Output |
|------|------------------|-------|--------|
| Hex to decimal | `(hex)` | `1E (hex)` | `30` |
| Binary to decimal | `(bin)` | `10 (bin)` | `2` |
| Uppercase | `(up)` | `ready (up)` | `READY` |
| Lowercase | `(low)` | `LOUD (low)` | `loud` |
| Capitalize | `(cap)` | `bridge (cap)` | `Bridge` |
| Apply to N previous words | `(up, 2)` / `(low, 3)` / `(cap, 2)` | `so exciting (up, 2)` | `SO EXCITING` |
| Punctuation spacing | `. , ! ? : ;` | `hello ,there .` | `hello, there.` |
| Single quotes | `' '` | `' awesome '` | `'awesome'` |
| Article correction | `a` before a vowel or `h` | `a apple` / `a hour` | `an apple` / `an hour` |

Notes:
- A marker acts on the word immediately before it. With a count, like `(up, 2)`, it acts on that many previous words.
- Punctuation attaches to the word before it and gets one space after, while groups such as `...` and `!?` are kept together.

## Prerequisites

- Go 1.18 or later installed and on your PATH.

## Usage

Run the program with two arguments: the input file and the output file.

```bash
go run . input.txt output.txt
```

The program writes the transformed text to `output.txt`. If the wrong number of arguments is given, the program exits without doing anything.

### Example

Given an `input.txt` containing:

```
it (cap) was the (up) best of times. it was the worst of times (up, 4)
a amazing thing happened ,and i said ' wow '
```

Run:

```bash
go run . input.txt output.txt
```

`output.txt` then contains text with the markers applied, the punctuation tightened, the quotes wrapped, and `a` corrected to `an`.

## How it works

`main` is a simple pipeline. It reads the file into a string, passes that string through three transformation stages in order, then writes the result out:

```
text -> checkFlags -> checkPunctuations -> checkVowels -> output
```

Each stage takes a string and returns a string.

### Project structure

| Function | Job |
|----------|-----|
| `main` | Reads the two file arguments, reads the input file, runs the three stages, writes the output file. |
| `checkFlags` | Handles the markers `(hex)`, `(bin)`, `(up)`, `(low)`, `(cap)` and their counted forms like `(up, 2)`. Works on a slice of words, reaching back to transform the previous word or words, then removing the marker. |
| `checkPunctuations` | Works character by character. Removes the space before punctuation, adds a single space after it (keeping groups like `...` together), and tightens single quotes around their content. |
| `checkVowels` | Works on words. Changes `a` or `A` to `an` or `An` when the next word starts with a vowel or `h`. |

### Standard library packages used

- `os` for reading arguments and files.
- `fmt` for output and number-to-string conversion.
- `strconv` for parsing hex and binary values (`ParseInt`) and the count (`Atoi`).
- `strings` for splitting, trimming, joining, case conversion, and the `strings.Builder` used while rebuilding punctuation.

## Author

Usang Emmanuel Inyang (Emmanuellsensai)
