# M-DX1.11: Enhanced Builtin Metadata

**Status**: Planned
**Prerequisites**: M-DX1.5 complete (v0.3.10) ✅
**Estimated Total Effort**: ~8-10 hours
**Target Release**: v0.3.15
**Priority**: P1 (High developer value, enables better tooling)

## Context

M-DX1.5 (v0.3.10) completed the core builtin migration with a minimal `BuiltinSpec`:
```go
type BuiltinSpec struct {
    Module  string            // Module path (e.g., "std/net")
    Name    string            // Builtin name (e.g., "_net_httpRequest")
    NumArgs int               // Number of arguments
    IsPure  bool              // true = no side effects
    Effect  string            // "" for pure, "Net"/"IO"/"FS" for effects
    Type    func() types.Type // Type signature constructor
    Impl    EffectImpl        // Implementation function
}
```

**What's missing**: Rich metadata for documentation, tooling, and developer experience.

## Problem Statement

### Current Limitations

1. **No inline documentation**
   - Developers must read implementation code to understand what a builtin does
   - `ailang builtins list` shows only names and purity, not descriptions
   - REPL `:type` command (M-DX1.6) won't show usage examples

2. **No versioning information**
   - Can't tell when a builtin was added
   - No deprecation tracking or migration hints
   - Breaking changes are invisible

3. **Poor discoverability**
   - No "see also" links to related builtins
   - No examples in the registry
   - Hard to find the right builtin for a task

4. **Limited tooling support**
   - Can't generate API documentation from specs
   - LSP (future) would have no hover documentation
   - AI code generation has no semantic context

### Real-World Impact

**Example: Developer wants to make an HTTP request**
```bash
$ ailang builtins list --by-module
# std/net (1)
  _net_httpRequest               [net]
```

**Questions the developer has:**
- What does `_net_httpRequest` do? (no description)
- What arguments does it take and what do they mean? (no parameter docs)
- What does it return? (no return description)
- How do I use it? (no examples)
- Is there a simpler version? (no "see also")
- Is it stable or experimental? (no stability info)

**Current solution:** Search through code, examples, or ask AI. **Time cost: 5-10 minutes per builtin**

## Proposed Solution

### Enhanced BuiltinSpec

```go
type BuiltinSpec struct {
    // === Core Metadata (v0.3.10 - existing) ===
    Module  string
    Name    string
    NumArgs int
    IsPure  bool
    Effect  string
    Type    func() types.Type
    Impl    EffectImpl

    // === Documentation Metadata (v0.3.15 - NEW) ===
    Description string   // One-line summary (required)
    LongDesc    string   // Multi-line detailed description (optional)
    Params      []ParamDoc  // Parameter documentation
    Returns     string   // Return value description
    Examples    []Example   // Usage examples
    SeeAlso     []string // Related builtins

    // === Versioning Metadata (v0.3.15 - NEW) ===
    Since       string   // Version when added (e.g., "v0.2.0")
    Deprecated  string   // If non-empty, deprecation message
    Stability   Stability // Experimental, Stable, Deprecated

    // === Semantic Metadata (v0.3.15 - NEW) ===
    Tags        []string // Searchable tags (e.g., "http", "network", "io")
    Category    string   // High-level category (e.g., "network", "string", "math")
}

type ParamDoc struct {
    Name        string // Parameter name
    Description string // What it does
}

type Example struct {
    Code        string // AILANG code example
    Description string // What the example demonstrates
}

type Stability int

const (
    StabilityExperimental Stability = iota // May change in any release
    StabilityStable                        // Backwards compatible
    StabilityDeprecated                    // Use alternative
)
```

### Example Registration (Before vs After)

**Before (v0.3.10):**
```go
RegisterEffectBuiltin(BuiltinSpec{
    Module:  "std/net",
    Name:    "_net_httpRequest",
    NumArgs: 4,
    IsPure:  false,
    Effect:  "Net",
    Type:    makeHTTPRequestType,
    Impl:    effects.NetHTTPRequest,
})
```

**After (v0.3.15):**
```go
RegisterEffectBuiltin(BuiltinSpec{
    // Core metadata (unchanged)
    Module:  "std/net",
    Name:    "_net_httpRequest",
    NumArgs: 4,
    IsPure:  false,
    Effect:  "Net",
    Type:    makeHTTPRequestType,
    Impl:    effects.NetHTTPRequest,

    // Documentation metadata (NEW)
    Description: "Make an HTTP request with custom headers and body",
    LongDesc: `Performs an HTTP request to the specified URL with the given method,
headers, and request body. Returns a Result type containing either the HTTP
response (status, headers, body) or a NetError if the request fails.

The Net capability must be granted to use this function.`,

    Params: []ParamDoc{
        {Name: "url", Description: "Target URL (must start with http:// or https://)"},
        {Name: "method", Description: "HTTP method (GET, POST, PUT, DELETE, etc.)"},
        {Name: "headers", Description: "List of {name, value} header records"},
        {Name: "body", Description: "Request body as a string"},
    },

    Returns: "Result[HttpResponse, NetError] where HttpResponse contains status, headers, and body",

    Examples: []Example{
        {
            Code: `let response = _net_httpRequest(
  "https://api.example.com/users",
  "GET",
  [{name: "Accept", value: "application/json"}],
  ""
) in
match response with
| Ok(r) -> _io_println(r.body)
| Err(e) -> _io_println("Request failed")
end`,
            Description: "Simple GET request with error handling",
        },
    },

    SeeAlso: []string{"_net_httpGet", "_net_httpPost"},

    // Versioning metadata (NEW)
    Since:     "v0.2.0",
    Stability: StabilityStable,

    // Semantic metadata (NEW)
    Tags:     []string{"http", "network", "request", "api"},
    Category: "network",
})
```

## Implementation Plan

### Phase 1: Core Metadata Types (~2 hours)

**Task 1.1: Define new metadata types** (~1h)
- Create `ParamDoc`, `Example`, `Stability` types
- Add fields to `BuiltinSpec` (backward compatible - all optional except Description)
- Update validation to require `Description` for new builtins

**Files:**
- `internal/builtins/spec.go` (~100 LOC additions)
- `internal/builtins/metadata.go` (~150 LOC new file for types)

**Task 1.2: Update validation** (~1h)
- Check that `Description` is non-empty for new registrations
- Validate `Since` version format (if provided)
- Validate `SeeAlso` references exist (cross-reference check)
- Warn if `Stability` is Experimental or Deprecated

**Files:**
- `internal/builtins/validator.go` (~100 LOC additions)
- `internal/builtins/validator_test.go` (~100 LOC tests)

### Phase 2: Migration Tool (~2 hours)

**Task 2.1: Generate metadata from code** (~2h)
- Parse implementation function comments (if present)
- Extract parameter names from type signature
- Auto-generate basic `ParamDoc` entries
- Detect `Since` from git history (first commit that added the builtin)
- Default `Stability` to Stable for existing builtins

**Files:**
- `tools/migrate_builtin_metadata.go` (~300 LOC new)
- Can be run manually: `go run tools/migrate_builtin_metadata.go`

**Output:** Suggested metadata for each builtin (human reviews and edits)

### Phase 3: Update Existing Builtins (~2 hours)

**Task 3.1: Add metadata to all 51 builtins** (~2h)
- Use migration tool output as starting point
- Human review and enhance each description
- Add examples for commonly used builtins (top 20)
- Mark experimental builtins (if any)

**Files:**
- `internal/builtins/register.go` (~500 LOC additions - metadata only)

**Strategy:**
- Batch 1: String operations (9 builtins) - ~30 min
- Batch 2: Math operations (12 builtins) - ~30 min
- Batch 3: Comparisons (25 builtins) - ~30 min
- Batch 4: Effects (IO, Net, JSON) (5 builtins) - ~30 min

### Phase 4: Enhanced CLI Tools (~2 hours)

**Task 4.1: Update `ailang builtins list`** (~1h)
- Add `--verbose` flag to show descriptions
- Show parameter names and return type
- Display stability badges
- Format examples nicely

**Example output:**
```bash
$ ailang builtins list --verbose std/net

# std/net (1)

_net_httpRequest [net] [stable since v0.2.0]
  Description: Make an HTTP request with custom headers and body

  Parameters:
    url: string         - Target URL (must start with http:// or https://)
    method: string      - HTTP method (GET, POST, PUT, DELETE, etc.)
    headers: [{...}]    - List of {name, value} header records
    body: string        - Request body as a string

  Returns: Result[HttpResponse, NetError]

  Tags: http, network, request, api
  See Also: _net_httpGet, _net_httpPost

  Example:
    let response = _net_httpRequest(
      "https://api.example.com/users",
      "GET",
      [{name: "Accept", value: "application/json"}],
      ""
    )
```

**Files:**
- `cmd/ailang/main.go` (~150 LOC modifications to `listBuiltinsByModule`)

**Task 4.2: Update `ailang doctor builtins`** (~30 min)
- Check for missing descriptions
- Check for missing examples on commonly used builtins
- Warn about deprecated builtins
- Suggest adding metadata

**Files:**
- `internal/builtins/validator.go` (~50 LOC additions)

**Task 4.3: Add `ailang builtins search`** (~30 min)
- Search by tags, category, or description keywords
- Fuzzy matching for discoverability

**Example:**
```bash
$ ailang builtins search "http"
Found 3 builtins:

_net_httpRequest [net] - Make an HTTP request with custom headers and body
_net_httpGet [net] - Simple HTTP GET request
_net_httpPost [net] - Simple HTTP POST request
```

**Files:**
- `cmd/ailang/main.go` (~100 LOC new subcommand)

### Phase 5: Documentation Generation (~2 hours)

**Task 5.1: Generate markdown API reference** (~1h)
- Create `ailang builtins docs` command
- Output markdown file with all builtins
- Group by module, show full metadata
- Include examples and cross-references

**Example:**
```bash
$ ailang builtins docs > docs/API_REFERENCE.md
Generated API reference for 51 builtins
```

**Output format:**
```markdown
# AILANG Builtin Function Reference

## std/net

### _net_httpRequest

**Signature:** `(url: string, method: string, headers: [{name: string, value: string}], body: string) -> <Net> Result[HttpResponse, NetError]`

**Description:** Make an HTTP request with custom headers and body

**Since:** v0.2.0
**Stability:** Stable
**Effect:** Net

**Parameters:**
- `url` (string): Target URL (must start with http:// or https://)
- `method` (string): HTTP method (GET, POST, PUT, DELETE, etc.)
- `headers` ([{name: string, value: string}]): List of header records
- `body` (string): Request body as a string

**Returns:** Result[HttpResponse, NetError] where HttpResponse contains status, headers, and body

**Example:**
\`\`\`ailang
let response = _net_httpRequest(
  "https://api.example.com/users",
  "GET",
  [{name: "Accept", value: "application/json"}],
  ""
) in
match response with
| Ok(r) -> _io_println(r.body)
| Err(e) -> _io_println("Request failed")
end
\`\`\`

**See Also:** _net_httpGet, _net_httpPost

**Tags:** http, network, request, api
```

**Files:**
- `cmd/ailang/builtin_docs.go` (~200 LOC new)
- `docs/API_REFERENCE.md` (~auto-generated, ~2000 LOC)

**Task 5.2: Integrate with website** (~1h)
- Add to Docusaurus docs
- Create searchable builtin reference page
- Link from main docs

**Files:**
- `docs/docs/reference/builtins.md` (generated)
- `docs/docusaurus.config.js` (add to nav)

## Success Criteria

**Developer Experience:**
- ✅ Every builtin has a description
- ✅ Top 20 builtins have usage examples
- ✅ `ailang builtins list --verbose` shows full metadata
- ✅ `ailang builtins search` enables discovery
- ✅ Auto-generated API reference documentation

**Tooling:**
- ✅ `ailang doctor builtins` checks metadata completeness
- ✅ Validation requires Description for new builtins
- ✅ Cross-reference validation (SeeAlso links exist)

**Documentation:**
- ✅ docs/API_REFERENCE.md auto-generated
- ✅ Website builtin reference page
- ✅ All metadata fields documented in ADDING_BUILTINS.md

**Metrics:**
- Time to understand a builtin: 5-10 min → <1 min (-80%)
- Time to find the right builtin: 10 min → <2 min (-80%)
- 51/51 builtins have descriptions (100%)
- 20/51 builtins have examples (39% coverage on high-value items)

## Risks & Mitigations

**Risk 1: Metadata becomes stale**
- **Mitigation:** Validation in CI checks for missing descriptions
- **Mitigation:** `ailang doctor builtins` warns about incomplete metadata
- **Mitigation:** Automated tests verify examples still compile

**Risk 2: Too much boilerplate for new builtins**
- **Mitigation:** Only `Description` is required
- **Mitigation:** Other fields are optional (progressive enhancement)
- **Mitigation:** Migration tool generates initial metadata

**Risk 3: Examples become outdated**
- **Mitigation:** Extract examples to separate `.ail` files
- **Mitigation:** `make verify-examples` tests example code
- **Mitigation:** CI fails if example doesn't compile

## Future Work (v0.4.0+)

**Not in this milestone:**
- LSP hover documentation (requires LSP server)
- Interactive example runner in REPL
- AI-powered builtin suggestions
- Automated example generation
- Performance benchmarks in metadata
- Security/safety annotations

These are valuable but not critical for the core DX improvement goal.

## Timeline

**Week 1** (4 hours):
- Phase 1: Core metadata types (2h)
- Phase 2: Migration tool (2h)

**Week 2** (4 hours):
- Phase 3: Update existing builtins (2h)
- Phase 4: Enhanced CLI tools (2h)

**Week 3** (2 hours):
- Phase 5: Documentation generation (2h)

**Total: ~10 hours across 3 weeks**

## Example: Full Lifecycle

**1. Developer adds new builtin:**
```go
RegisterEffectBuiltin(BuiltinSpec{
    Module:      "std/string",
    Name:        "_str_reverse",
    NumArgs:     1,
    IsPure:      true,
    Type:        makeStrReverseType,
    Impl:        strReverseImpl,
    Description: "Reverse a string (UTF-8 aware)",  // ← REQUIRED
    Since:       "v0.3.15",
    Stability:   StabilityStable,
})
```

**2. Validation catches missing example (if high-priority):**
```bash
$ make test
Warning: Builtin '_str_reverse' is commonly used but has no examples
Consider adding an Example to BuiltinSpec
```

**3. Developer adds example:**
```go
Examples: []Example{
    {
        Code: `_str_reverse("hello")  # => "olleh"`,
        Description: "Simple string reversal",
    },
}
```

**4. CI auto-generates docs:**
```bash
$ make docs
Generating API reference...
✅ docs/API_REFERENCE.md updated
✅ 51/51 builtins documented
```

**5. User discovers the builtin:**
```bash
$ ailang builtins search "reverse"
Found 1 builtin:

_str_reverse [pure] - Reverse a string (UTF-8 aware)
```

**6. User checks details:**
```bash
$ ailang builtins list --verbose std/string

_str_reverse [pure] [stable since v0.3.15]
  Description: Reverse a string (UTF-8 aware)

  Parameters:
    s: string - Input string to reverse

  Returns: string - Reversed string

  Example:
    _str_reverse("hello")  # => "olleh"
```

## Dependencies

**Requires:**
- ✅ M-DX1.5 complete (v0.3.10) - All builtins migrated to registry
- ✅ Registry infrastructure working
- ✅ `ailang builtins list` command exists

**Enables:**
- M-DX1.6: REPL `:type` can show descriptions
- M-DX1.8: docs/ADDING_BUILTINS.md can reference metadata fields
- Future LSP server can use metadata for hover/completion
- Future AI code generation can use semantic tags

## References

- **Original design:** `design_docs/planned/m-dx1-future-polish.md`
- **Current implementation:** `internal/builtins/spec.go` (v0.3.10)
- **Related work:** M-DX1.1-1.5 (v0.3.9-v0.3.10)
- **Motivation:** Developer feedback on builtin discoverability

## Changelog Entry (Draft)

```markdown
## [v0.3.15] - TBD

### Added
- Enhanced builtin metadata system with documentation, versioning, and examples
- `Description`, `LongDesc`, `Params`, `Returns` fields in BuiltinSpec
- `Since`, `Deprecated`, `Stability` versioning metadata
- `Tags`, `Category` for semantic search
- `ailang builtins list --verbose` shows full metadata
- `ailang builtins search <keyword>` for discovery
- `ailang builtins docs` generates API reference markdown
- Auto-generated `docs/API_REFERENCE.md` with all builtins
- Metadata validation in `ailang doctor builtins`
- Migration tool to generate metadata from existing code

### Changed
- `BuiltinSpec.Description` is now required for new builtins
- `ailang builtins list` output includes stability badges
- `ailang doctor builtins` checks metadata completeness

### Documentation
- All 51 builtins now have descriptions
- 20 most-used builtins have usage examples
- Complete API reference documentation auto-generated
```
