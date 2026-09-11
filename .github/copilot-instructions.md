# Writing Style

- Never use em-dashes (—).
- Never use semicolons (;) as punctuation.
- Place commas and periods outside closing quotation marks (logical punctuation), not inside (American style). Write "SEPT", not "SEPT,".
- Use single spaces between sentences, not double spaces.
- Use simple, direct wording. Prefer precise terms over vague language.
- Strongly prefer avoiding unnecessary hyphenated compound modifiers. Write "device state" instead
  of "device-state" when the meaning is clear. Preserve established terms such as P-state.
- Prefer to avoid the possessive 's when a rephrase reads well. Write "the TDVPR page of the vCPU"
  instead of "the vCPU's TDVPR page". This is a preference, not a hard rule.
- State the exact component, event, condition, or behavior.
- Avoid filler words such as "generally", "basically", "roughly", "seems", or
  "typically" unless the statement is truly uncertain.
- Prefer concrete, testable statements over broad claims.
- Use the terms already used in this repository. Prefer established project terminology over
  new wording.
- Do not invent new terms unless the project already uses them or the term is necessary for
  clarity.

# Markdown Style

- Always surround lists with blank lines.
- Hard-wrap lines at 100 characters. Fill each line as close to 100 characters as possible
  before wrapping to the next line.
- Use backticks for CPU instructions and API/tool names (e.g., `CPUID`, `PCONFIG`,
  `TDH.MEM.PAGE.ADD`). Do not backtick mechanisms or concepts (e.g., IBPB, MK-TME).
- Use backticks for x86 exception mnemonics, with or without a qualifier (e.g., `#VE`, `#GP(0)`,
  `#VE(CONFIG_PARAVIRT)`, `#PF`, `#MC`).
- Do not use backticks inside headings, inside link text, or inside bold spans, to avoid
  nested formatting.
- When bolding a phrase with punctuation, leave the punctuation outside the bold markers. Write
  `**MSR accesses**.`, not `**MSR accesses.**`.
- Capitalize section headings in title case (e.g., "### What the MC Checks", not
  "### What the MC checks").
- Always specify a language tag on fenced code blocks. Never use a bare ` ``` ` with no tag.
  Common tags: `python`, `c`, `bash`, `yaml`, `markdown`, `text`. Use `text` for plain output or
  command examples.
- Do not split markdown links or backtick quotations across multiple lines. Keep them complete on
  a single line to maintain readability and proper formatting.
- In pseudo-graphics (ASCII diagrams, trees, flow sketches), use ASCII-only symbols and avoid
  non-ASCII box-drawing or arrow characters for mobile compatibility.
