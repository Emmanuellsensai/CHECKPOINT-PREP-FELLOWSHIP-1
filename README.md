# Checkpoint Study Guide — go-reloaded · ascii-art · ascii-art-web

> Goal: not to memorize, but to understand each project deeply enough to **rebuild it from a blank file** and **explain every function out loud**.

---

## How to use this guide (read this first)

The checkpoint tests two skills:

1. **Explain** — "Walk me through your project. What does this function do? Why is it here?"
2. **Write** — "Implement X from scratch right now."

Reading this guide passively will trick your brain into thinking you know it. You don't, until you can reproduce it. So for every section:

1. Read the code and the *why*.
2. **Close the guide.**
3. Rewrite the function on paper or in an empty file from memory.
4. Open the guide and compare. Wherever you guessed, that's your weak spot — repeat it.

This is called **active recall**. It feels harder than re-reading. That difficulty *is* the learning.

---

## The checkpoint mindset — what examiners actually probe

They rarely ask "does it run." They ask:

- "Why did you split this into functions instead of one big `main`?" → because each function does **one job** (single responsibility), which makes it testable and readable.
- "What does this line do?" → so never have a line you can't explain. If you copied something you don't understand, that's where they'll strike.
- "What happens if the input is empty / wrong / malicious?" → edge cases.
- "Change it live." → e.g. "add a new banner" or "return 400 instead of 500 here."

Rule: **you must be able to explain every single line.** A clever line you can't defend is worse than a simple line you can.

---

# PROJECT 1 — go-reloaded

## Mission (one sentence)

Read a text file, apply a set of text-transformation rules, and write the corrected text to an output file.

## Run signature

```bash
go run . input.txt output.txt
```

## The rules it must handle

| Marker / Rule        | Example in                | Example out          |
|----------------------|---------------------------|----------------------|
| `(hex)`              | `1E (hex) files`          | `30 files`           |
| `(bin)`              | `10 (bin) apples`         | `2 apples`           |
| `(up)`               | `ready (up)`              | `READY`              |
| `(low)`              | `LOUD (low)`              | `loud`               |
| `(cap)`              | `bridge (cap)`            | `Bridge`             |
| `(up, 2)`            | `so exciting (up, 2)`     | `SO EXCITING`        |
| Punctuation spacing  | `hello ,there .`          | `hello, there.`      |
| Single quotes        | `' awesome '`             | `'awesome'`          |
| `a` → `an`           | `a apple`, `a hour`       | `an apple`, `an hour`|

## Mental model

Think in **3 phases**:

1. **Tokenize** — break the text into words.
2. **Apply markers** — markers like `(up)` act on the *word(s) before them*, then the marker itself disappears.
3. **Fix spacing rules** — punctuation, quotes, and `a`/`an` are applied to the rejoined string.

## Function map (this is the breakdown they'll ask for)

```
main()              -> reads args, reads file, calls process(), writes file
process(text)       -> orchestrates the 3 phases, returns final string
applyAction(...)    -> mutates the last n words (up/low/cap/hex/bin)
hexToDec(s)         -> "1E" -> "30"
binToDec(s)         -> "10" -> "2"
capitalize(s)       -> "bridge" -> "Bridge"
fixPunctuation(s)   -> " ." -> "."
fixQuotes(s)        -> "' x '" -> "'x'"
fixArticles(s)      -> "a apple" -> "an apple"
```

Each function does **one thing**. That sentence is your answer when they ask "why so many functions?"

## The code

### `main` — the entry point

```go
package main

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

func main() {
	// Expect exactly: program input.txt output.txt
	if len(os.Args) != 3 {
		fmt.Println("Usage: go run . input.txt output.txt")
		return
	}

	data, err := os.ReadFile(os.Args[1]) // read the whole input file
	if err != nil {
		fmt.Println("Error reading file:", err)
		return
	}

	result := process(string(data)) // do all the work

	err = os.WriteFile(os.Args[2], []byte(result), 0644) // 0644 = rw-r--r--
	if err != nil {
		fmt.Println("Error writing file:", err)
	}
}
```

**Why explain-ready:** `os.Args` holds command-line arguments; `os.Args[0]` is the program name, so we need length 3 for two filenames. `os.ReadFile` returns `[]byte`, so we convert to `string`. `0644` is the file permission (owner read/write, others read).

### `process` — the orchestrator

```go
func process(text string) string {
	words := strings.Fields(text) // split on any whitespace into a slice of words
	var out []string

	for i := 0; i < len(words); i++ {
		w := words[i]

		// Single-token markers: (up) (low) (cap) (hex) (bin)
		if isMarker(w) {
			action := strings.Trim(w, "()") // "(up)" -> "up"
			applyAction(out, action, 1)
			continue // drop the marker itself by not appending it
		}

		// Two-token markers: "(up," followed by "2)"
		if strings.HasPrefix(w, "(") && i+1 < len(words) && strings.HasSuffix(words[i+1], ")") {
			action := strings.Trim(w, "(,")               // "(up," -> "up"
			countStr := strings.TrimSuffix(words[i+1], ")") // "2)" -> "2"
			n, _ := strconv.Atoi(countStr)
			applyAction(out, action, n)
			i++       // skip the count token as well
			continue
		}

		out = append(out, w) // a normal word, keep it
	}

	result := strings.Join(out, " ")
	result = fixPunctuation(result)
	result = fixQuotes(result)
	result = fixArticles(result)
	return result
}
```

**Key idea to say out loud:** markers reference the words *already collected* in `out`, so we apply them to the tail of `out` and never add the marker itself.

### `isMarker` and `applyAction`

```go
func isMarker(w string) bool {
	switch w {
	case "(up)", "(low)", "(cap)", "(hex)", "(bin)":
		return true
	}
	return false
}

// applyAction edits the LAST n words already collected.
// Because slices share their backing array, editing words[idx] is permanent.
func applyAction(words []string, action string, n int) {
	for i := 0; i < n && i < len(words); i++ {
		idx := len(words) - 1 - i // walk backwards from the last word
		switch action {
		case "up":
			words[idx] = strings.ToUpper(words[idx])
		case "low":
			words[idx] = strings.ToLower(words[idx])
		case "cap":
			words[idx] = capitalize(words[idx])
		case "hex":
			words[idx] = hexToDec(words[idx])
		case "bin":
			words[idx] = binToDec(words[idx])
		}
	}
}
```

**Why `len(words) - 1 - i`:** `len-1` is the last word, `len-2` the one before, etc. The `i < len(words)` guard stops us from indexing out of bounds if there are fewer words than `n`.

### The number converters

```go
func hexToDec(s string) string {
	n, err := strconv.ParseInt(s, 16, 64) // base 16, fits in 64-bit int
	if err != nil {
		return s // not valid hex -> leave it unchanged
	}
	return strconv.FormatInt(n, 10) // back to a base-10 string
}

func binToDec(s string) string {
	n, err := strconv.ParseInt(s, 2, 64) // base 2
	if err != nil {
		return s
	}
	return strconv.FormatInt(n, 10)
}
```

**The one fact they love:** `strconv.ParseInt(s, base, bitSize)`. Base 16 = hex, base 2 = binary, base 10 = decimal. It returns `(int64, error)`.

### `capitalize`

```go
func capitalize(s string) string {
	if s == "" {
		return s
	}
	return strings.ToUpper(s[:1]) + strings.ToLower(s[1:])
	// s[:1] = first byte, s[1:] = the rest
}
```

### Spacing rules (the fiddly part — be honest about edge cases)

```go
func fixPunctuation(text string) string {
	for _, p := range []string{".", ",", "!", "?", ":", ";"} {
		text = strings.ReplaceAll(text, " "+p, p) // remove the space BEFORE punctuation
	}
	return text
}

func fixQuotes(text string) string {
	text = strings.ReplaceAll(text, "' ", "'") // space right after opening quote
	text = strings.ReplaceAll(text, " '", "'") // space right before closing quote
	return text
}

func fixArticles(text string) string {
	words := strings.Split(text, " ")
	for i := 0; i < len(words)-1; i++ {
		next := words[i+1]
		if (words[i] == "a" || words[i] == "A") && next != "" && isVowelOrH(next[0]) {
			if words[i] == "a" {
				words[i] = "an"
			} else {
				words[i] = "An"
			}
		}
	}
	return strings.Join(words, " ")
}

func isVowelOrH(c byte) bool {
	lower := strings.ToLower(string(c))
	return strings.Contains("aeiouh", lower)
}
```

> **Honest note:** this punctuation/quote handling is the *clean, explainable* version and passes the common cases. The full spec has nastier edges (punctuation groups like `...` or `!?`, multiple quoted phrases in one line). If you have time, harden these — but for explaining your logic, this structure is exactly what they want to see, and you can say "I handle the standard cases here; the grouped-punctuation edge case would need a character-by-character scan."

## Tricky bits checklist

- Markers act on **previous** words, not following ones.
- `(up, 2)` arrives as **two tokens** after `strings.Fields`: `(up,` and `2)`.
- Editing a slice element inside a function is **permanent** (slices share memory) — that's why `applyAction` works without returning anything.
- `strconv.ParseInt` returns `(value, error)` — always handle the error.

## Practice — EXPLAIN IT (answer out loud)

1. Why do we apply markers to `out` and not to `words`?
2. What does `strconv.ParseInt("FF", 16, 64)` return, and what type?
3. Why does `applyAction` not need to return the slice?
4. Walk through `process` on the input: `a amazing thing (up, 2)`.
5. What's `os.Args[0]`, and why do we check `len(os.Args) != 3`?

## Practice — WRITE IT (blank file, no peeking)

1. Write `hexToDec` and `binToDec` from scratch.
2. Write `capitalize` so `"hELLo"` → `"Hello"`.
3. Write the loop in `applyAction` that walks the last `n` words backwards.
4. Write `fixArticles` for the `a`/`an` rule.

---

# PROJECT 2 — ascii-art

## Mission (one sentence)

Turn a string of text into big ASCII-art letters using a banner font file.

## Run signature

```bash
go run . "Hello"
```

## How the banner file works (the core insight)

A banner file (e.g. `standard.txt`) draws every printable ASCII character (codes **32 to 126**) using exactly **8 lines** of art each.

The file layout:

```
              <- line 0: empty (the file starts with a blank line)
  (8 lines of art for SPACE, ASCII 32)   <- lines 1..8
              <- line 9: empty separator
  (8 lines of art for '!', ASCII 33)     <- lines 10..17
              <- line 18: empty separator
  ...
```

So each character "owns" **9 lines** in the file: 8 art lines + 1 separator.

### THE FORMULA (memorize and be able to derive it)

For a character `c`, the first of its 8 art lines is at index:

```
start = (int(c) - 32) * 9 + 1
```

- `- 32` → because ASCII 32 (space) is the **first** character represented.
- `* 9` → because each character block is **9 lines** tall (8 art + 1 blank).
- `+ 1` → because the file **begins with one blank line** before the first character.

Its 8 rows are then `start+0`, `start+1`, ..., `start+7`.

**Derive it live:** "Space is ASCII 32 and comes first, so subtract 32 to make it 0-indexed. Each letter takes 9 lines, so multiply by 9. The file opens with a blank line, so add 1."

## Mental model

To draw a word, you **print it row by row, not letter by letter.** For row 0, you print row 0 of every letter side by side. Then row 1 of every letter. Eight rows total. That's why letters appear next to each other instead of stacked.

## Function map

```
main()                  -> parse args, read banner, handle \n, call printLine
printLine(text, banner) -> prints one line of input as 8 rows of ASCII art
```

## The code

```go
package main

import (
	"fmt"
	"os"
	"strings"
)

func main() {
	if len(os.Args) != 2 {
		fmt.Println("Usage: go run . \"text\"")
		return
	}
	input := os.Args[1]

	data, err := os.ReadFile("standard.txt")
	if err != nil {
		fmt.Println("Error:", err)
		return
	}
	banner := strings.Split(string(data), "\n") // each element is one line of the file

	// The user types \n (backslash-n) to request a line break.
	// In the argument it is the two characters: '\' and 'n'.
	for _, segment := range strings.Split(input, "\\n") {
		printLine(segment, banner)
	}
}

func printLine(text string, banner []string) {
	if text == "" {
		fmt.Println() // empty segment -> just a blank line
		return
	}
	for row := 0; row < 8; row++ { // 8 rows make up one line of art
		line := ""
		for _, char := range text {
			start := (int(char)-32)*9 + 1 // first art line for this char
			line += banner[start+row]     // glue this char's current row on
		}
		fmt.Println(line)
	}
}
```

**Line they'll point at:** `line += banner[start+row]`. Explain: "I'm building one output row by concatenating the same row from each character's 8-line block."

## Tricky bits checklist

- The file **starts with a blank line** — forget the `+1` and everything shifts up by one row (you'll print garbage). This is the #1 bug.
- Input `\n` is **two characters** in the argument, so you split on `"\\n"` (escaped backslash) in Go source.
- Output is **row-major**: outer loop = 8 rows, inner loop = each character.
- Only ASCII 32–126 exist in the file. Anything outside (like a tab or emoji) will index out of range — that's why ascii-art-web later validates input.

## Practice — EXPLAIN IT

1. Derive the `(int(c) - 32) * 9 + 1` formula from scratch.
2. Why does the outer loop run exactly 8 times?
3. Why is the inner loop over characters and the outer loop over rows, not the other way round?
4. What breaks if you forget the `+ 1`?
5. How does the program produce a line break from input like `"a\nb"`?

## Practice — WRITE IT

1. Write `printLine` from a blank file.
2. Write just the indexing line and explain each term.
3. Modify `main` to accept a second argument for the banner name (`standard`, `shadow`, `thinkertoy`) and read `name + ".txt"`.

---

# PROJECT 3 — ascii-art-web

## Mission (one sentence)

Wrap the ascii-art engine in a web server: a user types text in a browser form, picks a banner, submits, and sees the ASCII art on the page.

## Architecture (the whole thing in 3 routes)

| Method | Path          | Handler            | Job                                |
|--------|---------------|--------------------|------------------------------------|
| GET    | `/`           | `homeHandler`      | Show the page with the input form  |
| POST   | `/ascii-art`  | `asciiArtHandler`  | Read form, generate art, show result|
| GET    | `/static/...` | file server        | Serve CSS / assets                 |

## Core concepts (each is a likely question)

- **`http.HandleFunc(path, handler)`** registers which function handles which path.
- **`http.ListenAndServe(":8080", nil)`** starts the server on port 8080.
- **`r.Method`** tells you GET / POST. **`r.URL.Path`** is the requested path.
- **`r.FormValue("name")`** reads a form field from a POST request.
- **`html/template`** renders HTML and auto-escapes data (prevents injection).
- **Status codes:** 200 OK · 400 bad input · 404 path not found · 405 wrong method · 500 server error.

## Function map

```
main()              -> register routes, start server
homeHandler()       -> guard path + method, render the form page
asciiArtHandler()   -> guard method, read form, generate, render result
generate(text,b)    -> the ascii-art engine from Project 2, returning a string + error
```

## The code

```go
package main

import (
	"html/template"
	"net/http"
)

type PageData struct {
	Art string // whatever we put here, the template can display
}

func main() {
	http.HandleFunc("/", homeHandler)
	http.HandleFunc("/ascii-art", asciiArtHandler)

	// Serve files in the ./static folder under the /static/ URL.
	fs := http.FileServer(http.Dir("static"))
	http.Handle("/static/", http.StripPrefix("/static/", fs))

	http.ListenAndServe(":8080", nil)
}

func homeHandler(w http.ResponseWriter, r *http.Request) {
	// "/" is a catch-all in Go, so we must reject anything that isn't exactly "/".
	if r.URL.Path != "/" {
		http.Error(w, "404 page not found", http.StatusNotFound)
		return
	}
	if r.Method != http.MethodGet {
		http.Error(w, "405 method not allowed", http.StatusMethodNotAllowed)
		return
	}

	tmpl, err := template.ParseFiles("index.html")
	if err != nil {
		http.Error(w, "500 internal server error", http.StatusInternalServerError)
		return
	}
	tmpl.Execute(w, nil) // no data to show yet, just the empty form
}

func asciiArtHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "405 method not allowed", http.StatusMethodNotAllowed)
		return
	}

	text := r.FormValue("text")
	banner := r.FormValue("banner")

	art, err := generate(text, banner) // your Project 2 engine, returning (string, error)
	if err != nil {
		http.Error(w, "400 bad request", http.StatusBadRequest)
		return
	}

	tmpl, err := template.ParseFiles("index.html")
	if err != nil {
		http.Error(w, "500 internal server error", http.StatusInternalServerError)
		return
	}
	tmpl.Execute(w, PageData{Art: art}) // pass the art into the template
}
```

## The two rules that trip everyone

**1. Why guard the path in `homeHandler`?**
In Go, the pattern `"/"` matches **every** path that no other handler claims. So a request to `/banana` would still hit `homeHandler`. Without `if r.URL.Path != "/"`, you'd wrongly show the home page for unknown URLs instead of a 404.

**2. Why must headers come before the body?**
Once you write any body bytes, Go automatically sends a `200 OK` header. After that you **cannot** change the status code. So you must call `http.Error` / `w.WriteHeader(...)` **before** writing content. Getting this backwards means your 404 silently becomes a 200.

## 404 vs 405 — say the difference cleanly

- **404 Not Found** → the *path* doesn't exist (`/banana`).
- **405 Method Not Allowed** → the path exists but you used the *wrong verb* (GET on `/ascii-art`, which only takes POST).

## Tricky bits checklist

- `r.FormValue` works for POST form fields; the field name must match the `name=""` in your HTML `<input>`/`<select>`.
- Use `html/template`, **not** `text/template` — the `html` one auto-escapes and is safe for web output.
- The `PageData` struct field must be **capitalized** (`Art`, not `art`) or the template can't see it (Go exports only capitalized fields).
- Static files: `http.StripPrefix` removes `/static/` so `http.Dir("static")` finds the real file.

## Practice — EXPLAIN IT

1. Why do you check `r.URL.Path != "/"` inside `homeHandler`?
2. Difference between 404 and 405 — give an example URL for each.
3. Why must `http.Error` be called before writing any page content?
4. Why is the struct field `Art` capitalized?
5. What does `http.StripPrefix` do and why is it needed for static files?
6. Why `html/template` and not `text/template`?

## Practice — WRITE IT

1. Write `main` with all three route registrations from memory.
2. Write `homeHandler` with both guards.
3. Write `asciiArtHandler`: method guard → read form → generate → handle error → render.
4. Add a 400 response for when the text contains a non-ASCII character.

---

# Cross-cutting Go fundamentals they test on any project

These show up regardless of which project:

- **Slices share memory.** Passing a slice to a function and editing its elements changes the original. (Why `applyAction` works.)
- **Multiple return values & errors.** `v, err := f()` — always check `err`. Never ignore it in code you'll defend.
- **`strings` toolkit:** `Split`, `Fields`, `Join`, `ReplaceAll`, `ToUpper`, `ToLower`, `TrimSuffix`, `HasPrefix`, `Contains`.
- **`strconv`:** `ParseInt(s, base, bits)`, `Atoi(s)`, `FormatInt(n, base)`.
- **Runes vs bytes:** `for _, c := range s` gives runes (good for characters); `s[i]` gives a single byte.
- **Exported names:** capitalized = visible outside the package / to templates.
- **`os.Args`:** index 0 is the program name; real arguments start at index 1.

---

# Your 2-day plan (Wed–Thu, checkpoint Fri)

**Wednesday**
- Morning: go-reloaded. Read the section once, then rewrite every function from a blank file. Run it against the example table.
- Afternoon: ascii-art. Derive the formula 5 times until it's automatic. Rewrite `printLine` from scratch.

**Thursday**
- Morning: ascii-art-web. Rewrite all three handlers from memory. Be able to recite the route table and status codes.
- Afternoon: mixed drilling — answer every "EXPLAIN IT" question out loud as if facing the examiner. Re-do any "WRITE IT" you stumbled on.

**Friday morning (light)**
- Skim the three "Tricky bits" checklists.
- Re-derive the ascii-art formula and re-state 404 vs 405. Don't cram new things.

**The night before:** sleep. A rested brain recalls; a tired one blanks.

---

*Built for your Friday checkpoint. You don't need to memorize this — you need to be able to rebuild it. Close the guide and prove you can.*
