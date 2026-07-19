# Writing Style

- Never use em-dashes (—).
- Never use semicolons (;) as punctuation.
- Place commas and periods outside closing quotation marks (logical punctuation), not inside (American style). Write "SEPT", not "SEPT,".
- Use single spaces between sentences, not double spaces.

# Markdown Style

- Always surround lists with blank lines.
- Hard-wrap lines at 100 characters. Fill each line as close to 100 characters as possible
  before wrapping to the next line.
- Use backticks for CPU instructions and API/tool names (e.g., `CPUID`, `PCONFIG`,
  `TDH.MEM.PAGE.ADD`). Do not backtick mechanisms or concepts (e.g., IBPB, MK-TME).
- Capitalize section headings in title case (e.g., "### What the MC Checks", not
  "### What the MC checks").
- Always specify a language tag on fenced code blocks. Never use a bare ` ``` ` with no tag.
  Common tags: `python`, `c`, `bash`, `yaml`, `markdown`, `text`. Use `text` for plain output or
  command examples.
- Do not split markdown links or backtick quotations across multiple lines. Keep them complete on
  a single line to maintain readability and proper formatting.
