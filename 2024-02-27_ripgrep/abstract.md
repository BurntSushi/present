Title: Engineering a Fast Grep

Abstract:
Grep is a command line tool for searching the contents of files for a
regular expression and printing lines that match. Tools like grep are
commonly used to search plain text such as log files and code
repositories in an ad hoc manner. As the size of code repositories has
increased, the utility of faster searching has increased. In this talk
I'll discuss a tool I built called ripgrep, used in VS Code's "Find in
Files" feature, and the engineering challenges of making it fast. I'll
talk about parallelism, SIMD, amortizing allocation, glob matching and
ripgrep's regex engine.
