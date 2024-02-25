<!--
theme: default
paginate: true
style: |
  section {
    justify-content: start;
  }
-->


# ripgrep

https://github.com/BurntSushi/ripgrep

2024-02-27

Andrew Gallant

---

# What is `grep`?

```
$ grep andrew /etc/passwd
andrew:x:1000:100::/home/andrew:/bin/bash
```

**G**lobally search a **RE**gular expression and **P**rint

---

# What is ripgrep?

```
$ rg 'fn line_buffer'
crates/searcher/src/searcher/mod.rs
216:    fn line_buffer(&self) -> LineBuffer {
```

* Recursively search a directory.
* By default, it ignores:
  * Files that match your `.gitignore`, `.ignore` and `.rgignore`.
  * Hidden files and directories.
  * Binary files.
* `rg -uuu` disables all automatic filtering.

---

# Slide2

Here's some Rust code:

```rust
fn main() -> anyhow::Result<()> {
    writeln!(&mut std::io::stdout(), "Hello, world!")?;
    Ok(())
}
```

---

# Slide 3
