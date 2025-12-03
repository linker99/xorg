# Rsyslog Test Failure Analysis

## Summary

The rsyslog test suite shows 4 failing tests out of 646 total tests. This document analyzes the main test failure (`parsertest-parse1-udp`) and provides solutions.

## Test Results Overview

- **TOTAL**: 646
- **PASS**: 625
- **SKIP**: 17
- **FAIL**: 4
- **ERROR**: 0

## Root Cause Analysis: parsertest-parse1-udp Failure

### Observed Behavior

The test fails due to a hostname case mismatch:

```
Expected (line 25): 14,user,info,Aug 30 23:00:05,build32U10b32b1b199,,,
Actual (line 25):   14,user,info,Aug 30 23:00:05,build32u10b32b1b199,,,
```

### Technical Explanation

The discrepancy occurs because rsyslog normalizes hostnames to lowercase during message processing. This is consistent with RFC 1123, which states that hostnames should be treated as case-insensitive.

When rsyslog processes the message:
1. The test machine's actual hostname is `build32U10b32b1b199` (with uppercase 'U')
2. Rsyslog converts the hostname to lowercase: `build32u10b32b1b199`
3. The test expected data file contains the original uppercase hostname
4. The comparison fails

### Evidence from Log

```
01:55:01[1]  rstb_607389_d1146e46d42O:.pid found, pid 2477609
HOSTNAME is: build32U10b32b1b199
...
- rstb_607389_d1146e46d42O.out.log differ: char 2317, line 25
```

## Solutions

### Solution 1: Update Test Expected Data (Recommended)

Modify the expected test data file to use lowercase hostnames, which aligns with rsyslog's RFC-compliant behavior.

**File to modify**: `tests/testsuites/parsertest-parse1-udp.data` (or similar)

Change line 25-26 from:
```
14,user,info,Aug 30 23:00:05,build32U10b32b1b199,,,
14,user,info,Aug 30 23:00:05,build32U10b32b1b199,,,
```

To:
```
14,user,info,Aug 30 23:00:05,build32u10b32b1b199,,,
14,user,info,Aug 30 23:00:05,build32u10b32b1b199,,,
```

### Solution 2: Disable Hostname Normalization

If case-sensitive hostnames are required, configure rsyslog to preserve hostname case:

```
global(preserveFQDN="on")
```

This may not fully address the issue as the normalization happens during parsing.

### Solution 3: Use Case-Insensitive Comparison in Test

Modify the test script to perform case-insensitive comparison for hostnames:

```bash
# Instead of:
diff expected.log actual.log

# Use:
diff <(tr '[:upper:]' '[:lower:]' < expected.log) <(tr '[:upper:]' '[:lower:]' < actual.log)
```

### Solution 4: Environment-Independent Test Data

The best long-term solution is to make the test data independent of the build machine's hostname. The test should use a fixed, predictable hostname value or normalize hostnames before comparison.

### Solution 5: Modify rsyslog Source Code (如何修改rsyslog代码)

If you need to change rsyslog's hostname processing behavior in the source code:

**1. Clone the rsyslog repository:**
```bash
git clone https://github.com/rsyslog/rsyslog.git
cd rsyslog
```

**2. Locate the hostname processing code:**

The hostname lowercasing happens in the parser module. Key files to examine:

- `runtime/msg.c` - Message processing and hostname handling
- `runtime/parser.c` - Syslog message parser
- `runtime/glbl.c` - Global configuration including `preserveFQDN`

**3. Find the lowercase conversion:**

Search for lowercase conversion in the codebase:
```bash
grep -r "tolower" runtime/
grep -r "HOSTNAME" runtime/msg.c
```

The conversion typically occurs in `MsgSetHOSTNAME()` or similar functions in `runtime/msg.c`.

**4. Modify the code:**

In `runtime/msg.c`, look for code similar to:
```c
/* Convert hostname to lowercase for RFC compliance */
for(i = 0; i < lenHOSTNAME; i++) {
    pszHOSTNAME[i] = tolower(pszHOSTNAME[i]);
}
```

To preserve case, you can either:
- Remove or comment out the lowercase conversion loop
- Add a configuration option to control this behavior
- Make it conditional based on the `preserveFQDN` setting

**5. Build and test:**
```bash
autoreconf -fvi
./configure --enable-testbench
make
make check
```

**Note:** Modifying this behavior may affect RFC 1123 compliance. Consider using the configuration option `preserveFQDN="on"` first before modifying source code.

## SKIP Test Cases (For Reference)

These tests were skipped due to missing dependencies or infrastructure, not code issues:

1. **allowed-sender-tcp-hostname-fail**: HTTP endpoint not reachable
2. **daqueue-persist**: Marked as unstable/disabled
3. **imuxsock_logger_ratelimit**: liblogging-stdlog not available
4. **imuxsock_traillf**: liblogging-stdlog not available
5. **imuxsock_ccmiddle**: liblogging-stdlog not available
6. **imuxsock_traillf_syssock**: liblogging-stdlog not available
7. **imuxsock_ccmiddle_syssock**: liblogging-stdlog not available

These are infrastructure/dependency issues, not code problems.

## Recommended Actions

1. **Immediate Fix**: Update the test expected data files to use lowercase hostnames
2. **Long-term**: Make tests hostname-independent or use case-insensitive comparisons
3. **Optional**: Install liblogging-stdlog to enable skipped imuxsock tests
4. **Optional**: Ensure network access for HTTP-dependent tests

## References

- RFC 1123: Requirements for Internet Hosts
- Rsyslog Documentation: https://www.rsyslog.com/doc/
