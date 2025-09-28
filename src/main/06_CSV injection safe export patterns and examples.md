# CSV injection: safe export patterns and examples

## Why a leading space isn’t safe

A single space can be trimmed by importers, cleanup tools, or user actions. If the space disappears, the cell may begin with =, +, -, or @ and run as a formula. Use a single quote ' or export to a typed format like .xlsx to guarantee text.

---

## Simple rule for CSV export

- If a value starts with any of: =, +, -, @, prefix it with a single quote '
- Then apply standard CSV quoting (RFC 4180):
    - Escape embedded double quotes by doubling them
    - Wrap the field in quotes if it contains a comma, quote, or newline

---

## Example implementations

### Java

```java
public final class CsvEscaper {
  private CsvEscaper() {}

  public static String escapeForCsvCell(String s) {
    if (s == null) return "";

    // Neutralize spreadsheet formula starters
    if (s.startsWith("=") || s.startsWith("+") || s.startsWith("-") || s.startsWith("@")) {
      s = "'" + s;
    }

    // RFC 4180 quoting
    boolean needsQuotes = s.indexOf(',') >= 0 || s.indexOf('"') >= 0 || s.indexOf('\n') >= 0 || s.indexOf('\r') >= 0;
    if (s.indexOf('"') >= 0) {
      s = s.replace("\"", "\"\"");
    }
    if (needsQuotes) {
      s = '"' + s + '"';
    }
    return s;
  }
}
```

### JavaScript / TypeScript

```tsx
export function escapeForCsvCell(input: string = ""): string {
  let s = input;
  // Neutralize risky starters
  if (/^[=+\-@]/.test(s)) s = "'" + s;
  // RFC 4180 quoting
  const needsQuotes = /[",\r\n]/.test(s);
  if (s.includes('"')) s = s.replace(/"/g, '""');
  return needsQuotes ? `"${s}"` : s;
}
```

### Python

```python
def escape_for_csv_cell(s: str | None) -> str:
    if s is None:
        return ""
    if s[:1] in {"=", "+", "-", "@"}:
        s = "'" + s
    needs_quotes = any(c in s for c in [',', '"', '\n', '\r'])
    if '"' in s:
        s = s.replace('"', '""')
    return f'"{s}"' if needs_quotes else s
```

---

## End-to-end CSV row builder (JS)

```tsx
function escapeForCsvCell(input: string = ""): string {
  let s = input;
  if (/^[=+\-@]/.test(s)) s = "'" + s; // neutralize formula starters
  const needsQuotes = /[",\r\n]/.test(s);
  if (s.includes('"')) s = s.replace(/"/g, '""');
  return needsQuotes ? `"${s}"` : s;
}

function toCsvRow(values: Array<string | null | undefined>): string {
  return [values.map](http://values.map)(v => escapeForCsvCell(v ?? "")).join(',');
}

// Usage example
const rows: (string | null)[][] = [
  ["Company", "Contact"],
  ["=Acme", "[alice@example.com](mailto:alice@example.com)"],
  ["+Plus Co", "[bob@example.com](mailto:bob@example.com)"],
  ["Normal, Inc", 'He said "hello"'],
  ["Line\nBreak LLC", "@handle"],
];
const csv = [rows.map](http://rows.map)(toCsvRow).join('\n');
console.log(csv);
```

---

## Before and after examples

Input values → CSV cell output

- "=Acme" → "'=Acme"
- "+Plus Co" → "'+Plus Co"
- "-Dash Ltd" → "'-Dash Ltd"
- "@Home" → "'@Home"
- "A=OK" → "A=OK" (unchanged, since = isn’t first)
- "ACME, Inc" → """ACME, Inc""" (quoted per CSV rules)
- "He said ""hi""" → """He said """"hi""""""
- "LinenBreak" → '"LinenBreak"'

---

## Test cases to copy

```
=Acme
+Plus Co
-Dash Ltd
@Home
A=OK
ACME, Inc
He said "hi"
Line
Break
```

Expected exported cells:

```
'=Acme
'+Plus Co
'-Dash Ltd
'@Home
A=OK
"ACME, Inc"
"He said ""hi"""
"Line
Break"
```

---

## Safer alternatives to CSV

- Export to .xlsx and mark the cell type as text. This avoids formula execution without adding a leading quote.
- If staying with CSV, document the escaping rule for any downstream consumers.

---

## Notes

- Do not use a leading space for neutralization. Spaces can be trimmed.
- Only escape when the risky character is at position 0 so legitimate names like "A=OK" remain unchanged.
- Keep originals unchanged in your database; apply escaping only at export time.