# Fixes applied to verilog-vcd-parser

## VCDScanner.l

### 1. `strcmp` returns 0 on match but was used as boolean truth (timescale unit)

Lines 165-175: The `strcmp` calls were not negated, so the first branch
(`"s"`) always matched regardless of actual input, making the timescale
unit always `TIME_S`.

**Fix:** Negate each `strcmp` call: `if(!std::strcmp(yytext, "s"))` etc.

### 2. Signal reference names containing dots

The `IN_VAR_PID` scanner state used `SCOPE_IDENTIFIER` which does not
include dots or brackets. Signal names like `cos_out.d` or
`atan_table[0]` would be split into multiple tokens, causing parse
errors.

**Fix:** Added a `SIGNAL_REFERENCE` pattern that includes dots and
square brackets: `[a-zA-Z_][a-zA-Z_0-9\.\(\)\[\]]*`, used in the
`IN_VAR_PID` rule instead of `SCOPE_IDENTIFIER`.

### 3. Parsing multiple files segfaults (non-reentrant scanner state)

`scan_begin()` only set `yyin` without resetting the flex scanner's
internal buffer state. After parsing a first file and calling
`scan_end()` (which pops the buffer), the second `scan_begin()` left the
scanner with a stale/freed buffer, causing a segfault.

**Fix:** Added `yyrestart(yyin)` and `BEGIN(INITIAL)` at the end of
`scan_begin()` to properly reinitialize the scanner for each new file.

## VCDParser.ypp

### 4. Assertion crash on multi-bit signals without explicit bit ranges

Lines 152-156: An `assert` required multi-bit signals to have explicit
`[lindex:rindex]` declarations. Signals declared as e.g.
`$var wire 16 ! angle_in $end` (no bit range) would crash.

**Fix:** Replaced the assertion with inference logic: if `lindex == -1`
and `size > 1`, set `lindex = size - 1` and `rindex = 0`.
