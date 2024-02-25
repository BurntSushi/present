<!--
theme: default
paginate: true
backgroundColor: aqua
-->


# Slide 1

* Foo
* Bar
* Baz

---

# Slide A

- Foo
- Bar
- Baz

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

<section id="1">
  <h1>Bullet list</h1>
  <ul>
    <li>One</li>
    <li>Two</li>
    <li>Three</li>
  </ul>
</section>
<section id="2" data-marpit-fragments="3">
  <h1>Fragmented list</h1>
  <ul>
    <li data-marpit-fragment="1">One</li>
    <li data-marpit-fragment="2">Two</li>
    <li data-marpit-fragment="3">Three</li>
  </ul>
</section>
