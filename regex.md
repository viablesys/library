# regex Reference

Regular expressions for Rust with guaranteed linear-time matching. Current version: **1.11.x**.

Docs: <https://docs.rs/regex/latest/regex/>

---

## 1. Installation

```toml
[dependencies]
regex = "1.11"
```

| Feature | Purpose |
|---------|---------|
| `std` (default) | Standard library integration |
| `perf` (default) | All performance features (SIMD, literal optimizations, DFA) |
| `unicode` (default) | Full Unicode support (categories, scripts, case folding) |
| `perf-literal` | Literal substring optimizations (Aho-Corasick, memchr) |
| `perf-dfa` | DFA for faster matching on applicable patterns |

For minimal binary size (no Unicode, no SIMD):

```toml
regex = { version = "1.11", default-features = false, features = ["std", "perf"] }
```

For even smaller builds, consider `regex-lite` — same API, ~200KB smaller, no Unicode tables, no SIMD. Sufficient when patterns are ASCII-only.

---

## 2. Regex Basics

### Compilation

Regex compilation is expensive (~microseconds to milliseconds). Never compile inside a loop or hot path.

```rust
use regex::Regex;

// BAD: recompiles every call
fn is_secret_bad(input: &str) -> bool {
    let re = Regex::new(r"sk-[A-Za-z0-9]{20,}").unwrap();
    re.is_match(input)
}

// GOOD: compile once, reuse
use std::sync::LazyLock;

static SECRET_RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"sk-[A-Za-z0-9]{20,}").unwrap()
});

fn is_secret(input: &str) -> bool {
    SECRET_RE.is_match(input)
}
```

`std::sync::LazyLock` (stable since Rust 1.80) replaces `once_cell::sync::Lazy`. No external dependency needed.

### Core Methods

```rust
use regex::Regex;

let re = Regex::new(r"(\d{4})-(\d{2})-(\d{2})").unwrap();
let hay = "Date: 2026-02-14 and 2026-03-01";

// Test for match (bool)
assert!(re.is_match(hay));

// First match location
let m = re.find(hay).unwrap();
assert_eq!(m.as_str(), "2026-02-14");
assert_eq!(m.start(), 6);
assert_eq!(m.end(), 16);

// Capture groups
let caps = re.captures(hay).unwrap();
assert_eq!(&caps[1], "2026");
assert_eq!(&caps[2], "02");
assert_eq!(&caps[3], "14");

// All non-overlapping matches
let dates: Vec<&str> = re.find_iter(hay).map(|m| m.as_str()).collect();
assert_eq!(dates, vec!["2026-02-14", "2026-03-01"]);

// All captures
for caps in re.captures_iter(hay) {
    let (_, [y, m, d]) = caps.extract();
    println!("{y}-{m}-{d}");
}
```

### Named Captures

```rust
let re = Regex::new(r"(?P<key>\w+)=(?P<val>\S+)").unwrap();
let caps = re.captures("token=abc123").unwrap();
assert_eq!(&caps["key"], "token");
assert_eq!(&caps["val"], "abc123");
```

---

## 3. Replacement

### Static Replacement

```rust
use regex::Regex;

let re = Regex::new(r"sk-[A-Za-z0-9]{20,}").unwrap();

// Replace first match
let result = re.replace("key: sk-proj-abcdefghijklmnopqrstuvwxyz", "[REDACTED]");
assert_eq!(result, "key: [REDACTED]");

// Replace all matches
let input = "keys: sk-abc12345678901234567890 and sk-xyz09876543210987654321";
let result = re.replace_all(input, "[REDACTED]");
assert_eq!(result, "keys: [REDACTED] and [REDACTED]");
```

`replace` and `replace_all` return `Cow<str>` — borrows the original string if no match found, allocates only when replacement occurs.

```rust
use std::borrow::Cow;

let result = re.replace_all("no secrets here", "[REDACTED]");
match result {
    Cow::Borrowed(_) => println!("no allocation"),  // this path
    Cow::Owned(_) => println!("allocated new string"),
}
```

### Closure-Based Replacement

For dynamic replacement logic (e.g., preserving partial matches, logging):

```rust
let re = Regex::new(r"(?i)(password|token|secret)\s*[=:]\s*(\S+)").unwrap();

let result = re.replace_all(
    "password=hunter2 and token=abc123",
    |caps: &regex::Captures| {
        // Keep the key, redact the value
        format!("{}=[REDACTED]", &caps[1])
    },
);
assert_eq!(result, "password=[REDACTED] and token=[REDACTED]");
```

### Back-References in Replacement

```rust
let re = Regex::new(r"(\w+)@(\w+)\.com").unwrap();
// $1, $2 refer to capture groups
let result = re.replace_all("user@example.com", "$1@[REDACTED]");
assert_eq!(result, "user@[REDACTED]");
```

Use `${name}` for named captures, `$1` for indexed. Literal `$` requires `$$`.

---

## 4. RegexSet — Multi-Pattern Matching

`RegexSet` compiles multiple patterns into a single automaton. Tests all patterns in a single pass — much faster than iterating individual `Regex` objects when the common case is "no match."

### Fast Rejection Pattern

```rust
use regex::RegexSet;

let set = RegexSet::new(&[
    r"AKIA[0-9A-Z]{16}",                    // AWS key
    r"ghp_[A-Za-z0-9]{36}",                 // GitHub PAT
    r"sk-ant-[A-Za-z0-9\-]{20,}",           // Anthropic key
    r"sk-[A-Za-z0-9]{20,}",                 // OpenAI key
    r"(?i)bearer\s+[A-Za-z0-9_\-.~+/]{20,}",// Bearer token
]).unwrap();

// Fast path: single pass, returns bool
if set.is_match("no secrets here") {
    // won't reach here — common case is fast rejection
}

// Which patterns matched?
let matches = set.matches("my key is sk-ant-api03-abcdefghijklmnopqrstuvwxyz");
assert!(matches.matched(2));  // Anthropic pattern
assert!(matches.matched(3));  // also matches generic sk- pattern
```

### RegexSet Limitations

`RegexSet` answers "which patterns matched?" but does **not** provide:
- Match positions (where in the string)
- Capture groups
- Replacement

For replacement, you need individual `Regex` objects. The efficient pattern: use `RegexSet` for fast rejection, fall back to individual `Regex` only on match.

### RegexSet + Individual Regex for Redaction

This is the core pattern for ADR-007's secret filtering:

```rust
use regex::{Regex, RegexSet};
use std::sync::LazyLock;

struct SecretFilter {
    set: RegexSet,
    patterns: Vec<Regex>,
    placeholder: &'static str,
}

impl SecretFilter {
    fn new() -> Self {
        // Order matters: more-specific patterns first.
        // During sequential replacement, first match wins for overlapping patterns.
        let pattern_strings = vec![
            r"sk-ant-[A-Za-z0-9\-]{20,}",   // Anthropic (before generic sk-)
            r"sk-[A-Za-z0-9]{20,}",          // OpenAI / generic
            r"AKIA[0-9A-Z]{16}",             // AWS
            r"ghp_[A-Za-z0-9]{36}",          // GitHub PAT
            r"(?i)bearer\s+[A-Za-z0-9_\-.~+/]{20,}=*",
        ];

        let set = RegexSet::new(&pattern_strings).unwrap();
        let patterns: Vec<Regex> = pattern_strings.iter()
            .map(|p| Regex::new(p).unwrap())
            .collect();

        Self { set, patterns, placeholder: "[REDACTED]" }
    }

    fn redact(&self, input: &str) -> (String, bool) {
        // Fast path: single-pass check across all patterns
        if !self.set.is_match(input) {
            return (input.to_string(), false);
        }

        // Slow path: only reached when a secret is detected
        let mut output = input.to_string();
        let mut redacted = false;
        // Use matched() to only run replacement for patterns that matched
        let matches = self.set.matches(input);
        for idx in matches.into_iter() {
            let replaced = self.patterns[idx].replace_all(&output, self.placeholder);
            if let std::borrow::Cow::Owned(new) = replaced {
                output = new;
                redacted = true;
            }
        }
        (output, redacted)
    }
}

// Singleton — compile once at startup
static FILTER: LazyLock<SecretFilter> = LazyLock::new(SecretFilter::new);
```

**Key insight:** `RegexSet::is_match()` is the fast path — one pass over the input against all patterns simultaneously. On the common path (no secret), this is the only cost. Individual `Regex::replace_all` runs only for patterns that actually matched, identified by `RegexSet::matches().into_iter()`.

---

## 5. Pattern Syntax

### Common Constructs

| Pattern | Meaning |
|---------|---------|
| `.` | Any char except newline (unless `(?s)` flag) |
| `\d`, `\D` | Digit / non-digit (Unicode-aware by default) |
| `\w`, `\W` | Word char / non-word char (Unicode-aware) |
| `\s`, `\S` | Whitespace / non-whitespace |
| `\b` | Word boundary |
| `[A-Za-z0-9]` | Character class |
| `[^...]` | Negated character class |
| `a{3,5}` | 3 to 5 repetitions |
| `a{20,}` | 20 or more (no upper bound) |
| `(?i)` | Case-insensitive flag |
| `(?x)` | Verbose mode (whitespace and `#` comments) |
| `(?:...)` | Non-capturing group |
| `(?P<name>...)` | Named capture group |

### Flags

Flags can be set inline or globally:

```rust
// Inline flag — case-insensitive for part of pattern
let re = Regex::new(r"(?i)bearer\s+\S+").unwrap();

// Verbose mode — whitespace and comments
let re = Regex::new(r"(?x)
    AKIA          # prefix
    [0-9A-Z]{16}  # 16 uppercase alphanumeric chars
").unwrap();

// Multiple flags
let re = Regex::new(r"(?is)begin.*end").unwrap(); // case-insensitive + dotall
```

### What's NOT Supported

The `regex` crate guarantees linear-time matching by excluding:
- Backreferences (`\1`)
- Lookahead/lookbehind (`(?=...)`, `(?<=...)`)
- Atomic groups / possessive quantifiers

If you need these, consider `fancy-regex` crate (wraps `regex` with backtracking fallback).

---

## 6. Performance

### Compilation Cost

| Operation | Typical cost |
|-----------|-------------|
| `Regex::new` (simple pattern) | 5-50 us |
| `Regex::new` (complex pattern) | 50-500 us |
| `RegexSet::new` (12 patterns) | 100-1000 us |
| `is_match` (short string, no match) | 50-200 ns |
| `is_match` (short string, match) | 100-500 ns |
| `replace_all` (no match) | 50-200 ns (returns Cow::Borrowed) |
| `replace_all` (with match) | 500 ns - 5 us (allocation + copy) |

### Rules of Thumb

1. **Compile once.** Use `LazyLock` or struct fields. Never compile in loops.
2. **`RegexSet` for multi-pattern fast rejection.** One pass instead of N sequential `is_match` calls.
3. **`replace_all` is zero-cost on no-match.** Returns `Cow::Borrowed` — no allocation.
4. **Avoid `.*` when possible.** Prefer `[^...]+` or `\S+` for tighter matching.
5. **Use `(?x)` for complex patterns.** Readability prevents bugs.
6. **`regex-lite` for binary size.** Same API, ~200KB smaller, ASCII-only.

### Memory

A compiled `Regex` holds a DFA/NFA state machine. Typical sizes:
- Simple pattern: 1-5 KB
- Complex pattern with alternations: 5-50 KB
- `RegexSet` with 12 patterns: 10-100 KB

All are negligible for a CLI tool. Only matters if compiling thousands of patterns.

---

## 7. Error Handling

`Regex::new` returns `Result<Regex, regex::Error>`. Patterns compiled from static strings can safely `unwrap()` (if the pattern is wrong, it's a compile-time bug). User-supplied patterns must handle errors:

```rust
// Static patterns — unwrap is fine (developer error if wrong)
let re = Regex::new(r"AKIA[0-9A-Z]{16}").unwrap();

// User-supplied patterns — handle errors
fn compile_user_pattern(pattern: &str) -> Result<Regex, String> {
    Regex::new(pattern).map_err(|e| format!("Invalid pattern '{}': {}", pattern, e))
}
```

`RegexSet::new` returns `Result<RegexSet, regex::Error>`. If any pattern is invalid, the entire set fails to compile.

---

## 8. Testing Patterns

### Table-Driven Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_secret_detection() {
        let filter = SecretFilter::new();

        // Must redact (true positives)
        let must_catch = vec![
            ("AKIAIOSFODNN7EXAMPLE", "AWS key"),
            ("ghp_ABCDEFGHIJKLMNOPQRSTUVWXYZabcdef1234", "GitHub PAT"),
            ("sk-ant-api03-abcdefghijklmnopqrstuvwxyz", "Anthropic key"),
            ("sk-proj-abcdefghijklmnopqrstuvwxyz1234", "OpenAI key"),
            ("Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.abc.def", "JWT"),
            ("password=hunter2", "generic assignment"),
        ];
        for (input, label) in must_catch {
            let (output, redacted) = filter.redact(input);
            assert!(redacted, "should redact {label}: {input}");
            assert!(output.contains("[REDACTED]"), "output should contain placeholder for {label}");
            assert!(!output.contains(input), "original should not appear for {label}");
        }

        // Must NOT redact (false positives to avoid)
        let must_pass = vec![
            ("password_validator", "variable name"),
            ("git://github.com/user/repo", "git URL"),
            ("file:///tmp/token_cache/data", "file path"),
            ("sk-iplink", "short non-key"),
            ("the token count was 150", "natural language"),
        ];
        for (input, label) in must_pass {
            let (_, redacted) = filter.redact(input);
            assert!(!redacted, "should NOT redact {label}: {input}");
        }
    }

    #[test]
    fn test_partial_redaction_preserves_context() {
        let filter = SecretFilter::new();
        let (output, redacted) = filter.redact("Set key to sk-proj-abcdefghijklmnopqrstuvwxyz1234 in config");
        assert!(redacted);
        assert!(output.starts_with("Set key to "));
        assert!(output.ends_with(" in config"));
        assert!(output.contains("[REDACTED]"));
    }
}
```

### Testing RegexSet Consistency

Verify that `RegexSet` and individual patterns agree:

```rust
#[test]
fn test_set_and_individual_agree() {
    let filter = SecretFilter::new();
    let test_cases = vec![
        "AKIAIOSFODNN7EXAMPLE",
        "no secrets here",
        "sk-ant-api03-abcdefghijklmnopqrstuvwxyz",
    ];

    for input in test_cases {
        let set_matched = filter.set.is_match(input);
        let any_individual = filter.patterns.iter().any(|p| p.is_match(input));
        assert_eq!(set_matched, any_individual,
            "RegexSet and individual patterns disagree on: {input}");
    }
}
```

---

## 9. Common Pitfalls

### Unicode Surprises

`\w` matches Unicode word characters by default, not just `[a-zA-Z0-9_]`. For ASCII-only matching, use explicit character classes:

```rust
// Matches Unicode letters (accented chars, CJK, etc.)
let re = Regex::new(r"\w+").unwrap();
assert!(re.is_match("cafe")); // matches

// ASCII-only word chars
let re = Regex::new(r"[A-Za-z0-9_]+").unwrap();
```

### Greedy vs Lazy Matching

```rust
// Greedy (default) — matches longest possible
let re = Regex::new(r"token=\S+").unwrap();
let m = re.find("token=abc123&next=value").unwrap();
assert_eq!(m.as_str(), "token=abc123&next=value"); // too much!

// Fix: use non-greedy or tighter class
let re = Regex::new(r"token=[^\s&]+").unwrap();
let m = re.find("token=abc123&next=value").unwrap();
assert_eq!(m.as_str(), "token=abc123");
```

### Escaping Special Characters

Regex metacharacters must be escaped: `. * + ? ( ) [ ] { } | \ ^ $`

```rust
// Match literal dots
let re = Regex::new(r"file\.txt").unwrap();

// Use regex::escape for dynamic strings
let user_input = "foo.bar[0]";
let escaped = regex::escape(user_input);
assert_eq!(escaped, r"foo\.bar\[0\]");
let re = Regex::new(&escaped).unwrap();
```

### Overlapping Patterns in Sequential Replacement

When replacing sequentially, earlier patterns can alter the string such that later patterns no longer match:

```rust
let patterns = vec![
    Regex::new(r"sk-ant-[A-Za-z0-9\-]{20,}").unwrap(),
    Regex::new(r"sk-[A-Za-z0-9]{20,}").unwrap(),
];

let input = "sk-ant-api03-abcdefghijklmnopqrst";

// If sk-ant- runs first: "sk-ant-api03-abcdefghijklmnopqrst" -> "[REDACTED]" (correct)
// If sk- runs first: matches "sk-ant-api03-abcdefghijklmnopqrst" -> "[REDACTED]" (also works, but less precise)

// Rule: put more-specific patterns before broader ones in the list
```

Using `RegexSet::matches().into_iter()` to drive replacement (as shown in section 4) avoids this issue — only patterns that matched the original input are applied, and the broader pattern's replacement is harmless after the specific one already redacted.

---

## 10. regex-lite Alternative

For binary size-sensitive applications where patterns are ASCII-only:

```toml
[dependencies]
regex-lite = "0.1"
```

Same API as `regex` (drop-in replacement for most uses):

```rust
use regex_lite::Regex;

let re = Regex::new(r"sk-[A-Za-z0-9]{20,}").unwrap();
assert!(re.is_match("sk-proj-abcdefghijklmnopqrstuvwxyz1234"));
```

**Trade-offs:**
- ~200KB smaller binary (no Unicode tables, no SIMD)
- No `\w`, `\d` Unicode classes (ASCII equivalents only)
- No `RegexSet` (not available in regex-lite)
- Slower matching on some patterns (no DFA, no literal optimizations)

For nmem's secret filtering, full `regex` is preferred — `RegexSet` is essential for the fast-rejection pattern, and binary size is not a constraint.
