"""Read-only FEX search. Python 3.9+, standard library only.

Table/alias findings are file-level candidates, NOT resolved SQL lineage.
Includes, metadata, query scopes and runtime variables are not evaluated.
"""
import os
import re
from pathlib import Path


def read_source(path):
    raw = path.read_bytes()
    if raw.startswith((b'\xff\xfe', b'\xfe\xff')):
        return raw.decode('utf-16'), 'UTF-16'
    try:
        return raw.decode('utf-8-sig'), 'UTF-8'
    except UnicodeDecodeError:
        return raw.decode('cp1252'), 'Windows-1252 (fallback; verify text)'


def mask_noncode(text):
    # Preserve offsets and line numbers while hiding comments/string literals.
    pattern = r"(?m)^\s*-\*[^\n]*|--[^\n]*|/\*[\s\S]*?\*/|'(?:''|[^'])*'"
    return re.sub(pattern, lambda m: re.sub(r'[^\n]', ' ', m[0]), text)


def declarations(text):
    code = mask_noncode(text)
    part = r'(?:\[[^\]\n]+\]|"[^"\n]+"|[\w&][\w&$#/\\-]*)'
    name = part + r'(?:\.' + part + r')*'
    patterns = [
        ('SQL source candidate', rf'\b(?:FROM|JOIN)\s+({name})(?:\s+(?:AS\s+)?([A-Za-z_]\w*))?'),
        ('WebFOCUS logical file', rf'\b(?:TABLE|TABLEF|GRAPH|DEFINE|MODIFY)\s+FILE\s+({name})'),
        ('WebFOCUS TAG candidate', rf'\bIN\s+({name})\s+TAG\s+([A-Za-z_]\w*)'),
        ('HOLD output (not a base table)', rf'\bHOLD\s+AS\s+({name})'),
    ]
    reserved = set('WHERE ON JOIN LEFT RIGHT FULL INNER OUTER CROSS GROUP ORDER HAVING UNION EXCEPT INTERSECT END AS TO IN TAG WITH SET SELECT LIMIT FETCH OFFSET QUALIFY'.split())
    found = []
    for kind, pattern in patterns:
        for m in re.finditer(pattern, code, re.I):
            alias = m[2] if m.lastindex == 2 else ''
            if alias and alias.upper() in reserved:
                alias = ''
            number = text.count('\n', 0, m.start()) + 1
            evidence = ' '.join(text[m.start():m.end()].split())
            found.append((number, kind, m[1], alias or '', evidence))
    return sorted(set(found))


def matching_lines(lines, terms):
    return [(i, [t for t in terms if t.casefold() in line.casefold()])
            for i, line in enumerate(lines)
            if any(t.casefold() in line.casefold() for t in terms)]


def scan(folder, terms, context=3):
    discovered = scanned = reports = hits = 0
    errors = []
    def walk_error(error):
        errors.append(str(error))
    for directory, dirs, names in os.walk(folder, onerror=walk_error, followlinks=False):
        dirs.sort()
        for filename in sorted(names):
            if not filename.lower().endswith('.fex'):
                continue
            discovered += 1
            path = Path(directory) / filename
            try:
                text, encoding = read_source(path)
            except (OSError, UnicodeError) as error:
                errors.append(f'{path}: {error}')
                continue
            scanned += 1
            lines = text.splitlines()
            matches = matching_lines(lines, terms)
            if not matches:
                continue
            reports += 1
            hits += len(matches)
            print('\n' + '=' * 78)
            print(f'REPORT: {path}\nENCODING: {encoding}')
            print('TABLE / ALIAS EVIDENCE (whole file; verify query scope):')
            evidence = declarations(text)
            for number, kind, table, alias, snippet in evidence:
                print(f'  L{number}: {kind}: {table}' + (f' | alias/tag: {alias}' if alias else ''))
                print(f'      {snippet}')
            if not evidence:
                print('  Unresolved: no supported declaration found in this file.')
            includes = [(i + 1, line) for i, line in enumerate(lines)
                        if re.match(r'^\s*-INCLUDE\b', line, re.I)]
            for number, line in includes:
                print(f'  Include to review separately, L{number}: {line.strip()}')
            for index, matched_terms in matches:
                print(f'\nMATCH L{index + 1} | terms: {", ".join(matched_terms)}')
                print('Usage evidence (original source; > marks matching line):')
                for j in range(max(0, index - context), min(len(lines), index + context + 1)):
                    print(f'{">" if j == index else " "} {j + 1:5}: {lines[j]}')
    print('\n' + '=' * 78)
    print(f'FEX files discovered: {discovered} | Read: {scanned}')
    print(f'Matching reports: {reports} | Matching lines: {hits}')
    print('Literal, case-insensitive search; comments are included in matches.')
    print('Table evidence is heuristic. Logical files/CTEs/views may not be physical tables.')
    print('No includes executed, no metadata read, no reports run or changed.')
    if errors:
        print(f'INCOMPLETE SCAN: {len(errors)} read/directory errors:')
        for error in errors:
            print('  ' + error)
    return discovered, scanned, reports, hits, errors


def main():
    folder = Path(input('Reports folder (paste full path): ').strip().strip('"')).expanduser()
    if not folder.is_dir():
        print('Folder not found. Copy the full path from File Explorer.')
        return
    print('Enter one literal search term per line. Blank line starts the search.')
    terms = []
    while True:
        term = input('Search text: ').strip()
        if not term:
            break
        if term.casefold() not in [t.casefold() for t in terms]:
            terms.append(term)
    if not terms:
        print('No search terms entered; nothing searched.')
        return
    scan(folder.resolve(), terms)


if __name__ == '__main__':
    try:
        main()
    except (KeyboardInterrupt, EOFError):
        print('\nStopped.')
