# 08 - Performance Tooling Integration

Atelier integrates with low-level performance tools to validate Fidelity's compile-time guarantees. This document describes both Atelier's native integration and the general patterns for other editors.

## The Verification Challenge

Fidelity makes compile-time claims about cache behavior, memory layout, and concurrency correctness. These claims derive from:

- BAREWire's deterministic layouts
- Arena isolation guarantees
- `[<CacheLineAligned>]` attribute semantics

Verification confirms these claims against actual hardware behavior using tools like `perf`, VTune, and hardware performance counters.

## Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Tooling Integration Layers                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Presentation Layer (Editor-Specific)                               ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 ││
│  │  │  Atelier   │  │    nvim     │  │   VSCode    │  ...            ││
│  │  │  (Native)   │  │   (Lua)     │  │ (Extension) │                 ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘                 ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Protocol Layer (Editor-Agnostic)                                   ││
│  │  - LSP extensions (fidelity/*)                                      ││
│  │  - JSON verification results format                                 ││
│  │  - SARIF for CI/CD                                                  ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Analysis Layer                                                     ││
│  │  - fidelity-verify CLI                                              ││
│  │  - FSNAC LSP server                                                 ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Collection Layer (Platform-Specific)                               ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 ││
│  │  │ Linux/perf  │  │ Intel/VTune │  │ macOS/Instr │                 ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘                 ││
│  └─────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

## Atelier Native Integration

Atelier's multi-WebView architecture enables the richest integration:

### Verification Panel WebView

A dedicated WebView for verification results:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Atelier                                                    [─][□][×]  │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────┬────────────────────────────────────┐│
│  │  Editor                        │  Verification Panel                ││
│  │                                │                                    ││
│  │  1│ [<CacheLineAligned>]      │  Cache Analysis                    ││
│  │  2│ type WorkerState = {      │  ────────────────                  ││
│  │  3│   mutable counter: int64  │  [✓] Arena isolation confirmed     ││
│  │  4│   mutable flags: byte     │      0 HITM events                 ││
│  │  5│ }                         │                                    ││
│  │  6│                           │  [!] Line 12: False sharing        ││
│  │  7│ type SharedState = {      │      1,247 HITM events             ││
│  │  8│   mutable a: int64        │      Fields 'a' and 'b'            ││
│  │  9│   mutable b: int64        │                                    ││
│  │ 10│ }                         │  ────────────────                  ││
│  │ 11│                           │  Last run: 12:34:56                ││
│  │ 12│ let shared = ...⚠️        │  Profile: 2.3 MB, 1.2M samples    ││
│  │                                │                                    ││
│  └────────────────────────────────┴────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │  Terminal: perf c2c record complete (exit 0)                        ││
│  └─────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```fsharp
// F# Native backend
module VerificationEngine =
    /// Run perf collection
    let collect (binary: string) (workload: string option) : Async<ProfileData> =
        async {
            let args =
                match workload with
                | Some w -> $"c2c record -o perf.data -- {binary} {w}"
                | None -> $"c2c record -o perf.data -- {binary}"
            let! result = Process.run "perf" args
            return ProfileData.load "perf.data"
        }

    /// Analyze collected data
    let analyze (profile: ProfileData) (debug: DwarfInfo) : VerificationResult =
        let hitm = profile.GetHitmEvents()
        let mapped = hitm |> List.map (fun e -> debug.MapToSource e.Address)
        // Correlate with compile-time predictions
        { ... }

// WebView receives results via BAREWire IPC
module VerificationPanel =
    let onResults (results: VerificationResult) =
        // Update SolidJS signals
        setDiagnostics results.Diagnostics
        setSummary results.Summary
```

### Inline Diagnostics

Verification results appear as inline diagnostics in the editor:

```fsharp
// Partas.Solid component for inline diagnostic
let InlineDiagnostic (props: {| diagnostic: Diagnostic |}) =
    let severity = props.diagnostic.Severity
    let icon =
        match severity with
        | Info -> "✓"
        | Warning -> "⚠️"
        | Error -> "❌"

    span [
        class' $"inline-diagnostic {severity.ToString().ToLower()}"
        title props.diagnostic.Message
    ] [
        text icon
    ]
```

### Cache Line Visualization

For complex layouts, visualize cache line boundaries:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Cache Line Map: WorkerState[]                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Cache Line 0 (0x1000-0x103F)                                          │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │ WorkerState[0]                                           │          │
│  │ counter: 8B │ flags: 1B │ padding: 55B                   │          │
│  └──────────────────────────────────────────────────────────┘          │
│  Access: Thread 0 only ✓                                               │
│                                                                         │
│  Cache Line 1 (0x1040-0x107F)                                          │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │ WorkerState[1]                                           │          │
│  │ counter: 8B │ flags: 1B │ padding: 55B                   │          │
│  └──────────────────────────────────────────────────────────┘          │
│  Access: Thread 1 only ✓                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## nvim Integration

nvim integration uses the LSP protocol and Lua plugins:

### LSP Extension Support

FSNAC exposes custom LSP methods. nvim clients can call these:

```lua
-- nvim/lua/fidelity/verification.lua

local M = {}

-- Request verification run
function M.run_verification()
  local client = vim.lsp.get_active_clients({ name = "fsnac" })[1]
  if client then
    client.request("fidelity/runVerification", {
      binary = vim.fn.expand("%:p:r"),  -- Binary path from source file
    }, function(err, result)
      if err then
        vim.notify("Verification failed: " .. err.message, vim.log.levels.ERROR)
      else
        M.display_results(result)
      end
    end)
  end
end

-- Display results as diagnostics
function M.display_results(results)
  local diagnostics = {}
  for _, diag in ipairs(results.diagnostics) do
    table.insert(diagnostics, {
      lnum = diag.location.line - 1,
      col = diag.location.column - 1,
      message = diag.message,
      severity = M.map_severity(diag.severity),
      source = "fidelity-verify",
    })
  end
  vim.diagnostic.set(M.namespace, 0, diagnostics)
end

function M.map_severity(sev)
  if sev == "error" then return vim.diagnostic.severity.ERROR
  elseif sev == "warning" then return vim.diagnostic.severity.WARN
  else return vim.diagnostic.severity.INFO end
end

M.namespace = vim.api.nvim_create_namespace("fidelity")

return M
```

### Keybindings

```lua
-- nvim/lua/fidelity/init.lua
vim.keymap.set("n", "<leader>fv", require("fidelity.verification").run_verification,
  { desc = "Run Fidelity verification" })
vim.keymap.set("n", "<leader>fc", require("fidelity.verification").clear,
  { desc = "Clear verification diagnostics" })
```

### Terminal Fallback

For users without LSP support or preferring terminal workflow:

```lua
-- Direct terminal integration
function M.run_verification_terminal()
  local binary = vim.fn.expand("%:p:r")
  vim.cmd("terminal perf c2c record -o perf.data -- " .. binary)
  vim.cmd("terminal fidelity-verify perf.data --format=pretty")
end
```

## VSCode Integration

VSCode uses its extension API:

### Extension Structure

```
fidelity-vscode/
├── src/
│   ├── extension.ts
│   ├── verification.ts
│   ├── diagnostics.ts
│   └── views/
│       └── cacheLineView.ts
├── package.json
└── tsconfig.json
```

### LSP Client Extension

```typescript
// src/verification.ts
import * as vscode from 'vscode';
import { LanguageClient } from 'vscode-languageclient/node';

export async function runVerification(client: LanguageClient) {
  const editor = vscode.window.activeTextEditor;
  if (!editor) return;

  const result = await client.sendRequest('fidelity/runVerification', {
    binary: editor.document.uri.fsPath.replace(/\.[^.]+$/, ''),
  });

  displayDiagnostics(result.diagnostics);
}

function displayDiagnostics(diagnostics: FidelityDiagnostic[]) {
  const collection = vscode.languages.createDiagnosticCollection('fidelity');

  const byFile = new Map<string, vscode.Diagnostic[]>();
  for (const diag of diagnostics) {
    const uri = vscode.Uri.file(diag.location.file);
    const range = new vscode.Range(
      diag.location.line - 1, diag.location.column - 1,
      diag.location.line - 1, 100
    );
    const vsdiag = new vscode.Diagnostic(range, diag.message, mapSeverity(diag.severity));
    vsdiag.source = 'fidelity-verify';

    const existing = byFile.get(uri.toString()) || [];
    existing.push(vsdiag);
    byFile.set(uri.toString(), existing);
  }

  for (const [uri, diags] of byFile) {
    collection.set(vscode.Uri.parse(uri), diags);
  }
}
```

### Custom WebView Panel

For cache line visualization:

```typescript
// src/views/cacheLineView.ts
export class CacheLineViewProvider implements vscode.WebviewViewProvider {
  resolveWebviewView(webviewView: vscode.WebviewView) {
    webviewView.webview.options = { enableScripts: true };
    webviewView.webview.html = this.getHtml();

    // Receive verification results
    this.onResultsReceived = (results) => {
      webviewView.webview.postMessage({
        type: 'updateResults',
        results,
      });
    };
  }

  private getHtml(): string {
    return `
      <!DOCTYPE html>
      <html>
        <head>
          <style>
            .cache-line { border: 1px solid #ccc; margin: 4px; padding: 8px; }
            .contention { border-color: #f00; background: #fee; }
            .isolated { border-color: #0f0; background: #efe; }
          </style>
        </head>
        <body>
          <div id="cache-map"></div>
          <script>
            window.addEventListener('message', event => {
              if (event.data.type === 'updateResults') {
                renderCacheMap(event.data.results);
              }
            });
          </script>
        </body>
      </html>
    `;
  }
}
```

## Common Protocol

All editors consume the same verification result format:

```typescript
interface VerificationResult {
  version: "1.0";
  session: {
    binary: string;
    profile_data: string;
    timestamp: string;
  };
  diagnostics: Diagnostic[];
  summary: Summary;
}

interface Diagnostic {
  kind: "false_sharing_detected" | "false_sharing_confirmed" |
        "cache_line_crossing" | "arena_isolation_confirmed";
  severity: "info" | "warning" | "error";
  location: {
    file: string;
    line: number;
    column: number;
  };
  message: string;
  details: Record<string, unknown>;
}

interface Summary {
  total_hitm_events: number;
  confirmed_isolations: number;
  detected_issues: number;
}
```

## Workflow Patterns

### Pattern 1: Development Cycle Validation

During development, run verification after significant changes:

```
1. Write code with [<CacheLineAligned>] or arena isolation
2. Build with debug info: firefly compile --debug
3. Run with profiling: perf c2c record -o perf.data ./program
4. Analyze: fidelity-verify perf.data
5. Review results in editor
6. Iterate if issues found
```

### Pattern 2: Pre-Commit Hook

Automated verification before commits:

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Build and profile
firefly compile --debug
perf c2c record -o perf.data -- ./tests/benchmark

# Check for cache issues
result=$(fidelity-verify perf.data --format=json)
issues=$(echo "$result" | jq '.summary.detected_issues')

if [ "$issues" -gt 0 ]; then
  echo "Cache verification failed: $issues issues detected"
  fidelity-verify perf.data --format=pretty
  exit 1
fi
```

### Pattern 3: CI/CD Integration

```yaml
# .github/workflows/verify.yml
- name: Cache verification
  run: |
    firefly compile --debug
    perf c2c record -o perf.data -- ./benchmarks
    fidelity-verify perf.data --format=sarif > verification.sarif

- name: Upload results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: verification.sarif
```

## Platform Support

| Platform | Collection Tool | Integration Level |
|----------|----------------|-------------------|
| Linux | `perf c2c` | Full |
| Intel (any OS) | VTune | Full |
| macOS | Instruments | Partial (no HITM) |
| Windows | VTune, WPA | Partial |

## References

- Firefly Verification Workflow: `~/repos/Firefly/docs/Verification_Workflow_Architecture.md`
- BAREWire Cache-Aware Layouts: `~/repos/BAREWire/docs/09 Cache-Aware Layouts.md`
- fsnative Atomic Operations: `~/repos/fsnative-spec/spec/atomic-operations.md`
- perf c2c documentation: https://man7.org/linux/man-pages/man1/perf-c2c.1.html
