# Search

Searching the project in three ways: by file names, by file content with a regular expression, and by meaning. All modes are read-only.

## Modes

The mode is selected by the `mode` field with the values `files`, `content`, or `smart`. A missing or unknown value returns an error indicating the allowed values.

### files

Lists files by name pattern.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| pattern | string | yes | — | name pattern, for example `**/*.rs` or `*.{ts,tsx}` |
| path | string | no | session working directory | directory to search; must exist and be a directory |

The search runs recursively, respecting hidden files and ignored directories, sorted by modification time. It returns a list of paths relative to the session working directory (paths outside it are returned as absolute), with the number of files and a truncation flag. The result is limited to 100 files; truncation is marked in the result. The environment variables `KOT_GLOB_NO_IGNORE` and `KOT_GLOB_HIDDEN` (both on by default) control the treatment of ignored and hidden files, and `KOT_GLOB_MAX_RESULTS` sets the limit instead of 100.

### content

Searches file content with a regular expression.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| pattern | string | yes | — | regular expression; a leading `-` is wrapped in `-e` |
| path | string | no | session working directory | file or directory to search |
| glob | string | no | — | name filter; split by spaces and commas, tokens with `{` and `}` kept whole |
| output_mode | string | no | files_with_matches | `files_with_matches`, `content`, or `count` |
| -B | integer | no | — | lines before the match |
| -A | integer | no | — | lines after the match |
| -C | integer | no | — | lines before and after |
| context | integer | no | — | lines before and after; priority context > -C > -B/-A |
| -n | boolean | no | true | show line numbers |
| -i | boolean | no | false | case-insensitive search |
| type | string | no | — | file type filter, for example `rust`, `js`, `py` |
| head_limit | integer | no | 250 | maximum results; 0 means no limit |
| offset | integer | no | 0 | skip the first N results before head_limit |
| multiline | boolean | no | false | the dot matches newlines, patterns can cross lines |

Version-control directories (`.git`, `.svn`, `.hg`, `.bzr`, `.jj`, `.sl`) are excluded from the search, hidden files are included, and long lines are truncated at 500 columns. The path accepts both a directory and a single file.

The `files_with_matches` mode returns only the list of files, sorted from newest to oldest. The `content` mode returns the matched lines with line numbers and paths relative to the working directory. The `count` mode returns the number of matches per file and the total across all files. The `head_limit` limit is applied after the search; on actual truncation the flag is set and the text adds an instruction to continue with a new `offset`.

### smart

Searches by meaning in natural language.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| query | string | yes | — | search intent, for example "where JWT is validated" |
| max_results | integer | no | — | soft limit on the number of results |

The query is executed by a limited sub-agent that works read-only and has access only to the Files (read) and Search tools. The sub-agent's intermediate calls are hidden. A strict list of `hits` is returned, where each element contains the path, an optional line number, and a mandatory explanation of why the element is relevant. The sub-agent is limited to 8 turns and must return the answer in the given structure; an invalid answer is returned as an error. If launching a sub-agent is unavailable in the current configuration, the mode returns an error.

## Behavior and limitations

All modes are read-only, do not destroy state, and are safe under concurrent execution. A relative `path` resolves against the session working directory, not against the process working directory. An absolute path is used as-is. For the `files` mode, a path with `~` is expanded to the home directory; in the `content` mode the path is returned as-is.

The `files` and `content` modes are executed by the ripgrep command, invoked by name from PATH. A missing ripgrep returns an installation error. The default timeout is 20 seconds (the `KOT_GLOB_TIMEOUT_SECONDS` variable). A timeout with no results returns a timeout error; a timeout with partial results returns them as a normal result: in the `files` mode they are marked with a truncation flag, while in the `content` mode there is no separate flag for any `output_mode` value (`files_with_matches`, `content`, `count`). Exit code 1 means no matches (not an error); code 2 means a regular-expression error or invalid arguments; other codes mean a failure. An EAGAIN error is retried once with one thread. Output is limited to 20 MB (stderr — 1 MB); on excess the trailing incomplete line is dropped.

A UNC path in the `content` mode is rejected immediately; in the `files` mode the check is deferred without filesystem access. A nonexistent path returns an error; in the `files` mode, a path that is not a directory also returns an error.

## Examples

Listing all Rust sources:

```json
{"mode": "files", "pattern": "**/*.rs"}
```

List of files containing a function call:

```json
{"mode": "content", "pattern": "fn validate_input", "glob": "*.rs", "output_mode": "files_with_matches"}
```

Matched lines with two lines of context before and after:

```json
{"mode": "content", "pattern": "MAX_READ_SIZE", "output_mode": "content", "context": 2, "-n": true}
```

Continuation of the output from the next page:

```json
{"mode": "content", "pattern": "TODO", "output_mode": "content", "head_limit": 250, "offset": 250}
```

Semantic search for an implementation site:

```json
{"mode": "smart", "query": "where JWT token validity is checked", "max_results": 5}
```
