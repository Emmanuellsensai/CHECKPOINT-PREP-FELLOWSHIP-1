# Checkpoint Study Guide - go-reloaded · ascii-art · ascii-art-web

> Goal: not to memorize, but to understand each project deeply enough to **rebuild it from a blank file** and **explain every function out loud**.

---

## How to use this guide (read this first)

The checkpoint tests two skills:

1. **Explain** - "Walk me through your project. What does this function do? Why is it here?"
2. **Write** - "Implement X from scratch right now."

Reading this guide passively will trick your brain into thinking you know it. You don't, until you can reproduce it. So for every section:

1. Read the code and the *why*.
2. **Close the guide.**
3. Rewrite the function on paper or in an empty file from memory.
4. Open the guide and compare. Wherever you guessed, that's your weak spot - repeat it.

This is called **active recall**. It feels harder than re-reading. That difficulty *is* the learning.

---

## The checkpoint mindset - what examiners actually probe

They rarely ask "does it run." They ask:

- "Why did you split this into functions instead of one big `main`?" → because each function does **one job** (single responsibility), which makes it testable and readable.
- "What does this line do?" → so never have a line you can't explain. If you copied something you don't understand, that's where they'll strike.
- "What happens if the input is empty / wrong / malicious?" → edge cases.
- "Change it live." → e.g. "add a new banner" or "return 400 instead of 500 here."

Rule: **you must be able to explain every single line.** A clever line you can't defend is worse than a simple line you can.

---

# PROJECT 1 - go-reloaded

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

`main` is a **pipeline**. Read the file, then push the text through three stages in order, then write it out:

```
text -> checkFlags -> checkPunctuations -> checkVowels -> output
```

Each stage takes a string and returns a string. That sentence is your answer when they ask "how is your project structured?"

- **checkFlags** handles the markers: `(hex)`, `(bin)`, `(up)`, `(low)`, `(cap)`, and `(up, 2)`. Works on **words**.
- **checkPunctuations** fixes spacing around `. , ! ? : ;` and single quotes. Works on **characters**.
- **checkVowels** changes `a` to `an` before a vowel or `h`. Works on **words**.

## Function map (this is the breakdown they'll ask for)

```
main()                 -> read args, read file, run the 3 stages, write file
checkFlags(text)       -> apply (up)/(low)/(cap)/(hex)/(bin) to previous word(s)
checkPunctuations(text)-> no space before punctuation, one space after; fix quotes
checkVowels(text)      -> "a apple" -> "an apple"
```

## The code

### `main` - the pipeline

```go
func main() {
	if len(os.Args) != 3 { // program + input file + output file
		return
	}
	inputfile := os.Args[1]
	outputfile := os.Args[2]

	data, err := os.ReadFile(inputfile) // returns bytes
	if err != nil {
		fmt.Println("Error reading file:", err)
		return
	}

	text := string(data)            // bytes -> string
	text = checkFlags(text)         // stage 1: markers
	text = checkPunctuations(text)  // stage 2: spacing + quotes
	text = checkVowels(text)        // stage 3: a -> an

	err = os.WriteFile(outputfile, []byte(text), 0644) // 0644 = rw-r--r--
	if err != nil {
		fmt.Println("Error writing file:", err)
	}
}
```

**Say it out loud:** "I read the file into a string, pass it through three transformation functions one after another, then write the final string back out. Each function does one job."

### `checkFlags` - the markers (works on words)

```go
func checkFlags(text string) string {
	words := strings.Fields(text) // split on whitespace into a slice of words
	for i := 0; i < len(words); i++ {
		word := words[i]
		if strings.HasPrefix(word, "(") { // a word starting with "(" might be a marker
			op := ""
			count := 1        // how many previous words to change (default 1)
			isComplex := false // true for the two-word form like (up, 2)

			// Simple form: "(up)" starts with "(" AND ends with ")"
			if strings.HasSuffix(word, ")") {
				op = strings.Trim(word, "()") // "(up)" -> "up"
			}

			// Complex form: "(up," followed by "2)"
			if strings.HasSuffix(word, ",") && i+1 <= len(words) && strings.HasSuffix(words[i+1], ")") {
				op = strings.Trim(word, "(,")          // "(up," -> "up"
				numStr := strings.Trim(words[i+1], "()") // "2)" -> "2"
				val, err := strconv.Atoi(numStr)
				if err == nil {
					count = val
					isComplex = true
				}
			}

			if op == "hex" || op == "bin" || op == "up" || op == "low" || op == "cap" {
				// Reach BACKWARD: j=1 is the previous word, j=2 the one before, etc.
				for j := 1; j <= count && i-j >= 0; j++ {
					target := words[i-j]
					switch op {
					case "hex":
						val, _ := strconv.ParseInt(target, 16, 64) // read as base 16
						words[i-j] = fmt.Sprint(val)               // number -> string
					case "bin":
						val, _ := strconv.ParseInt(target, 2, 64) // read as base 2
						words[i-j] = fmt.Sprint(val)
					case "up":
						words[i-j] = strings.ToUpper(target)
					case "low":
						words[i-j] = strings.ToLower(target)
					case "cap":
						words[i-j] = strings.Title(strings.ToLower(target)) // first letter up
					}
				}

				// Delete the marker so it doesn't appear in the output.
				if isComplex {
					words = append(words[:i], words[i+2:]...) // remove 2 tokens
				} else {
					words = append(words[:i], words[i+1:]...) // remove 1 token
				}
				i-- // list got shorter, step back so we don't skip a word
			}
		}
	}
	return strings.Join(words, " ")
}
```

**The three ideas that unlock this function:**

1. **Two shapes of marker.** Simple `(up)` is one word (starts `(`, ends `)`). Complex `(up, 2)` is two words (`(up,` then `2)`). The code detects which and sets `op` and `count`.
2. **Reaching backward.** `words[i-j]` is the j-th word *before* the marker. `j` runs from 1 up to `count`, and `i-j >= 0` stops it walking off the front. That loop IS the "affect the previous word(s)" rule.
3. **Deleting the marker.** `words[:i]` is everything before the marker; `words[i+1:]` (or `words[i+2:]` for complex) is everything after. Glue them and the marker disappears. `i--` keeps the loop aligned after the list shrinks.

**Trace `so cool (up, 2)`:** words = `["so","cool","(up,","2)"]`. At the marker `op="up"`, `count=2`. `j=1` -> `words[1]` "cool" becomes "COOL". `j=2` -> `words[0]` "so" becomes "SO". Delete the two marker tokens. Result: `SO COOL`.

### `checkPunctuations` - spacing and quotes (works on characters)

```go
func checkPunctuations(text string) string {
	var result strings.Builder
	for i := 0; i < len(text); i++ {
		// RULE 1: drop a space that sits right BEFORE punctuation
		if text[i] == ' ' && i+1 < len(text) && strings.ContainsRune(".,!?:;", rune(text[i+1])) {
			continue
		}
		result.WriteByte(text[i])
		// RULE 2: add ONE space AFTER punctuation (but not inside a group like "...")
		if strings.ContainsRune(".,!?:;", rune(text[i])) && i+1 < len(text) && !strings.ContainsRune(".,!?:; ", rune(text[i+1])) {
			result.WriteByte(' ')
		}
	}
	text = result.String()

	// Opening quote: remove the space right after it. "' awesome" -> "'awesome"
	text = strings.ReplaceAll(text, "' ", "'")

	// Closing quote: remove the space right before it, when a real word ends there.
	var sb strings.Builder
	for i := 0; i < len(text); i++ {
		if text[i] == ' ' && i+1 < len(text) && text[i+1] == '\'' && i > 0 {
			prev := text[i-1]
			isWordEnd := (prev >= 'a' && prev <= 'z') || (prev >= 'A' && prev <= 'Z') || prev == '.'
			if isWordEnd {
				continue // skip this space
			}
		}
		sb.WriteByte(text[i])
	}
	return sb.String()
}
```

**The two rules that do all the work (first loop):**

1. **No space before punctuation:** if the current char is a space and the next char is `. , ! ? : ;`, skip the space. `hello ,` -> `hello,`.
2. **One space after punctuation:** after writing a punctuation mark, if the next char is *not* another punctuation mark or a space, add a space. `hello.World` -> `hello. World`. Because it checks "next is not also punctuation," groups like `...` and `!?` stay glued.

**Quotes:** `ReplaceAll("' ", "'")` kills the space after an opening quote. The second loop kills the space before a closing quote, but only when the char before that space is a letter or `.` (so it knows it's a genuine closing quote). Together: `' awesome '` -> `'awesome'`.

### `checkVowels` - a becomes an (works on words)

```go
func checkVowels(text string) string {
	words := strings.Fields(text)
	for i := 0; i < len(words)-1; i++ { // every word except the last
		if (words[i] == "a" || words[i] == "A") &&
			strings.ContainsRune("aeiouhAEIOUH", rune(words[i+1][0])) {
			words[i] += "n" // "a" -> "an", "A" -> "An"
		}
	}
	return strings.Join(words, " ")
}
```

**Say it out loud:** "For each word that is exactly `a` or `A`, I look at the first letter of the next word. If it's a vowel or `h`, I add `n`." `words[i+1][0]` is just the first letter of the next word.

## Three things so they can't surprise you

1. **`i+1 <= len(words)`** in the complex check should really be `i+1 < len(words)`. `words[i+1]` is out of range when `i+1` equals the length, so a weird input ending in `(up,` could panic. It passed audit because the tests always have a word after `(up,`. One-character fix if you want it bulletproof.
2. **`strings.Title`** is officially deprecated (still works for plain ASCII). If asked: "I know it's deprecated, but I lowercase first so it only capitalizes the first letter, and it works fine for ASCII."
3. **`fmt.Sprint(val)`** just turns the number into a string, same result as `strconv.FormatInt(val, 10)`.

## Tricky bits checklist

- A marker affects the words **before** it, reached with `words[i-j]`.
- `(up, 2)` is **two tokens** after `strings.Fields`: `(up,` and `2)`.
- Editing `words[i-j]` changes the slice **in place** (slices share memory), so no return is needed inside the loop.
- After deleting the marker, `i--` keeps the loop from skipping a word.
- `checkPunctuations` works **character by character**; the other two work word by word.

## Practice - EXPLAIN IT (answer out loud)

1. In `checkFlags`, how does the code reach the words *before* the marker, and what stops it going too far back?
2. What is the difference between the "simple" and "complex" marker, and how does the code tell them apart?
3. After applying a marker, how does the code remove it from the output? Explain `append(words[:i], words[i+2:]...)`.
4. In `checkPunctuations`, what are the two rules in the first loop, and how do groups like `...` stay together?
5. Walk through `checkVowels` on `a apple` and on `a hour`.

## Practice - WRITE IT (blank file, no peeking)

1. Write the `for j := 1; j <= count && i-j >= 0; j++` loop and explain `words[i-j]`.
2. Write the hex/bin cases of the switch from memory.
3. Write `checkVowels` from scratch.
4. Write the two-rule first loop of `checkPunctuations`.

---

# PROJECT 2 - ascii-art

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
main()                  -> parse args, read banner, build map, handle \n, call printLine
buildMap(content)       -> turns the file into a lookup table: char -> its 8 art lines
printLine(text, art)    -> prints one line of input as 8 rows of ASCII art
```

The map is the heart of this project. `buildMap` is the only function that understands the file format; `printLine` just looks characters up and renders. One job each.

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

	art := buildMap(string(data)) // build the lookup table once

	// The user types \n (backslash-n) to request a line break.
	// In the argument it is the two characters: '\' and 'n'.
	for _, segment := range strings.Split(input, "\\n") {
		printLine(segment, art)
	}
}

// buildMap turns the banner file into a lookup table:
// each character -> its 8 lines of ASCII art.
func buildMap(content string) map[rune][]string {
	lines := strings.Split(content, "\n")
	art := make(map[rune][]string)

	for c := 32; c <= 126; c++ {
		start := (c-32)*9 + 1                 // where this char's art begins
		art[rune(c)] = lines[start : start+8] // its 8 lines, stored under the char
	}
	return art
}

func printLine(text string, art map[rune][]string) {
	if text == "" {
		fmt.Println() // empty segment -> just a blank line
		return
	}
	for row := 0; row < 8; row++ { // 8 rows make up one line of art
		var line strings.Builder         // a fresh notepad for this row
		for _, char := range text {
			line.WriteString(art[char][row]) // write this char's row onto it
		}
		fmt.Println(line.String())       // tear off the finished string
	}
}
```

**Line they'll point at:** `line.WriteString(art[char][row])`. Explain: "I look up this character's 8-line art block in the map, then write row `row` of it onto a `strings.Builder`, gluing the same row from each character side by side."

**Why `strings.Builder` and not `line += ...`:** strings in Go are immutable, so `+=` in a loop throws away the old string and allocates a brand-new one every iteration. `strings.Builder` writes into one growing buffer (`WriteString`) and produces the final string once at the end (`.String()`), so it avoids all those wasted copies. Use `+=` for one or two joins; use `Builder` when appending in a loop.

**Why a map instead of indexing into the file every time:** the lookup table is built **once** in `buildMap`, so each character lookup afterwards is instant (`art[char]`). It also separates concerns: only `buildMap` knows the 9-line file layout, and `printLine` stays simple. The formula `(c-32)*9 + 1` still runs, but only while filling the table.

**`lines[start : start+8]`** is a slice holding exactly 8 lines (`start` through `start+7`). That slice is the value stored for each character.

## Tricky bits checklist

- The file **starts with a blank line** - forget the `+1` and everything shifts up by one row (you'll print garbage). This is the #1 bug.
- Input `\n` is **two characters** in the argument, so you split on `"\\n"` (escaped backslash) in Go source.
- Output is **row-major**: outer loop = 8 rows, inner loop = each character.
- Only ASCII 32-126 exist in the file. Anything outside (like a tab or emoji) won't be in the map (and would index out of range when building it) - that's why ascii-art-web later validates input.
- The map value is a **slice of 8 strings**, not one string. `art[char]` gives the whole block; `art[char][row]` gives one row.

## Practice - EXPLAIN IT

1. Derive the `(int(c) - 32) * 9 + 1` formula from scratch.
2. Why use a map at all? What does it buy you over recomputing the index every time?
3. Why `strings.Builder` instead of `line += ...` in the row loop? (Hint: strings are immutable.)
4. What type is the value stored in the map, and why a slice of 8 and not one string?
4. Why does the outer loop run exactly 8 times?
5. Why is the inner loop over characters and the outer loop over rows, not the other way round?
6. What breaks if you forget the `+ 1` when building the map?
7. How does the program produce a line break from input like `"a\nb"`?

## Practice - WRITE IT

1. Write `buildMap` from a blank file. Explain the `lines[start : start+8]` part.
2. Write `printLine` from a blank file.
3. Write just the indexing line inside `buildMap` and explain each term.
4. Modify `main` to accept a second argument for the banner name (`standard`, `shadow`, `thinkertoy`) and read `name + ".txt"`.

---

# PROJECT 3 - ascii-art-web

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

## 404 vs 405 - say the difference cleanly

- **404 Not Found** → the *path* doesn't exist (`/banana`).
- **405 Method Not Allowed** → the path exists but you used the *wrong verb* (GET on `/ascii-art`, which only takes POST).

## Tricky bits checklist

- `r.FormValue` works for POST form fields; the field name must match the `name=""` in your HTML `<input>`/`<select>`.
- Use `html/template`, **not** `text/template` - the `html` one auto-escapes and is safe for web output.
- The `PageData` struct field must be **capitalized** (`Art`, not `art`) or the template can't see it (Go exports only capitalized fields).
- Static files: `http.StripPrefix` removes `/static/` so `http.Dir("static")` finds the real file.

## Practice - EXPLAIN IT

1. Why do you check `r.URL.Path != "/"` inside `homeHandler`?
2. Difference between 404 and 405 - give an example URL for each.
3. Why must `http.Error` be called before writing any page content?
4. Why is the struct field `Art` capitalized?
5. What does `http.StripPrefix` do and why is it needed for static files?
6. Why `html/template` and not `text/template`?

## Practice - WRITE IT

1. Write `main` with all three route registrations from memory.
2. Write `homeHandler` with both guards.
3. Write `asciiArtHandler`: method guard → read form → generate → handle error → render.
4. Add a 400 response for when the text contains a non-ASCII character.

---

# Cross-cutting Go fundamentals they test on any project

These show up regardless of which project:

- **Slices share memory.** Passing a slice to a function and editing its elements changes the original. (Why `applyAction` works.)
- **Multiple return values & errors.** `v, err := f()` - always check `err`. Never ignore it in code you'll defend.
- **`strings` toolkit:** `Split`, `Fields`, `Join`, `ReplaceAll`, `ToUpper`, `ToLower`, `TrimSuffix`, `HasPrefix`, `Contains`.
- **`strconv`:** `ParseInt(s, base, bits)`, `Atoi(s)`, `FormatInt(n, base)`.
- **Runes vs bytes:** `for _, c := range s` gives runes (good for characters); `s[i]` gives a single byte.
- **Exported names:** capitalized = visible outside the package / to templates.
- **`os.Args`:** index 0 is the program name; real arguments start at index 1.

---

# Your 2-day plan (Wed-Thu, checkpoint Fri)

**Wednesday**
- Morning: go-reloaded. Read the section once, then rewrite every function from a blank file. Run it against the example table.
- Afternoon: ascii-art. Derive the formula 5 times until it's automatic. Rewrite `buildMap` and `printLine` from scratch.

**Thursday**
- Morning: ascii-art-web. Rewrite all three handlers from memory. Be able to recite the route table and status codes.
- Afternoon: mixed drilling - answer every "EXPLAIN IT" question out loud as if facing the examiner. Re-do any "WRITE IT" you stumbled on.

**Friday morning (light)**
- Skim the three "Tricky bits" checklists.
- Re-derive the ascii-art formula and re-state 404 vs 405. Don't cram new things.

**The night before:** sleep. A rested brain recalls; a tired one blanks.

---

*Built for your Friday checkpoint. You don't need to memorize this - you need to be able to rebuild it. Close the guide and prove you can.*
