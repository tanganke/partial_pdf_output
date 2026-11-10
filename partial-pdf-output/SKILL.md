---
name: partial-pdf-output
description: Make one LaTeX source produce two PDFs - the whole document and only a later part of it - with the kept pages identical to the full document (same layout, same page numbers, cross-references intact). Use when someone wants to drop the first pages of a compiled document, split one source into a full and a partial PDF, or when a pdfpages page-extraction wrapper fails.
---

# Partial PDF output from a single LaTeX source

Sometimes one document must ship twice: complete, and with only its later part — a chapter issued on its own, the second volume of a manual, the closing sections handed to a different audience. The parts are usually not independent: the later part cross-references labels, equations or tables defined in the earlier one, so it cannot simply be moved into a document of its own. What is wanted is the tail of the compiled document, unchanged, in a file of its own.

## Why `pdfpages` alone does not do this

- `\includepdf` inserts pages of an **existing** PDF into the document being typeset. It cannot remove pages the same run has already produced, so "put `\includepdf` at the end and drop the front" does not exist.
- The usual two-file trick — compile `full.tex`, then compile a shell with `\includepdf[pages=15-]{full.pdf}` — only works when the shell can read `full.pdf`. With `-output-directory=build` (or `.compile`) it cannot: TeX finds only project files and TEXINPUTS entries, so `\IfFileExists{build/full.pdf}` is **false** although the file is there, and the bare file name is not found either. The trick then needs a manual copy of the compiled PDF into the project after every build.
- Keep it as an option when the PDF is already available in the project (a one-off assembly, or parts that do not cross-reference), but not as the general answer.

## The approach that works in one source

Two entry points, one real document:

- `full.tex` — the document, unchanged.
- `part.tex` — a two-line entry point:

```latex
\def\onlypart{1}
\input{full.tex}
```

The jobname differs, so the `.aux`, `.bbl`, `.toc` and PDF of the two entries never collide.

### 1. Drop the pages the output should not contain

In the preamble of `full.tex`, guarded so that a normal compile is untouched:

```latex
\newif\ifkeeppages                  % false: pages are dropped
\ifdefined\onlypart
  \AddToHook{shipout/before}{\ifkeeppages\else\DiscardShipoutBox\fi}
\fi
```

`shipout/before` runs for every page just before it is written to the PDF; `\DiscardShipoutBox` (LaTeX kernel, 2020 or newer, no package needed) voids that page. The page counter still advances, so the kept pages keep the numbers they have in the full document, and floats, page breaks and layout are exactly those of the full build.

### 2. Keep the auxiliary file complete

This is the part that is easy to miss. A non-immediate `\write` is not executed when it is issued: it is a whatsit placed in the page box and runs at shipout. `\label`, `\cite` and `\@writefile` (table-of-contents entries) all go through `\protected@write`, i.e. through deferred writes — so every entry belonging to a discarded page is lost, and every `\ref` to it turns into `??` even though the text was typeset.

Make those writes immediate while the dropped pages are being built (put the block **after** every package — `hyperref` in particular — so that nothing redefines `\protected@write` again behind it):

```latex
\ifdefined\onlypart
  \makeatletter
  \let\orig@protected@write\protected@write
  \long\def\protected@write#1#2#3{%
    \begingroup
      \let\thepage\relax
      #2%
      \let\protect\@unexpandable@protect
      \edef\reserved@a{\immediate\write#1{#3}}\reserved@a
    \endgroup
    \if@nobreak\ifvmode\nobreak\fi\fi}
  \makeatother
\fi
```

Only the `\write` → `\immediate\write` changes; the rest is the kernel definition.

### 3. Switch at the cut and restore the kernel write

Where the kept part begins:

```latex
\clearpage                          % the last dropped page leaves here
\keeppagestrue                      % from now on pages are kept
\ifdefined\onlypart
  \makeatletter\let\protected@write\orig@protected@write\makeatother
\fi
```

The `\clearpage` must come **before** the flag is raised, or the last page of the dropped part is kept. Restoring the deferred write matters as much: an immediate write records `\thepage` at the moment of writing, which is not the page a float or a contents line finally lands on. Without the restore, every table-of-contents page number on the kept pages collapses to the same value — a silent, plausible-looking corruption.

## Pitfalls

- Guard every piece with the switch; the full build must never see the redefinition of `\protected@write`.
- `\DiscardShipoutBox` is only valid inside `shipout/before`.
- Compilation does not get cheaper: the dropped part is still fully typeset, floats and all.
- Hyperlink destinations that lived on dropped pages are gone: pdfTeX logs `dest ... does not exist` and latexmk reports "failed to resolve N reference(s)". The PDF is correct for print; links inside the kept part still work. Turn link creation off for that build if the warnings must go.
- Bookmarks of the dropped part stay in the PDF outline and lead nowhere; suppress them for that build if the outline is part of the deliverable.
- `\pageref` to a label in the dropped part now returns the page where it was written, which for drift-prone floats may be off by a page — harmless when the target is not printed, worth knowing when it is.

## Verification

- Compile both entry points; both must report zero errors and no LaTeX "undefined reference" warnings.
- Compare the partial PDF with the full one: page count, the folio of the first and last page, and one middle page — text and layout should be identical, page for page.
- Check the table of contents on the first kept page: the page numbers must differ per entry, not all show the same one.
- The only warnings left should be pdfTeX destination warnings for links into the dropped part.

## When not to use this

- Parts that are self-contained and share no labels: compile the second part as its own document, with `xr`/`xr-hyper` if it needs a few references.
- A PDF assembled from existing PDFs: `pdfpages` with the source copied into the project is simpler and more honest.
