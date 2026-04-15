# #️⃣ Hash Tables — Presentation

An interactive slide deck covering hash functions, collision resolution strategies, open addressing, cuckoo hashing, Swiss Tables, and concurrent hash maps. Aimed at mid-level software engineers.

## ▶ [Open Presentation](https://brendanjameslynskey.github.io/Hash_Tables/index.html)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic |
|---|-------|
| 01 | Dictionary / map ADT |
| 02 | Direct addressing — when keys are small integers |
| 03 | Hash functions — division, multiplication, practical |
| 04 | Universal hashing — defeating adversarial inputs |
| 05 | Separate chaining |
| 06 | Open addressing — linear & quadratic probing |
| 07 | Double hashing |
| 08 | Load factor & performance |
| 09 | Resizing & rehashing strategies |
| 10 | Robin Hood hashing |
| 11 | Cuckoo hashing — worst-case O(1) lookup |
| 12 | Hopscotch hashing |
| 13 | Swiss Table (Google's flat_hash_map) |
| 14 | Perfect hashing — two-level scheme |
| 15 | Hash tables in practice — Python & Java |
| 16 | Hash tables in practice — Go & C++ |
| 17 | Thread-safe hash maps — lock striping, CAS, RCU |
| 18 | Applications & limitations |
| 19 | Summary & further reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | append `?print-pdf` to URL |

---

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) (Monokai) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm.

---

## References

- Cormen, T.H. et al. *Introduction to Algorithms* (CLRS), 4th ed. MIT Press, 2022
- Knuth, D.E. *The Art of Computer Programming*, Vol. 3: Sorting and Searching. Addison-Wesley
- Pagh, R. & Rodler, F.F. "Cuckoo Hashing." *Journal of Algorithms*, 2004
- Herlihy, M. & Shavit, N. *The Art of Multiprocessor Programming*. Morgan Kaufmann
- [Abseil Swiss Tables Design Notes](https://abseil.io/about/design/swisstables)
- Celis, P. et al. "Robin Hood Hashing." *FOCS*, 1985

## License

Educational use. Code examples provided as-is. Standards references are to publicly available documentation.
