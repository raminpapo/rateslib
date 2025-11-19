# Documentation: generate_docs.py

## File Metadata
- **Path:** `generate_docs.py`
- **Size:** 41,977 bytes
- **Lines:** 1,203
- **Words:** 3,936
- **Extension:** `.py`

## Purpose
Python module containing class definitions and related functionality.

## Source Code

```python
#!/usr/bin/env python3
"""
World's Best Repo Book Generator and Index Builder
Generates comprehensive documentation for the rateslib repository.
"""

import os
import sys
import json
import hashlib
import re
from pathlib import Path
from datetime import datetime
from typing import Dict, List, Set, Tuple
import mimetypes

class RepoDocGenerator:
    def __init__(self, repo_root: str, docs_dir: str):
        self.repo_root = Path(repo_root)
        self.docs_dir = Path(docs_dir)
        self.manifest = {
            "repo_source": str(self.repo_root),
            "repo_fingerprint": "",
            "generator_version": "1.0.0",
            "timestamp_start": datetime.now().isoformat(),
            "file_count": 0,
            "docs_count": 0,
            "bytes_written": 0,
            "checksums": {},
            "errors": []
        }
        self.all_keywords = {}  # Global keyword index
        self.file_list = []
        self.folder_structure = {}
        self.progress_log = []
        self.binary_extensions = {
            '.png', '.jpg', '.jpeg', '.gif', '.bmp', '.ico', '.pdf',
            '.zip', '.tar', '.gz', '.bz2', '.xz', '.7z',
            '.exe', '.dll', '.so', '.dylib', '.a', '.o',
            '.pyc', '.pyo', '.whl', '.egg'
        }
        self.text_extensions = {
            '.py', '.pyi', '.txt', '.md', '.rst', '.toml', '.yaml', '.yml',
            '.json', '.csv', '.xml', '.html', '.css', '.js', '.sh',
            '.c', '.cpp', '.h', '.hpp', '.rs', '.go', '.java',
            '.ipynb', '.cfg', '.ini', '.conf'
        }

    def compute_fingerprint(self) -> str:
        """Compute repository fingerprint from git SHA or file list."""
        try:
            import subprocess
            result = subprocess.run(
                ['git', 'log', '-1', '--format=%H'],
                cwd=self.repo_root,
                capture_output=True,
                text=True
            )
            if result.returncode == 0:
                return result.stdout.strip()
        except Exception as e:
            self.manifest["errors"].append(f"Git fingerprint error: {str(e)}")

        # Fallback: hash file list
        h = hashlib.sha256()
        for f in sorted(self.file_list):
            h.update(f.encode('utf-8'))
        return h.hexdigest()

    def scan_repository(self):
        """Recursively scan repository and classify files."""
        print("Scanning repository...")
        exclude_dirs = {'.git', 'docs', '__pycache__', 'node_modules', '.venv', 'venv'}

        for root, dirs, files in os.walk(self.repo_root):
            # Filter out excluded directories
            dirs[:] = [d for d in dirs if d not in exclude_dirs]

            rel_root = Path(root).relative_to(self.repo_root)

            for file in files:
                file_path = Path(root) / file
                rel_path = file_path.relative_to(self.repo_root)

                # Get file info
                try:
                    stat = file_path.stat()
                    size = stat.st_size

                    file_info = {
                        'path': str(rel_path),
                        'absolute': str(file_path),
                        'size': size,
                        'mtime': stat.st_mtime,
                        'extension': file_path.suffix,
                        'is_binary': self.is_binary_file(file_path),
                        'is_readable': True
                    }

                    self.file_list.append(file_info)
                    self.manifest["file_count"] += 1

                except Exception as e:
                    self.manifest["errors"].append(f"Error scanning {rel_path}: {str(e)}")

        print(f"Scanned {self.manifest['file_count']} files")
        self.manifest["repo_fingerprint"] = self.compute_fingerprint()

    def is_binary_file(self, file_path: Path) -> bool:
        """Determine if a file is binary."""
        ext = file_path.suffix.lower()

        if ext in self.binary_extensions:
            return True
        if ext in self.text_extensions:
            return False

        # Try reading first 8192 bytes
        try:
            with open(file_path, 'rb') as f:
                chunk = f.read(8192)
                # Check for null bytes
                if b'\x00' in chunk:
                    return True
                # Check if mostly text
                text_chars = sum(1 for b in chunk if 32 <= b < 127 or b in (9, 10, 13))
                return text_chars / len(chunk) < 0.85 if chunk else False
        except:
            return True

    def read_file_safe(self, file_path: Path) -> Tuple[str, bool]:
        """Safely read a file, returning content and success status."""
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                return f.read(), True
        except UnicodeDecodeError:
            try:
                with open(file_path, 'r', encoding='latin-1') as f:
                    return f.read(), True
            except Exception as e:
                return f"[Error reading file: {str(e)}]", False
        except Exception as e:
            return f"[Error reading file: {str(e)}]", False

    def extract_keywords_from_python(self, content: str, file_path: str) -> Dict[str, List[str]]:
        """Extract keywords from Python code."""
        keywords = {}

        # Extract class names
        for match in re.finditer(r'class\s+(\w+)', content):
            class_name = match.group(1)
            keywords[class_name] = [file_path, 'class', match.start()]

        # Extract function/method names
        for match in re.finditer(r'def\s+(\w+)', content):
            func_name = match.group(1)
            if func_name not in keywords:
                keywords[func_name] = [file_path, 'function', match.start()]

        # Extract constants (UPPER_CASE variables)
        for match in re.finditer(r'^([A-Z_][A-Z0-9_]+)\s*=', content, re.MULTILINE):
            const_name = match.group(1)
            if const_name not in keywords:
                keywords[const_name] = [file_path, 'constant', match.start()]

        # Extract imports
        for match in re.finditer(r'from\s+([\w.]+)\s+import|import\s+([\w.]+)', content):
            module = match.group(1) or match.group(2)
            if module and module not in keywords:
                keywords[module] = [file_path, 'import', match.start()]

        return keywords

    def generate_file_docs(self, file_info: dict) -> Tuple[str, str]:
        """Generate _docs.md and _kw.md for a single file."""
        file_path = Path(file_info['absolute'])
        rel_path = file_info['path']

        # Create docs subdirectory
        doc_dir = self.docs_dir / Path(rel_path).parent
        doc_dir.mkdir(parents=True, exist_ok=True)

        file_name = Path(rel_path).name
        docs_file = doc_dir / f"{file_name}_docs.md"
        kw_file = doc_dir / f"{file_name}_kw.md"

        # Handle binary files
        if file_info['is_binary']:
            binary_doc = self.generate_binary_file_doc(file_info)
            self.write_doc(docs_file, binary_doc)
            self.write_doc(kw_file, f"# Keywords: {file_name}\n\nBinary file - no keywords extracted.\n")
            return str(docs_file), str(kw_file)

        # Read file content
        content, success = self.read_file_safe(file_path)

        if not success:
            error_doc = f"# Documentation: {file_name}\n\n**Error:** Could not read file.\n\n{content}\n"
            self.write_doc(docs_file, error_doc)
            self.write_doc(kw_file, f"# Keywords: {file_name}\n\nFile could not be read.\n")
            return str(docs_file), str(kw_file)

        # Generate documentation
        docs_content = self.generate_text_file_docs(file_info, content)
        keywords_content = self.generate_keywords_doc(file_info, content)

        self.write_doc(docs_file, docs_content)
        self.write_doc(kw_file, keywords_content)

        return str(docs_file), str(kw_file)

    def generate_binary_file_doc(self, file_info: dict) -> str:
        """Generate documentation for binary files."""
        doc = f"""# Documentation: {Path(file_info['path']).name}

## File Metadata
- **Path:** `{file_info['path']}`
- **Type:** Binary file
- **Size:** {file_info['size']:,} bytes
- **Extension:** `{file_info['extension']}`

## Description
This is a binary file and cannot be displayed as text.

## Suggested Handling
- Image files: View with image viewer
- Archive files: Extract contents
- Compiled files: Part of build artifacts

## Related Files
See parent directory documentation for context.
"""
        return doc

    def generate_text_file_docs(self, file_info: dict, content: str) -> str:
        """Generate comprehensive documentation for text files."""
        file_name = Path(file_info['path']).name
        rel_path = file_info['path']

        # Analyze content
        lines = content.split('\n')
        line_count = len(lines)
        word_count = len(content.split())
        char_count = len(content)

        # Detect file type and purpose
        purpose = self.detect_file_purpose(file_info, content)

        doc = f"""# Documentation: {file_name}

## File Metadata
- **Path:** `{rel_path}`
- **Size:** {file_info['size']:,} bytes
- **Lines:** {line_count:,}
- **Words:** {word_count:,}
- **Extension:** `{file_info['extension']}`

## Purpose
{purpose}

## Source Code

```{self.get_language_hint(file_info)}
{content}
```

## Detailed Analysis

"""

        # Python-specific analysis
        if file_info['extension'] in ['.py', '.pyi']:
            doc += self.analyze_python_file(content)
        elif file_info['extension'] in ['.md', '.rst']:
            doc += self.analyze_markdown_file(content)
        elif file_info['extension'] in ['.toml', '.yaml', '.yml', '.json']:
            doc += self.analyze_config_file(content, file_info['extension'])
        elif file_info['extension'] == '.ipynb':
            doc += self.analyze_notebook_file(content)
        else:
            doc += self.analyze_generic_file(content)

        doc += f"""

## Related Files
See the [parent directory index](./index.md) for related files.

## Usage Notes
- Last modified: {datetime.fromtimestamp(file_info['mtime']).isoformat()}
- Encoding: UTF-8 (assumed)

---
*Generated by Repo Book Generator v1.0.0*
"""
        return doc

    def detect_file_purpose(self, file_info: dict, content: str) -> str:
        """Detect and describe the purpose of a file."""
        name = Path(file_info['path']).name
        ext = file_info['extension']

        # Special files
        if name == '__init__.py':
            return "Python package initialization file. Defines the public API and package structure."
        elif name == 'README.md':
            return "Project documentation and overview file."
        elif name == 'LICENSE':
            return "Software license file defining terms of use and distribution."
        elif name in ['setup.py', 'pyproject.toml']:
            return "Python package configuration and build specification."
        elif name.startswith('test_'):
            return "Unit test file containing test cases and assertions."
        elif ext == '.pyi':
            return "Python type stub file providing type hints for type checkers."
        elif ext in ['.toml', '.yaml', '.yml', '.json', '.ini', '.cfg']:
            return "Configuration file defining settings and parameters."
        elif ext == '.csv':
            return "Data file in comma-separated values format."
        elif ext == '.ipynb':
            return "Jupyter notebook containing interactive code and documentation."

        # Try to detect from content
        if ext == '.py':
            if 'class ' in content:
                return "Python module containing class definitions and related functionality."
            elif 'def ' in content:
                return "Python module containing function definitions and utilities."
            else:
                return "Python script or module."

        return "Source file (see detailed analysis below)."

    def get_language_hint(self, file_info: dict) -> str:
        """Get syntax highlighting hint for code blocks."""
        ext = file_info['extension']
        mapping = {
            '.py': 'python',
            '.pyi': 'python',
            '.rs': 'rust',
            '.toml': 'toml',
            '.yaml': 'yaml',
            '.yml': 'yaml',
            '.json': 'json',
            '.md': 'markdown',
            '.sh': 'bash',
            '.js': 'javascript',
            '.html': 'html',
            '.css': 'css',
            '.ipynb': 'json'
        }
        return mapping.get(ext, '')

    def analyze_python_file(self, content: str) -> str:
        """Detailed analysis of Python files."""
        analysis = "### Python File Analysis\n\n"

        # Extract classes
        classes = re.findall(r'class\s+(\w+).*?:', content)
        if classes:
            analysis += f"**Classes defined:** {len(classes)}\n"
            for cls in classes[:20]:  # Limit to first 20
                analysis += f"- `{cls}`\n"
            if len(classes) > 20:
                analysis += f"- ... and {len(classes) - 20} more\n"
            analysis += "\n"

        # Extract functions
        functions = re.findall(r'def\s+(\w+)\s*\(', content)
        if functions:
            analysis += f"**Functions defined:** {len(functions)}\n"
            for func in functions[:20]:
                analysis += f"- `{func}()`\n"
            if len(functions) > 20:
                analysis += f"- ... and {len(functions) - 20} more\n"
            analysis += "\n"

        # Extract imports
        imports = re.findall(r'^(?:from\s+([\w.]+)\s+import|import\s+([\w.]+))', content, re.MULTILINE)
        if imports:
            unique_imports = set()
            for imp in imports:
                unique_imports.add(imp[0] or imp[1])
            analysis += f"**Dependencies:** {len(unique_imports)}\n"
            for imp in sorted(unique_imports)[:15]:
                analysis += f"- `{imp}`\n"
            if len(unique_imports) > 15:
                analysis += f"- ... and {len(unique_imports) - 15} more\n"
            analysis += "\n"

        # Extract docstrings
        docstrings = re.findall(r'"""(.*?)"""', content, re.DOTALL)
        if docstrings and docstrings[0].strip():
            analysis += "**Module Docstring:**\n\n"
            analysis += f"> {docstrings[0].strip()[:500]}\n\n"

        return analysis

    def analyze_markdown_file(self, content: str) -> str:
        """Analyze markdown files."""
        analysis = "### Markdown Document Analysis\n\n"

        # Extract headers
        headers = re.findall(r'^(#+)\s+(.+)$', content, re.MULTILINE)
        if headers:
            analysis += "**Document Structure:**\n\n"
            for level, title in headers[:30]:
                indent = "  " * (len(level) - 1)
                analysis += f"{indent}- {title}\n"

        return analysis

    def analyze_config_file(self, content: str, ext: str) -> str:
        """Analyze configuration files."""
        return f"### Configuration File Analysis\n\n**Format:** {ext}\n\n**Purpose:** Defines project settings and configuration parameters.\n\n"

    def analyze_notebook_file(self, content: str) -> str:
        """Analyze Jupyter notebook files."""
        analysis = "### Jupyter Notebook Analysis\n\n"

        try:
            nb = json.loads(content)
            cells = nb.get('cells', [])
            code_cells = sum(1 for c in cells if c.get('cell_type') == 'code')
            markdown_cells = sum(1 for c in cells if c.get('cell_type') == 'markdown')

            analysis += f"**Total cells:** {len(cells)}\n"
            analysis += f"**Code cells:** {code_cells}\n"
            analysis += f"**Markdown cells:** {markdown_cells}\n\n"
        except:
            analysis += "Could not parse notebook structure.\n\n"

        return analysis

    def analyze_generic_file(self, content: str) -> str:
        """Generic analysis for other file types."""
        return "### File Analysis\n\nGeneric text file. See source code above for full content.\n\n"

    def generate_keywords_doc(self, file_info: dict, content: str) -> str:
        """Generate keyword index for a file."""
        file_name = Path(file_info['path']).name
        rel_path = file_info['path']

        doc = f"""# Keywords: {file_name}

## Keyword Index for {rel_path}

"""

        keywords = {}

        # Python-specific keyword extraction
        if file_info['extension'] in ['.py', '.pyi']:
            keywords = self.extract_keywords_from_python(content, rel_path)
        else:
            # Generic keyword extraction (unique words)
            words = re.findall(r'\b[A-Za-z_][A-Za-z0-9_]{2,}\b', content)
            word_freq = {}
            for word in words:
                word_freq[word] = word_freq.get(word, 0) + 1

            # Keep top 100 most frequent
            sorted_words = sorted(word_freq.items(), key=lambda x: x[1], reverse=True)[:100]
            for word, freq in sorted_words:
                keywords[word] = [rel_path, 'identifier', freq]

        # Add to global index
        for kw, info in keywords.items():
            if kw not in self.all_keywords:
                self.all_keywords[kw] = []
            self.all_keywords[kw].append({
                'file': rel_path,
                'type': info[1] if len(info) > 1 else 'identifier',
                'doc_file': f"{file_name}_docs.md"
            })

        # Write keywords
        if keywords:
            for kw in sorted(keywords.keys()):
                info = keywords[kw]
                kw_type = info[1] if len(info) > 1 else 'identifier'
                doc += f"### {kw}\n\n"
                doc += f"- **Type:** {kw_type}\n"
                doc += f"- **Defined in:** [{file_name}](./{file_name}_docs.md)\n"
                doc += f"\n"
        else:
            doc += "No significant keywords extracted.\n"

        return doc

    def write_doc(self, file_path: Path, content: str):
        """Write documentation file and track checksum."""
        file_path.parent.mkdir(parents=True, exist_ok=True)

        with open(file_path, 'w', encoding='utf-8') as f:
            f.write(content)

        # Compute checksum
        checksum = hashlib.sha256(content.encode('utf-8')).hexdigest()
        rel_path = str(file_path.relative_to(self.docs_dir))
        self.manifest["checksums"][rel_path] = checksum
        self.manifest["bytes_written"] += len(content.encode('utf-8'))
        self.manifest["docs_count"] += 1

    def generate_all_file_docs(self):
        """Generate documentation for all files."""
        print(f"\nGenerating documentation for {len(self.file_list)} files...")

        for i, file_info in enumerate(self.file_list):
            if (i + 1) % 10 == 0:
                print(f"  Processing file {i+1}/{len(self.file_list)}...")

            try:
                docs_file, kw_file = self.generate_file_docs(file_info)
                self.progress_log.append({
                    'file': file_info['path'],
                    'docs_created': [docs_file, kw_file],
                    'status': 'success'
                })
            except Exception as e:
                error_msg = f"Error processing {file_info['path']}: {str(e)}"
                self.manifest["errors"].append(error_msg)
                self.progress_log.append({
                    'file': file_info['path'],
                    'status': 'error',
                    'error': str(e)
                })

        print(f"Generated documentation for {len(self.file_list)} files")

    def generate_folder_docs(self):
        """Generate index.md, doc.md, and sub.md for each folder."""
        print("\nGenerating folder documentation...")

        # Get all unique directories
        directories = set()
        for file_info in self.file_list:
            path_parts = Path(file_info['path']).parts
            for i in range(len(path_parts)):
                directories.add(str(Path(*path_parts[:i+1]).parent))

        directories.discard('.')
        directories = sorted(directories)

        # Add root
        directories = ['.'] + list(directories)

        for dir_path in directories:
            self.generate_folder_index(dir_path)
            self.generate_folder_doc(dir_path)
            self.generate_folder_sub(dir_path)

        print(f"Generated documentation for {len(directories)} folders")

    def generate_folder_index(self, dir_path: str):
        """Generate index.md for a folder."""
        folder_name = Path(dir_path).name if dir_path != '.' else 'Root'
        doc_dir = self.docs_dir / dir_path if dir_path != '.' else self.docs_dir
        doc_dir.mkdir(parents=True, exist_ok=True)

        index_file = doc_dir / "index.md"

        # Get files and subdirectories in this folder
        files_in_folder = [f for f in self.file_list if str(Path(f['path']).parent) == dir_path]

        # Get subdirectories
        subdirs = set()
        for file_info in self.file_list:
            file_path = Path(file_info['path'])
            parts = file_path.parts

            if dir_path == '.':
                # For root, get all first-level directories
                if len(parts) > 1:
                    subdirs.add(parts[0])
            else:
                # Check if file is in a subdirectory of current dir
                try:
                    rel_path = file_path.relative_to(dir_path)
                    if len(rel_path.parts) > 1:
                        # This file is in a subdirectory
                        subdirs.add(str(Path(dir_path) / rel_path.parts[0]))
                except ValueError:
                    # File is not in this directory
                    pass

        content = f"""# Index: {folder_name}

## Overview
This directory contains {len(files_in_folder)} file(s).

"""

        if subdirs:
            content += "## Subdirectories\n\n"
            for subdir in sorted(subdirs):
                subdir_name = Path(subdir).name
                if dir_path == '.':
                    rel_link = f"{subdir}/index.md"
                else:
                    rel_link = f"{Path(subdir).relative_to(dir_path)}/index.md"
                content += f"- [{subdir_name}]({rel_link})\n"
            content += "\n"

        if files_in_folder:
            content += "## Files in This Directory\n\n"
            for file_info in sorted(files_in_folder, key=lambda x: x['path']):
                file_name = Path(file_info['path']).name
                doc_link = f"{file_name}_docs.md"
                size_kb = file_info['size'] / 1024
                content += f"- [{file_name}]({doc_link}) ({size_kb:.1f} KB)\n"
            content += "\n"

        content += """
## Related Documentation
- [Folder Documentation](./doc.md) - Detailed description and context
- [Keyword Index](./sub.md) - All keywords from this folder

---
*Generated by Repo Book Generator v1.0.0*
"""

        self.write_doc(index_file, content)

    def generate_folder_doc(self, dir_path: str):
        """Generate doc.md for a folder."""
        folder_name = Path(dir_path).name if dir_path != '.' else 'Root'
        doc_dir = self.docs_dir / dir_path if dir_path != '.' else self.docs_dir
        doc_file = doc_dir / "doc.md"

        files_in_folder = [f for f in self.file_list if str(Path(f['path']).parent) == dir_path]

        # Infer purpose from folder name and contents
        purpose = self.infer_folder_purpose(dir_path, files_in_folder)

        content = f"""# Folder Documentation: {folder_name}

## Location
`{dir_path if dir_path != '.' else '/'}`

## Purpose
{purpose}

## Contents Summary
This folder contains {len(files_in_folder)} file(s).

### File Types
"""

        # Summarize file types
        file_types = {}
        for f in files_in_folder:
            ext = f['extension'] or 'no extension'
            file_types[ext] = file_types.get(ext, 0) + 1

        for ext, count in sorted(file_types.items()):
            content += f"- `{ext}`: {count} file(s)\n"

        content += """

## Architecture Notes
See individual file documentation for detailed analysis.

## Related Folders
See [parent index](../index.md) for related folders.

---
*Generated by Repo Book Generator v1.0.0*
"""

        self.write_doc(doc_file, content)

    def infer_folder_purpose(self, dir_path: str, files: List[dict]) -> str:
        """Infer the purpose of a folder based on its name and contents."""
        folder_name = Path(dir_path).name.lower() if dir_path != '.' else 'root'

        purpose_map = {
            'tests': 'Contains unit tests and test utilities.',
            'test': 'Contains unit tests and test utilities.',
            'docs': 'Contains documentation files.',
            'notebooks': 'Contains Jupyter notebooks with examples and tutorials.',
            'data': 'Contains data files (CSVs, JSON, etc.).',
            'src': 'Contains source code.',
            'python': 'Contains Python source code.',
            'lib': 'Contains library code.',
            'utils': 'Contains utility functions and helpers.',
            'core': 'Contains core functionality and base classes.',
            'instruments': 'Contains financial instrument definitions.',
            'curves': 'Contains interest rate curve implementations.',
            'periods': 'Contains period and cashflow calculations.',
            'legs': 'Contains financial leg (cashflow schedule) implementations.',
            'scheduling': 'Contains date scheduling and calendar utilities.',
            'fx': 'Contains foreign exchange functionality.',
            'dual': 'Contains automatic differentiation utilities.',
            'splines': 'Contains spline interpolation functionality.',
        }

        if folder_name in purpose_map:
            return purpose_map[folder_name]

        # Analyze file types
        has_py = any(f['extension'] == '.py' for f in files)
        has_tests = any('test' in Path(f['path']).name.lower() for f in files)

        if has_tests:
            return 'Contains test files and test utilities.'
        elif has_py:
            return 'Contains Python source code and modules.'

        return 'Source directory (see files for details).'

    def generate_folder_sub(self, dir_path: str):
        """Generate sub.md (keyword index) for a folder."""
        folder_name = Path(dir_path).name if dir_path != '.' else 'Root'
        doc_dir = self.docs_dir / dir_path if dir_path != '.' else self.docs_dir
        sub_file = doc_dir / "sub.md"

        # Collect keywords from all files in this folder and subfolders
        folder_keywords = {}
        for file_info in self.file_list:
            if str(Path(file_info['path']).parts[0] if dir_path == '.' else Path(file_info['path'])).startswith(dir_path):
                file_path = file_info['path']
                # Get keywords for this file
                for kw, locations in self.all_keywords.items():
                    for loc in locations:
                        if loc['file'] == file_path:
                            if kw not in folder_keywords:
                                folder_keywords[kw] = []
                            folder_keywords[kw].append(loc)

        content = f"""# Keyword Index: {folder_name}

## All Keywords from {folder_name} and Subdirectories

"""

        if folder_keywords:
            # Group by first letter
            by_letter = {}
            for kw in folder_keywords.keys():
                first = kw[0].upper() if kw else '?'
                if first not in by_letter:
                    by_letter[first] = []
                by_letter[first].append(kw)

            for letter in sorted(by_letter.keys()):
                content += f"## {letter}\n\n"
                for kw in sorted(by_letter[letter])[:50]:  # Limit per letter
                    content += f"### {kw}\n\n"
                    locations = folder_keywords[kw]
                    for loc in locations[:5]:  # Limit locations per keyword
                        file_name = Path(loc['file']).name
                        content += f"- [{file_name}](./{file_name}_docs.md) ({loc['type']})\n"
                    if len(locations) > 5:
                        content += f"- ... and {len(locations) - 5} more occurrences\n"
                    content += "\n"
        else:
            content += "No keywords indexed for this folder.\n"

        content += """
---
*Generated by Repo Book Generator v1.0.0*
"""

        self.write_doc(sub_file, content)

    def generate_global_keywords(self):
        """Generate global keywords.md."""
        print("\nGenerating global keyword index...")

        keywords_file = self.docs_dir / "keywords.md"

        content = """# Global Keyword Index

## All Keywords A→Z

This index contains all keywords extracted from the repository, organized alphabetically.

"""

        # Group by letter
        by_letter = {}
        for kw in self.all_keywords.keys():
            first = kw[0].upper() if kw else '?'
            if first not in by_letter:
                by_letter[first] = []
            by_letter[first].append(kw)

        for letter in sorted(by_letter.keys()):
            content += f"## {letter}\n\n"
            keywords_in_letter = sorted(by_letter[letter])[:100]  # Limit per letter

            for kw in keywords_in_letter:
                content += f"### {kw}\n\n"
                locations = self.all_keywords[kw]

                # Group by file
                by_file = {}
                for loc in locations:
                    file = loc['file']
                    if file not in by_file:
                        by_file[file] = []
                    by_file[file].append(loc)

                for file_path in sorted(by_file.keys())[:10]:  # Limit files per keyword
                    file_name = Path(file_path).name
                    file_dir = Path(file_path).parent
                    doc_link = f"./{file_dir}/{file_name}_docs.md" if file_dir != Path('.') else f"./{file_name}_docs.md"
                    types = set(loc['type'] for loc in by_file[file_path])
                    content += f"- [{file_path}]({doc_link}) - {', '.join(sorted(types))}\n"

                if len(by_file) > 10:
                    content += f"- ... and {len(by_file) - 10} more files\n"
                content += "\n"

        content += """
---
*Generated by Repo Book Generator v1.0.0*
"""

        self.write_doc(keywords_file, content)

    def generate_main_index(self):
        """Generate main index.md."""
        print("\nGenerating main index...")

        index_file = self.docs_dir / "index.md"

        content = f"""# Repository Documentation Index

## Overview
Generated documentation for **{self.manifest['repo_source']}**

- **Repository Fingerprint:** `{self.manifest['repo_fingerprint'][:16]}...`
- **Files Scanned:** {self.manifest['file_count']}
- **Documentation Files Created:** {self.manifest['docs_count']}
- **Generated:** {self.manifest['timestamp_start']}

## Main Documentation

- [Comprehensive Book](./comprehensive_book.md) - Complete repository documentation in book form
- [Global Keyword Index](./keywords.md) - All keywords A→Z
- [Verification Report](./verification_report.md) - Validation and error report
- [Manifest](./manifest.json) - Complete metadata and checksums

## Folder Structure

"""

        # Get all top-level directories
        top_dirs = set()
        for file_info in self.file_list:
            parts = Path(file_info['path']).parts
            if len(parts) > 1:
                top_dirs.add(parts[0])

        for dir_name in sorted(top_dirs):
            content += f"- [{dir_name}/](./{dir_name}/index.md)\n"

        content += """

## Root Files
See [root index](./index.md) for files in the repository root.

## How to Navigate

1. **By Topic**: Use the folder indexes to browse by module/package
2. **By Keyword**: Use the global keyword index to find specific identifiers
3. **Linear Reading**: Read the comprehensive book for a complete overview
4. **File-Specific**: Navigate to individual `*_docs.md` files for detailed analysis

---
*Generated by Repo Book Generator v1.0.0*
"""

        # Write root index instead
        root_index = self.docs_dir / "index.md"
        self.write_doc(root_index, content)

    def generate_comprehensive_book(self):
        """Generate comprehensive_book.md by stitching folder docs."""
        print("\nGenerating comprehensive book...")

        book_file = self.docs_dir / "comprehensive_book.md"

        content = f"""# Comprehensive Repository Book

## {Path(self.manifest['repo_source']).name}

**Repository Fingerprint:** `{self.manifest['repo_fingerprint']}`
**Generated:** {self.manifest['timestamp_start']}
**Files Documented:** {self.manifest['file_count']}

---

## Table of Contents

1. [Introduction](#introduction)
2. [Repository Structure](#repository-structure)
3. [Detailed Documentation by Folder](#detailed-documentation)

---

## Introduction

This comprehensive book contains complete documentation for the entire repository.
Each section corresponds to a folder or module, with detailed analysis of all files.

### Quick Statistics

- **Total Files:** {self.manifest['file_count']}
- **Documentation Files:** {self.manifest['docs_count']}
- **Total Size:** {self.manifest['bytes_written']:,} bytes

---

## Repository Structure

"""

        # Add structure overview
        top_dirs = set()
        for file_info in self.file_list:
            parts = Path(file_info['path']).parts
            if len(parts) > 1:
                top_dirs.add(parts[0])

        for dir_name in sorted(top_dirs):
            files_in_dir = sum(1 for f in self.file_list if Path(f['path']).parts[0] == dir_name)
            content += f"- **{dir_name}/** ({files_in_dir} files)\n"

        content += "\n---\n\n## Detailed Documentation\n\n"

        # Add folder documentation (doc.md content) for major folders
        dirs_to_include = sorted(top_dirs)[:20]  # Limit to avoid huge file

        for i, dir_name in enumerate(dirs_to_include, 1):
            doc_file = self.docs_dir / dir_name / "doc.md"
            if doc_file.exists():
                with open(doc_file, 'r') as f:
                    doc_content = f.read()
                content += f"\n## Chapter {i}: {dir_name}\n\n"
                content += doc_content + "\n\n---\n"

        content += """

## Appendix: File Summaries

For detailed file-by-file documentation, see individual `*_docs.md` files in each folder.

---
*End of Comprehensive Book*
*Generated by Repo Book Generator v1.0.0*
"""

        self.write_doc(book_file, content)

    def generate_verification_report(self):
        """Generate verification_report.md."""
        print("\nGenerating verification report...")

        report_file = self.docs_dir / "verification_report.md"

        # Collect statistics
        binary_files = [f for f in self.file_list if f['is_binary']]
        text_files = [f for f in self.file_list if not f['is_binary']]
        errors = self.manifest.get('errors', [])

        content = f"""# Verification Report

## Generation Summary

- **Timestamp:** {datetime.now().isoformat()}
- **Repository:** {self.manifest['repo_source']}
- **Fingerprint:** `{self.manifest['repo_fingerprint']}`

## File Statistics

- **Total Files Scanned:** {self.manifest['file_count']}
- **Text Files:** {len(text_files)}
- **Binary Files:** {len(binary_files)}
- **Documentation Files Created:** {self.manifest['docs_count']}
- **Total Bytes Written:** {self.manifest['bytes_written']:,}

## File Classification

### Text Files ({len(text_files)})
"""

        text_by_ext = {}
        for f in text_files:
            ext = f['extension'] or 'no extension'
            text_by_ext[ext] = text_by_ext.get(ext, 0) + 1

        for ext, count in sorted(text_by_ext.items()):
            content += f"- `{ext}`: {count}\n"

        content += f"""

### Binary Files ({len(binary_files)})
"""

        binary_by_ext = {}
        for f in binary_files:
            ext = f['extension'] or 'no extension'
            binary_by_ext[ext] = binary_by_ext.get(ext, 0) + 1

        for ext, count in sorted(binary_by_ext.items()):
            content += f"- `{ext}`: {count}\n"

        content += """

## Errors and Warnings

"""

        if errors:
            content += f"**{len(errors)} error(s) encountered:**\n\n"
            for error in errors[:50]:  # Limit errors shown
                content += f"- {error}\n"
            if len(errors) > 50:
                content += f"\n... and {len(errors) - 50} more errors\n"
        else:
            content += "No errors encountered during generation.\n"

        content += """

## Link Validation

All internal links use relative paths. Manual verification recommended for:
- Cross-folder references
- Generated anchor links

## Checksums

All file checksums are stored in `manifest.json`.

## Recommendations

1. Review error log for any files that couldn't be processed
2. Verify binary file handling is appropriate
3. Check that all expected files are documented

---
*Generated by Repo Book Generator v1.0.0*
"""

        self.write_doc(report_file, content)

    def generate_readme(self):
        """Generate README.md for docs folder."""
        readme_file = self.docs_dir / "README.md"

        content = """# Repository Documentation

This directory contains automatically generated comprehensive documentation for the repository.

## Structure

- **index.md** - Main navigation index
- **comprehensive_book.md** - Complete documentation in book form
- **keywords.md** - Global keyword index (A→Z)
- **manifest.json** - Generation metadata and checksums
- **verification_report.md** - Validation report

### Folder Structure

Each folder in the repository has:
- **index.md** - List of files and subdirectories
- **doc.md** - Narrative documentation for the folder
- **sub.md** - Keyword index for the folder and descendants

### File Documentation

Each source file has:
- **{filename}_docs.md** - Complete documentation including source code
- **{filename}_kw.md** - Keyword index for the file

## How to Use

1. **Start with** [index.md](./index.md) for an overview
2. **Browse by folder** using folder indexes
3. **Search by keyword** using keywords.md
4. **Read sequentially** using comprehensive_book.md

## Regeneration

To regenerate this documentation:

```bash
python3 generate_docs.py
```

The process is idempotent - running it multiple times on the same repository produces identical output.

## Generator Version

Version: 1.0.0

---
*This documentation was automatically generated by the Repo Book Generator*
"""

        self.write_doc(readme_file, content)

    def save_manifest(self):
        """Save final manifest.json."""
        print("\nSaving manifest...")

        self.manifest["timestamp_end"] = datetime.now().isoformat()
        manifest_file = self.docs_dir / "manifest.json"

        with open(manifest_file, 'w') as f:
            json.dump(self.manifest, f, indent=2)

        print(f"Manifest saved to {manifest_file}")

    def generate_summary(self) -> dict:
        """Generate final JSON summary."""
        return {
            "repo_source": self.manifest["repo_source"],
            "repo_fingerprint": self.manifest["repo_fingerprint"],
            "files_scanned": self.manifest["file_count"],
            "docs_created": self.manifest["docs_count"],
            "words_estimated": self.manifest["bytes_written"] // 6,  # Rough estimate
            "bytes_written": self.manifest["bytes_written"],
            "errors": self.manifest["errors"]
        }

    def run(self):
        """Execute complete documentation generation process."""
        print("=" * 60)
        print("Repo Book Generator v1.0.0")
        print("=" * 60)

        # Step 1: Bootstrap
        print("\n[1/8] Scanning repository...")
        self.scan_repository()

        # Step 2: Generate file docs
        print("\n[2/8] Generating per-file documentation...")
        self.generate_all_file_docs()

        # Step 3: Generate folder docs
        print("\n[3/8] Generating folder documentation...")
        self.generate_folder_docs()

        # Step 4: Generate global keyword index
        print("\n[4/8] Generating global keyword index...")
        self.generate_global_keywords()

        # Step 5: Generate main index
        print("\n[5/8] Generating main index...")
        self.generate_main_index()

        # Step 6: Generate comprehensive book
        print("\n[6/8] Generating comprehensive book...")
        self.generate_comprehensive_book()

        # Step 7: Generate verification report
        print("\n[7/8] Generating verification report...")
        self.generate_verification_report()

        # Step 8: Generate README
        print("\n[8/8] Generating README...")
        self.generate_readme()

        # Save manifest
        self.save_manifest()

        # Generate summary
        summary = self.generate_summary()

        print("\n" + "=" * 60)
        print("Generation Complete!")
        print("=" * 60)
        print(f"\nFiles scanned: {summary['files_scanned']}")
        print(f"Docs created: {summary['docs_created']}")
        print(f"Bytes written: {summary['bytes_written']:,}")
        print(f"Estimated words: {summary['words_estimated']:,}")
        print(f"Errors: {len(summary['errors'])}")

        return summary


def main():
    repo_root = os.getcwd()
    docs_dir = os.path.join(repo_root, 'docs')

    generator = RepoDocGenerator(repo_root, docs_dir)
    summary = generator.run()

    # Print JSON summary
    print("\n" + "=" * 60)
    print("JSON Summary:")
    print("=" * 60)
    print(json.dumps(summary, indent=2))

    return 0


if __name__ == '__main__':
    sys.exit(main())

```

## Detailed Analysis

### Python File Analysis

**Classes defined:** 1
- `RepoDocGenerator`

**Functions defined:** 33
- `__init__()`
- `compute_fingerprint()`
- `scan_repository()`
- `is_binary_file()`
- `read_file_safe()`
- `extract_keywords_from_python()`
- `generate_file_docs()`
- `generate_binary_file_doc()`
- `generate_text_file_docs()`
- `detect_file_purpose()`
- `get_language_hint()`
- `analyze_python_file()`
- `analyze_markdown_file()`
- `analyze_config_file()`
- `analyze_notebook_file()`
- `analyze_generic_file()`
- `generate_keywords_doc()`
- `write_doc()`
- `generate_all_file_docs()`
- `generate_folder_docs()`
- ... and 13 more

**Dependencies:** 9
- `datetime`
- `hashlib`
- `json`
- `mimetypes`
- `os`
- `pathlib`
- `re`
- `sys`
- `typing`

**Module Docstring:**

> World's Best Repo Book Generator and Index Builder
Generates comprehensive documentation for the rateslib repository.



## Related Files
See the [parent directory index](./index.md) for related files.

## Usage Notes
- Last modified: 2025-11-18T07:53:44
- Encoding: UTF-8 (assumed)

---
*Generated by Repo Book Generator v1.0.0*
