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
- Prefer plural subjects for general statements. Write "Metadata pages contain" instead of "Each
  metadata page contains". This is a preference, not a hard rule.
- State the exact component, event, condition, or behavior.
- Avoid filler words such as "generally", "basically", "roughly", "seems", or
  "typically" unless the statement is truly uncertain.
- Prefer concrete, testable statements over broad claims.
- Use the terms already used in this repository. Prefer established project terminology over
  new wording.
- Do not invent new terms unless the project already uses them or the term is necessary for
  clarity.

# Paragraph Structure

- Write one idea per paragraph. Multiple ideas are fine only if they are very closely related.
- Keep paragraphs short. The preferred maximum is 7 lines of up to 100 characters each. Split
  longer paragraphs, usually at an idea boundary.
- Bridge paragraphs. A paragraph should not switch topics abruptly. Start it with a sentence that
  links to the previous one, or end the previous one with a pointer to the next topic.
- Bridging is not always possible or needed. Skip it when adjacent paragraphs discuss very related
  things.

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
- Start the text after a colon with a capital letter in list items that have a label, a term, or a
  link before the colon (e.g., `- **Transport**: Delivers messages.`). Do not capitalize after a
  colon in the middle of a sentence.
- Always specify a language tag on fenced code blocks. Never use a bare ` ``` ` with no tag.
  Common tags: `python`, `c`, `bash`, `yaml`, `markdown`, `text`. Use `text` for plain output or
  command examples.
- Keep Markdown links, backtick quotations, bold spans, and italic spans on one line. Never split
  them across a line break when hard-wrapping, even if the line becomes shorter than 100
  characters. If a link would exceed the 100-character line limit, put the complete link on its own
  line, even if that line exceeds the limit.
- In pseudo-graphics (ASCII diagrams, trees, flow sketches), use ASCII-only symbols and avoid
  non-ASCII box-drawing or arrow characters for mobile compatibility.

# Notes

- Use a Note for side information that the reader does not need to follow the main flow. Typical
  cases are caveats, exceptions, mappings to TDX specification terminology, clarifications of table
  or list content, and pointers to related sections. Put the main line of explanation in regular
  paragraphs.
- Write a Note as its own paragraph that starts with `**Note**:`. Keep the colon outside the bold
  span and start the text with a capital letter.
- Place a Note right after the paragraph, list, or table it qualifies, with blank lines around it.
  Do not embed a Note inside a list item.
- Keep a Note to one idea, usually up to 4 lines. If it needs more, write regular paragraphs.
- Use these variants when they fit:
  - `**Linux Note**:` for Linux kernel behavior.
  - `**Terminology Note**:` for how this document uses a term or how it maps to other terminology.
  - `**Important**:` for a caveat that can lead to wrong conclusions or wrong code.
  - `**Note 1**:`, `**Note 2**:` when several Notes follow each other for the same text.
  - `**Notes**:` or `**Linux Notes**:` followed by a list when several short Notes apply together.

# Links

- Use a link where the reader may want to stop reading and jump to understand a concept first.
- Do not link forward to a subsection of the current section. The reader should not jump away while
  reading the current text. Either give enough general information for the current context and let
  the subsections add details, or write "see below" or "explained below".
- Do not link the same target more than once in a section or subsection. In a large section with
  large subsections, linking once per subsection is fine. The criterion is whether a reader can
  start at this subsection without reading the entire section, for example after following a link
  from elsewhere.
- Prefer referencing a section (usually the `##` level) over a subsection. This is a preference,
  not a hard rule.
