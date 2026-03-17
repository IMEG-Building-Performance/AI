---
name: skyspark-html-email-generation
description: "Guide for generating HTML email report functions in SkySpark's Axon language. Use this skill whenever the user wants to create an Axon function that sends HTML emails, build email templates in SkySpark, generate data quality reports as email, or work with triple-quoted strings containing HTML in Axon. Also trigger when the user mentions email_meterDataQuality, emailSend, or asks about Outlook-compatible email templates in SkySpark."
---

# SkySpark HTML Email Generation

This skill covers how to write Axon functions that generate and send HTML email reports from SkySpark view data. It captures hard-won lessons about SkySpark's triple-quoted string parser and Outlook-compatible email patterns.

## Architecture Pattern

The standard pattern for an email generation function in Axon:

1. **Function signature**: `(recipients, dates) => do ... end` — takes a list of email addresses and a DateSpan
2. **Data fetch**: Call a view function (e.g., `view_meterDataQuality`) to get a table of data
3. **Data processing**: Split/filter rows using `findAll`, compute counts
4. **Date formatting**: Use `.format("MMM D, YYYY")` etc. for display strings
5. **HTML template**: Define the full HTML document as a triple-quoted string with `{{PLACEHOLDER}}` tokens
6. **Row building**: Loop over data rows with `.each()` to build HTML table row strings via concatenation
7. **Placeholder replacement**: Chain `.replace("{{TOKEN}}", value)` calls to fill the template
8. **Send**: Call `emailSend(recipients, subject, html)`

## Triple-Quoted String Rules (Critical)

SkySpark's Axon parser is strict about triple-quoted strings (`"""`). These rules were learned through multiple iterations of parser errors:

### No blank lines inside the string block
Every line between the opening `"""` and closing `"""` must contain content or whitespace-only indentation. A completely empty line (zero characters) will cause a parse error. If you need visual separation, use a line with spaces matching the surrounding indentation level, or use HTML comments like `<!-- spacer -->`.

### Indentation must be consistent
All lines inside a triple-quoted string must share the same base indentation. The closing `"""` sets the indentation baseline. Every content line must be indented at or beyond that baseline. The SkySpark parser strips the common leading whitespace.

### Opening line starts content immediately
The first line of content goes on the same line as the opening `"""`:
```axon
html: """<!DOCTYPE html>
         <html>
         ...
       """
```

Do NOT put a newline after the opening `"""` — start the HTML content on the same line.

### Match a known working indentation pattern
When in doubt, follow this exact pattern from the working `email_meterDataQuality` function:
- Variable assignment with `"""` and first line of HTML on same line
- Content lines indented consistently (all aligned to the same column)
- Closing `"""` on its own line, indented less than or equal to the content

## Outlook / MSO Compatibility

Email HTML must work in Outlook (Word rendering engine), Gmail, and Apple Mail. Key constraints:

- **Inline CSS only** — no `<style>` blocks, no external stylesheets
- **Table-based layout** — use `<table role="presentation">` for structure, not divs/flexbox/grid
- **MSO conditional comments** — include OfficeDocumentSettings XML for Outlook pixel density:
  ```html
  <!--[if mso]>
  <noscript>
      <xml>
          <o:OfficeDocumentSettings>
              <o:PixelsPerInch>96</o:PixelsPerInch>
          </o:OfficeDocumentSettings>
      </xml>
  </noscript>
  <![endif]-->
  ```
- **XML namespaces** on the `<html>` tag: `xmlns:v="urn:schemas-microsoft-com:vml" xmlns:o="urn:schemas-microsoft-com:office:office"`
- **Font stacks**: Use web-safe fonts with fallbacks (e.g., `Georgia,'Times New Roman',serif` for headings, `Arial,Helvetica,sans-serif` for body)
- **HTML entities** for special characters: `&#8211;` (en dash), `&#8212;` (em dash), `&nbsp;`

## Building Data Table Rows in Axon

Since triple-quoted strings can't contain dynamic data, build table rows via string concatenation in a loop:

```axon
tableRows: ""
rows.each((row, index) => do
  bgColor: if (index.isOdd) "#ffffff" else "#fdf8f8"
  borderStyle: if (index < rows.size - 1) "border-bottom:1px solid #f0e0e0;" else ""

  tableRows = tableRows +
    "<tr>" +
      "<td style=\"padding:8px 10px;background-color:" + bgColor + ";" + borderStyle + "\">" + row->site + "</td>" +
      "<td style=\"padding:8px 10px;background-color:" + bgColor + ";" + borderStyle + "\">" + row->meter + "</td>" +
      "<td style=\"padding:8px 10px;background-color:" + bgColor + ";" + borderStyle + "text-align:center;\">" + row->usage.format + "</td>" +
    "</tr>\n"
end)
```

Key details:
- Use `\"` to escape quotes inside concatenated strings
- Alternate row colors using `index.isOdd`
- Skip bottom border on the last row
- Access grid columns with the `->` dereference operator (e.g., `row->site`)

## Placeholder Replacement Pattern

Use `{{DOUBLE_BRACE}}` tokens in the HTML template, then chain `.replace()` calls:

```axon
html = html.replace("{{MONTH_LABEL}}", monthLabel)
           .replace("{{CRITICAL_COUNT}}", criticalCount.toStr)
           .replace("{{ROWS}}", tableRows)
```

Convert numbers to strings with `.toStr` before replacement.

## Email Structure Template

A well-structured email report typically has:

1. **Header bar** — brand color background with title and date range
2. **KPI cards** — summary statistics in colored cards with left border accent
3. **Data tables** — one per category, with colored header rows and alternating row backgrounds
4. **Footer** — dark background with generation date and source info

## Existing Reference: email_meterDataQuality

The `email_meterDataQuality` function in `clients/Wash U/Meter Data Quality Email/` is a complete working example. It:
- Queries `view_meterDataQuality(@nav:equip.all, dates, 1)` which returns columns: `site`, `meter`, `usage`
- Splits into Critical (usage == 0) and Additional (usage < 0) categories
- Uses Wash U brand colors: red `#a51417`, green `#215732`
- Sends via `emailSend(recipients, subject, html)`

## Future Work

The next planned enhancement is adding a button to the email that links directly back to the SkySpark view, allowing recipients to click through to the live data.
