# File Extension Validation Examples (JS, Python, Java)

### Overview

Quick examples to extract a file extension and allow only pdf, jpe, png. Normalize to lowercase. Reject missing extensions.

> Regex to reuse: `/(pdf|jpe|png)$/i` matched after a literal dot like `/\.(pdf|jpe|png)$/i`.
> 

---

### JavaScript

```jsx
function getExtension(filename) {
	// Handles names with multiple dots, ignores leading dot in hidden files
	const i = filename.lastIndexOf('.');
	if (i <= 0 || i === filename.length - 1) return ''; // no ext or ends with dot
	return filename.slice(i + 1).toLowerCase();
}

function isAllowedFile(filename) {
	const ext = getExtension(filename);
	return ['pdf', 'jpe', 'png'].includes(ext);
}

// Regex alternative
function isAllowedFileRegex(filename) {
	return /\.(pdf|jpe|png)$/i.test(filename);
}

// Examples
console.log(isAllowedFile('report.PDF'));     // true
console.log(isAllowedFile('image.jpeg'));     // false (only "jpe" allowed)
console.log(isAllowedFile('photo.jpe'));      // true
console.log(isAllowedFile('.env'));           // false
console.log(isAllowedFile('archive.tar.gz')); // false
```

---

### Python

```python
import os

def get_extension(filename: str) -> str:
	# os.path.splitext splits off the last extension
	_, ext = os.path.splitext(filename)
	if not ext or ext == '.':  # no extension
		return ''
	return ext[1:].lower()  # drop leading "."

def is_allowed_file(filename: str) -> bool:
	ext = get_extension(filename)
	return ext in {'pdf', 'jpe', 'png'}

# Examples
print(is_allowed_file('report.PDF'))     # True
print(is_allowed_file('image.jpeg'))     # False (only "jpe" allowed)
print(is_allowed_file('photo.jpe'))      # True
print(is_allowed_file('.env'))           # False
print(is_allowed_file('archive.tar.gz')) # False
```

---

### Java

```java
import java.util.Locale;

public class FileUtil {
	public static String getExtension(String filename) {
		if (filename == null) return "";
		int lastDot = filename.lastIndexOf('.');
		// no dot, dot is first char (hidden file), or ends with dot
		if (lastDot <= 0 || lastDot == filename.length() - 1) return "";
		return filename.substring(lastDot + 1).toLowerCase(Locale.ROOT);
	}

	public static boolean isAllowedFile(String filename) {
		String ext = getExtension(filename);
		return ext.equals("pdf") || ext.equals("jpe") || ext.equals("png");
	}

	// Regex alternative
	public static boolean isAllowedFileRegex(String filename) {
		if (filename == null) return false;
		return filename.matches("(?i).*\\.(pdf|jpe|png)$");
	}

	public static void main(String[] args) {
		System.out.println(isAllowedFile("report.PDF"));     // true
		System.out.println(isAllowedFile("image.jpeg"));     // false (only "jpe" allowed)
		System.out.println(isAllowedFile("photo.jpe"));      // true
		System.out.println(isAllowedFile(".env"));           // false
		System.out.println(isAllowedFile("archive.tar.gz")); // false
	}
}
```

---

### Notes

- To allow jpeg or jpg as well, add them to the allow list.
- For uploads, also validate MIME type server-side. Extensions can be spoofed.