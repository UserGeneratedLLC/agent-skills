# optimize-luau CLI Setup

Consult this before Phase 0. Handles temp directory hygiene and Luau CLI installation across OSes.

## Workspace hygiene

CLI tools generate residual files (`stats.json`, `profile.out`, `coverage.out`, `trace.json`). At the start of execution, create a temp working directory and use it for all tool output:

```
# Windows
$LUAU_TMP = New-Item -ItemType Directory -Path "$env:TEMP\luau-optimize-$(Get-Random)"
# Linux/Mac
LUAU_TMP=$(mktemp -d)
```

Route all output there: `--stats-file=$LUAU_TMP\stats.json`, write benchmark files to `$LUAU_TMP\bench.luau`, run profiling/coverage tools with `$LUAU_TMP` as working directory. The OS handles cleanup -- no manual deletion needed.

## CLI install

After creating the temp directory, check if `luau-compile` is on PATH. If missing, download the latest release from https://github.com/luau-lang/luau and extract to `$LUAU_TMP`, then prepend to PATH for the session:

```
# Windows (PowerShell)
if (-not (Get-Command luau-compile -ErrorAction SilentlyContinue)) {
    $tag = (Invoke-RestMethod "https://api.github.com/repos/luau-lang/luau/releases/latest").tag_name
    Invoke-WebRequest -Uri "https://github.com/luau-lang/luau/releases/download/$tag/luau-windows.zip" -OutFile "$LUAU_TMP\luau.zip"
    Expand-Archive -Path "$LUAU_TMP\luau.zip" -DestinationPath $LUAU_TMP
    $env:PATH = "$LUAU_TMP;$env:PATH"
}

# macOS (bash/zsh)
if ! command -v luau-compile &> /dev/null; then
    tag=$(curl -s "https://api.github.com/repos/luau-lang/luau/releases/latest" | grep -o '"tag_name": "[^"]*"' | cut -d'"' -f4)
    curl -L "https://github.com/luau-lang/luau/releases/download/$tag/luau-macos.zip" -o "$LUAU_TMP/luau.zip"
    unzip "$LUAU_TMP/luau.zip" -d "$LUAU_TMP" && chmod +x "$LUAU_TMP"/luau*
    export PATH="$LUAU_TMP:$PATH"
fi

# Linux (bash)
if ! command -v luau-compile &> /dev/null; then
    tag=$(curl -s "https://api.github.com/repos/luau-lang/luau/releases/latest" | grep -o '"tag_name": "[^"]*"' | cut -d'"' -f4)
    curl -L "https://github.com/luau-lang/luau/releases/download/$tag/luau-ubuntu.zip" -o "$LUAU_TMP/luau.zip"
    unzip "$LUAU_TMP/luau.zip" -d "$LUAU_TMP" && chmod +x "$LUAU_TMP"/luau*
    export PATH="$LUAU_TMP:$PATH"
fi
```
