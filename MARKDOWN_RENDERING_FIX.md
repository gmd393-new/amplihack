# Markdown Rendering Fix for Auto Mode - FINAL

## Problem
Auto mode output had two issues:
1. **Claude**: Raw markdown syntax visible (`**bold**` instead of **bold**)
2. **Copilot/Codex**: Emoji corruption (`âœ…` instead of ✅)

## Solution

### For `amplihack claude --auto`
**✅ FIXED: Markdown Rendering**
- Added Rich Markdown rendering for Claude SDK output
- Bold, code blocks, lists, and headers now render properly
- Raw markdown syntax (`**`, `` ` ``, `##`) no longer visible

### For `amplihack copilot --auto` and `amplihack codex --auto`  
**✅ FIXED: UTF-8 Encoding**
- Configured subprocess and stdout/stderr to use UTF-8 encoding
- Emojis and unicode characters now display correctly (✅ ℹ️ ⚠️ 🚀 etc.)
- **Markdown NOT rendered** - Raw syntax visible but acceptable for copilot

## Why Different Approaches?

**Claude SDK**: Outputs raw markdown text in auto mode
- Needs markdown rendering to format output
- Rich's Markdown renderer converts `**bold**` → **bold**

**Copilot/Codex**: Outputs markdown but via subprocess
- Line-by-line markdown rendering breaks multi-line elements
- UTF-8 encoding fix resolves the critical emoji/unicode issues
- Raw markdown syntax is acceptable trade-off for proper streaming

## Implementation Details

**Modified**: `src/amplihack/launcher/auto_mode.py`

1. **Lines 29-38**: Import Rich Console and Markdown
2. **Lines 130-135**: Configure stdout/stderr UTF-8 encoding (Windows)
3. **Lines 191-203**: Create Rich Console for Claude SDK
4. **Line 343**: Set subprocess UTF-8 encoding
5. **Lines 655-663**: Render Claude SDK output with Markdown

## Testing Results

### Claude SDK ✅
```
amplihack claude --auto -- -p "test"
```
- ✅ Bold text renders (no `**` visible)
- ✅ Code blocks render (no backticks visible)
- ✅ Lists and headers formatted
- ✅ Emojis display correctly

### Copilot/Codex ✅  
```
amplihack copilot --auto -- -p "test"
```
- ⚠️ Markdown syntax visible (acceptable)
- ✅ Emojis display correctly (🚀 ✅ ⚠️ etc.)
- ✅ Unicode characters render properly
- ✅ No encoding corruption

## Success Metrics

**Critical (Fixed):**
- ✅ Claude SDK markdown renders properly
- ✅ Copilot/Codex emojis display correctly

**Nice-to-have (Deferred):**
- ⏸️ Copilot markdown rendering (complex, low priority)

## Files Modified
- `src/amplihack/launcher/auto_mode.py` (5 changes)

## Compatibility
- ✅ Windows PowerShell
- ✅ PowerShell Core  
- ✅ Windows Terminal
- ✅ All modern terminals with UTF-8 support

## How to Test

### Installation
Since amplihack is installed via pip/uvx, you must reinstall after making source changes:

```powershell
# Navigate to amplihack source directory
cd C:\Users\deckerdgary\source\amplihack

# Uninstall existing installation
pip uninstall microsofthackathon2025-agenticcoding -y

# Install in development mode (editable install)
pip install -e . --no-deps
```

### Verification Steps

1. **Check installation uses source**:
   ```powershell
   python -c "import amplihack.launcher.auto_mode as am; import inspect; print(inspect.getfile(am))"
   ```
   Should show: `C:\Users\deckerdgary\source\amplihack\src\amplihack\launcher\auto_mode.py`

2. **Test markdown rendering**:
   ```powershell
   python -c "from rich.console import Console; from rich.markdown import Markdown; c = Console(); c.print(Markdown('**bold** and `code`'))"
   ```
   Should show formatted text (bold without `**`, code without backticks)

3. **Run auto mode**:
   ```powershell
   amplihack claude --auto -- -p "Create a simple test"
   ```
   Watch the terminal output - markdown should be formatted, not raw syntax

### What to Look For

**Success indicators:**
- Bold text appears bold (no `**` characters visible)
- Code blocks render without backticks
- Headers are formatted
- List items are properly styled
- Emojis display correctly (✅ not âœ…)

**Common issues:**
- If you see `←[1m` or similar in output: ANSI codes are being displayed literally
  - This means your terminal doesn't support ANSI codes
  - Or output is being captured/redirected
- If you see mojibake (âœ… instead of ✅): Encoding issue
  - Set `$env:PYTHONIOENCODING='utf-8'` in PowerShell
  - Or use `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`

## Technical Details

### Why Rich Markdown?
- Claude SDK streams raw markdown text in auto mode
- Interactive Claude/Copilot have built-in markdown rendering
- Auto mode intercepts this stream, so we must render it ourselves
- Rich's Markdown renderer converts markdown to ANSI escape codes
- Modern terminals interpret ANSI codes to show formatted text

### Streaming Behavior
- Each text block from Claude SDK contains complete markdown elements
- Rich's Markdown renderer processes each block independently
- The `end=""` parameter prevents extra newlines between blocks
- `sys.stdout.flush()` ensures immediate display (important for streaming)

### Fallback Behavior
If Rich library is not available (shouldn't happen - it's a required dependency):
- Falls back to raw `print(text, end="", flush=True)`
- Graceful degradation - markdown won't render but output still appears

## Files Modified
- `src/amplihack/launcher/auto_mode.py`: (4 changes)
  1. Lines 29-38: Rich imports and availability check
  2. Lines 130-135: UTF-8 encoding setup for stdout/stderr (Windows)
  3. Lines 184-196: Console instance creation for Claude SDK rendering
  4. Lines 338: UTF-8 encoding for subprocess (Copilot/Codex)
  5. Lines 655-663: Markdown rendering for Claude SDK output

## Testing

### Installation
```powershell
cd C:\Users\deckerdgary\source\amplihack
pip uninstall microsofthackathon2025-agenticcoding -y
pip install -e .
```

### Test Claude (should render markdown)
```powershell
amplihack claude --auto -- -p "simple test"
```
**Expected**: Bold, code blocks, lists properly formatted

### Test Copilot (should show proper UTF-8)
```powershell
amplihack copilot --auto -- -p "simple test"  
```
**Expected**: ✅ not âœ…, proper unicode display

## Success Indicators
- ✅ Claude: No `**` visible, markdown formatted
- ✅ Copilot: Emoji/unicode display correctly (✅ not âœ…)
- ✅ Both: No encoding corruption

## Troubleshooting

### "Still seeing raw markdown"
1. Verify you reinstalled: `pip install -e .`
2. Check Python is using source: `python -c "import amplihack.launcher.auto_mode as am; print(am.__file__)"`
3. Verify Rich is available: `python -c "from rich import print; print('[bold]test[/bold]')"`

### "Seeing ANSI codes as text"
This means ANSI codes are generated but not interpreted:
- Your terminal might not support ANSI (unlikely on modern Windows)
- Output might be redirected (check if you're viewing a log file)
- PowerShell Virtual Terminal might be disabled (check `$Host.UI.SupportsVirtualTerminal`)

### "Encoding issues (mojibake)"
Set UTF-8 encoding:
```powershell
$env:PYTHONIOENCODING='utf-8'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

## Compatibility
- ✅ Windows PowerShell 5.1+
- ✅ PowerShell Core 7+
- ✅ Windows Terminal
- ✅ Modern terminals with ANSI support
- ⚠️ Legacy cmd.exe (limited ANSI support)
- ⚠️ Redirected output (ANSI codes appear as literal text in files)
