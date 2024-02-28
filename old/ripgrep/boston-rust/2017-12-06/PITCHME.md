# ripgrep

<small>https://github.com/BurntSushi/ripgrep

<small>Boston Rust, 2017-12-07</small>

<small>Andrew Gallant</small>

---

#### what is a grep?

```
$ grep andrew /etc/passwd
andrew:x:1000:100::/home/andrew:/bin/bash
```

Note: Globally search a REgular expression and Print

---

#### what is ripgrep?

```
$ rg 'fn is_empty'
termcolor/src/lib.rs
789:    pub fn is_empty(&self) -> bool {

globset/src/lib.rs
272:    pub fn is_empty(&self) -> bool {

ignore/src/overrides.rs
60:    pub fn is_empty(&self) -> bool {

ignore/src/gitignore.rs
152:    pub fn is_empty(&self) -> bool {

ignore/src/types.rs
422:    pub fn is_empty(&self) -> bool {
```

<small><ul>
  <li class="fragment">Recursively search a directory.</li>
  <li class="fragment">Ignores files that match your `.gitignore`.</li>
</ul></small>

---

#### history

- In the beginning, Ken Thompson gave us grep. First released in 1974. |
- ack, by Andy Lester, first released 2005 |
- The Silver Searcher, by Geoff Greer, first released 2011 |

Note:

Globally search a REgular expression and Print

ag was originally called "bta" or "better than ack."

---

#### why rust?

---

#### ~~why rust?~~

#### Origin Story

<ul>
  <li class="fragment">ripgrep started life as a benchmark tool called xrep</li>
  <li class="fragment">Used to test match overhead with known fast tools (like GNU grep)</li>
  <li class="fragment">After lots of optimization, turned out to be competitive</li>
  <li class="fragment">Hmmm...</li>
</ul>

---

#### recursive grep in under 20 lines

```rust
let pattern = env::args().nth(1).unwrap();
let re = Regex::new(&pattern)?;
for result in WalkDir::new("./") {
    let entry = result?;
    if !entry.file_type().is_file() {
        continue;
    }

    let mut file = File::open(entry.path())?;
    let mut contents = String::new();
    if file.read_to_string(&mut contents).is_ok() {
        for line in contents.lines() {
            if re.is_match(&line) {
                let path = entry.path().display();
                println!("{}:{}", path, line);
            }
        }
    }
}
```

Note:

ripgrep is over 12,000 lines of code.

---

#### UTF-8

<small>What is the problem with the following block of code?</small>

```rust
let mut file = File::open(entry.path())?;
let mut contents = String::new();
if file.read_to_string(&mut contents).is_ok() {
```

<small>
  <ul>
    <li class="fragment">Fails if the file isn't valid UTF-8.</li>
    <li class="fragment">We want to search files that aren't valid UTF-8.</li>
    <li class="fragment">regex crate can search &[u8] or &str.</li>
    <li class="fragment">Bonus: no UTF-8 check needed!</li>
  </ul>
</small>

---

#### Reading a file line-by-line

<small>Simple, but slow:</small>

```rust
for line in contents.lines() {
    if re.is_match(&line) {
        let path = entry.path().display();
        println!("{}:{}", path, line);
    }
}
```

<small>
  <ul>
    <li class="fragment">Spends time looking for line terminators.</li>
    <li class="fragment">Runs the regex engine for every line.</li>
    <li class="fragment">Instead: search large chunks at once</li>
    <li class="fragment">Bonus: memory maps (be careful, can be slower!)</li>
  </ul>
</small>

Note:

What happens when the regex can itself match a new line?

---

#### gitignore

<small>We need a lot more than this:</small>

```rust
for result in WalkDir::new("./") {
    let entry = result?;
    if !entry.file_type().is_file() {
        continue;
    }
```

<small><ul>
  <li class="fragment">.gitignore</li>
  <li class="fragment">.git/info/exclude</li>
  <li class="fragment">$HOME/.config/git/ignore</li>
  <li class="fragment">.ignore</li>
  <li class="fragment">Additional ignore files via --ignore-file</li>
  <li class="fragment">Include/exclude globs via -g/--glob</li>
</ul></small>

+++?code=mysql-gitignore&lang=text&title=but some gitignores are crazy

+++

#### gitignore optimizations

```rust
enum GlobSetMatchStrategy {
    // A simple literal like `foo/bar`
    Literal(LiteralStrategy),
    // A basename literal like `foo`
    BasenameLiteral(BasenameLiteralStrategy),
    // A simple extension rule like `*.foo`
    Extension(ExtensionStrategy),
    // A simple prefix rule like `foo*`
    Prefix(PrefixStrategy),
    /// A simple suffix rule like `*foo`
    Suffix(SuffixStrategy),
    /// An extension is required, like `foo*.bar`
    RequiredExtension(RequiredExtensionStrategy),
    /// Everything else: a regex
    Regex(RegexSetStrategy),
}
```

---

#### Parallelism, part 1: directory traversal

<small>Our simple grep tool uses single threaded traversal:</small>

```rust
for result in WalkDir::new("./") {
    let entry = result?;
```

<small><ul>
  <li class="fragment">Simple model is a search worker pool. (How ag does it.)</li>
  <li class="fragment">But: directory traversal is parallelizable!</li>
  <li class="fragment">Challenge: How do you know when you're done?</li>
  <li class="fragment">Challenge: Share gitignore compilation.</li>
</ul></small>

---

#### Parallelism, part 2: printing

<small>Printing in single threaded code is easy:</small>

```rust
let path = entry.path().display();
println!("{}:{}", path, line);
```

<small><ul>
  <li class="fragment">Many options leads to a complicated printer.</li>
  <li class="fragment">When using parallelism, prevent interleaving.</li>
  <li class="fragment">Deterministic ordering.</li>
</small></ul>

---

#### Parallelism, part 3: let there be color

<small><ul>
  <li class="fragment">ANSI escape sequences are no problem.</li>
  <li class="fragment">On Windows, coloring requires synchronous use of console APIs.</li>
  <li class="fragment">Solution: record color information and positions.</li>
  <li class="fragment">Works in cygwin too!</li>
</ul></small>

---

#### literal optimizations

```
baz
baz\w+
\w+foo
\w+\s+baz\s+\w+
```

<small><ul>
  <li class="fragment">Search for a simple literal like "foo" before using the regex engine.</li>
  <li class="fragment">Carefully tuned substring search is much faster than a full regex engine.</li>
  <li class="fragment">Prefix, suffix and interior literals.</li>
  <li class="fragment">A common literal might make this "optimization" slower!</li>
  <li class="fragment">Use memchr as much as possible, feed it rare bytes.</li>
</ul></small>

---

#### regex engine

```
if re.is_match(&line) {
```

<small><ul>
  <li class="fragment">Reduce match overhead as much as possible. (Lots of inlining.)</li>
  <li class="fragment">Use a DFA. (Built lazily during search.)</li>
  <li class="fragment">Build UTF-8 decoding into the DFA.</li>
</ul></small>

Note:

For UTF-8 decoding, show side-by-side comparison for
LC_ALL=en_US.UTF-8 grep -E -n --color=auto '(\w{5}\s+){6}\w{5}' OpenSubtitles2016.raw.ru

---

#### future

<small><ul>
  <li class="fragment">Persistent configuration file.</li>
  <li class="fragment">libripgrep</li>
  <li class="fragment">Multiline search.</li>
  <li class="fragment">Search compressed files automatically.</li>
</ul></small>

---

#### why rust?

<small><ul>
  <li class="fragment">Never had to debug memory related bug.</li>
  <li class="fragment">Everything pretty much just worked on Windows, Mac and Linux.</li>
  <li class="fragment">(Errrmmm... except for colors.)</li>
  <li class="fragment">Fast.</li>
  <li class="fragment">Crate ecosystem.</li>
</ul></small>

---

#### want more?

Benchmarks: http://blog.burntsushi.net/ripgrep/

Repository: https://github.com/BurntSushi/ripgrep
