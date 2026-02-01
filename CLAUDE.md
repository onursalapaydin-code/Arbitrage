# CLAUDE.md - BIST Arbitrage Trading System

> **Purpose:** This document provides AI assistants with comprehensive guidance for understanding and working with this codebase.
> **Version:** v19.4 (2026-02-01)
> **Related:** See `CLAUDE_TALIMAT_v19_4.md` for detailed Turkish documentation.

---

## Quick Reference

| Item | Value |
|------|-------|
| **Project** | BIST Arbitrage Trading System (SN-NF-PAIR) |
| **Language** | C# 5.0 (IdealData platform) |
| **Secondary** | Python (scrapers), HTML/JS (dashboard) |
| **Main File** | `ArbCodeFormVersion_v19_4.txt` (~10,600 lines) |
| **Working Dir** | `D:\arbit\` (Windows paths in code) |

---

## 1. Project Overview

This is a **production algorithmic trading system** for Borsa İstanbul (BIST) Turkish Stock Exchange. It implements three arbitrage strategies:

1. **Spot-Near (SN)** - Primary arbitrage: Buy SPOT + Sell VIOP Near Future
2. **Near-Far (NF)** - Calendar spread: Buy Near + Sell Far futures
3. **Pair Trading** - Cointegration-based VIOP Near-Near pairs

### Platform Information

- **Platform:** IdealData Portföy (NOT IdealPro, NOT MatriksIQ)
- **C# Version:** 5.0 - **Critical language constraints apply**
- **Execution:** Code is pasted into IdealData's script editor

---

## 2. Repository Structure

```
/home/user/Arbitrage/
├── ArbCodeFormVersion_v19_4.txt    # Main C# robot code (~10,600 lines)
├── index.html                      # Web dashboard (JS/HTML)
├── CLAUDE_TALIMAT_v19_4.md         # Turkish documentation
├── CLAUDE.md                       # This file
└── .git/                           # Version control

Runtime files (D:\arbit\ on Windows):
├── settings.json                   # Robot configuration
├── positions.json                  # Open positions state
├── teminat.json                    # Margin/collateral values
├── Temettu.json                    # Dividend calendar
├── Temettu_Scraper.py              # Dividend data scraper
├── srm.json                        # Capital increase calendar
├── srm_scraper.py                  # Capital increase scraper
├── gist_credentials.json           # GitHub Gist auth (SENSITIVE)
├── stats_history.json              # Historical P&L records
└── logs/arb.log                    # Rolling log (1000 lines)
```

---

## 3. Critical C# 5.0 Constraints

**IdealData uses C# 5.0. These features are NOT supported:**

### Do NOT Use

```csharp
// Null-conditional operator (C# 6.0+)
var x = obj?.Property ?? default;  // COMPILER ERROR

// String interpolation (C# 6.0+)
var s = $"Value: {x}";             // COMPILER ERROR

// nameof operator (C# 6.0+)
nameof(variable)                    // COMPILER ERROR
```

### Use Instead

```csharp
// Explicit null checks
var x = obj != null ? obj.Property : default;

// String concatenation
var s = "Value: " + x.ToString();

// Null-safe depth reading pattern
var nearBid = nearDepth.Bids != null && nearDepth.Bids.Length > 0 && nearDepth.Bids[0] != null
    ? (decimal)nearDepth.Bids[0].Price : 0m;
```

---

## 4. Key Code Patterns

### Forward Declaration Requirement

Functions must be defined before use. These functions are at the top (~line 350):
- `IsDividendDay`
- `IsAfterOpeningVolatility`
- `CheckSermayeArtirimi`
- `IsSermayeArtirimiBlocked`

**Note:** These forward-declared functions cannot use `Log()` since it's not yet defined.

### Symbol Normalization

```csharp
// F_THYAO0226 -> THYAO
string normalized = symbol.ToUpper();
if (normalized.StartsWith("F_")) {
    normalized = normalized.Substring(2);
    if (normalized.Length > 4)
        normalized = normalized.Substring(0, normalized.Length - 4);
}
```

### Safe File Writing

```csharp
using (var fs = new FileStream(path, FileMode.Create, FileAccess.Write, FileShare.ReadWrite)) {
    var bytes = Encoding.UTF8.GetBytes(content);
    fs.Write(bytes, 0, bytes.Length);
}
```

---

## 5. Data Structures

### Position Dictionaries

```csharp
// Spot-Near positions
snPos[symbol] = [spotQty, nearQty, spotEntry, nearEntry, entrySpread,
                  expectedPnl, addCount, entryTicks]

// Near-Far positions
nfPos[symbol] = [nearLot, farLot, nearEntry, farEntry, spreadEntry,
                  expectedPnl, daysN, nearC, farC]

// Pair Trading positions
pairPos["SYM_A-SYM_B"] = [qtyA, qtyB, avgA, avgB, openZ, tier,
                          expectedPnl, currentZ, ...]
```

---

## 6. Market Hours & Schedule

### Trading Hours

| Market | Hours |
|--------|-------|
| BIST Pay | 10:00 - 18:00 (continuous, NO lunch break) |
| VIOP | 09:30 - 18:10 (continuous) |
| Robot Active | 10:00 - 18:09 |
| Pair Trading | 10:05 - 17:59 (SPOT price dependency) |

### Scheduled Operations

| Time | Operation |
|------|-----------|
| 09:20 | Margin (teminat) file check |
| 09:25 | Dividend + Capital increase update |
| 09:56 | Depth cache warmup |
| 10:01 | Robot starts |
| 10:05 | Opening volatility ends, entries allowed |
| 10:10 | Capital increase position auto-close |
| 11:50 | Position synchronization |
| 17:35 | Position synchronization |
| 17:59 | Pair trading stops |
| 18:09 | Robot stops |

### Contract Expiry

- **Last trading day:** Last business day of each month
- If month ends on weekend, previous Friday is last day

---

## 7. Risk Management

### Protection Systems

| Protection | Behavior |
|------------|----------|
| **Dividend** | Block entry on ex-dividend dates |
| **Capital Increase** | Block entry if today OR within 7 days; auto-close at 10:10 |
| **Opening Volatility** | 5-minute wait after 10:00 |
| **Margin Levels** | Warning (10%), Danger (5%), Critical (0%) |
| **Daily Loss** | 10,000 TL maximum |
| **Circuit Breaker** | Stop NF if index -5% or stock -8% |

### Pair Trading Blocked Symbols

```csharp
PAIR_BLOCKED = ["TCELL", "TTKOM", "CIMSA", "OYAKC"]
```
These symbols are excluded from SN and NF strategies when Pair Trading is enabled.

---

## 8. External Dependencies

### Python Scrapers

```bash
# Required packages
pip install requests beautifulsoup4 pdfplumber

# Manual execution
python D:\arbit\Temettu_Scraper.py          # Dividend data
python D:\arbit\srm_scraper.py --force      # Capital increase data
```

### IdealData Platform API

Key functions from `Sistem` namespace:
- `Sistem.ViopHesapOku()` - Read VIOP account
- `Sistem.BistHesapOku()` - Read BIST account
- `Sistem.SonFiyat(symbol)` - Get current price
- `Sistem.Emir(...)` - Place order
- `Sistem.EmirIptal(...)` - Cancel order
- `Sistem.GrafikVerileriniOku()` - Read chart data
- `Sistem.BaglantiVar` - Connection status

### GitHub Gist API

Used for remote dashboard and control:
- Dashboard reads from public Gist
- STOP/PAUSE commands via Gist file updates

---

## 9. Development Workflow

### Making Changes

1. **Read the existing code** before making modifications
2. **Respect C# 5.0 constraints** - no null-conditional, no string interpolation
3. **Forward declare** any new utility functions if called early in execution
4. **Test with IS_TEST flag** before production
5. **Update positions.json structure** if adding new position fields

### Code Organization

- Main loop: ~lines 10363-10610
- Strategy functions: Throughout file
- Forward declarations: ~lines 350+
- Settings/config: Near top of file
- Form UI code: Near end of file

### Version Control

- Code stored as `.txt` file for easy copy-paste to IdealData
- Git history tracks changes
- Version number in filename and comments

### Stopping the Robot

- **Local:** Create `STOP.txt` file + select "Yok" in IdealData
- **Local:** Click "Çıkış" button on Form
- **Remote:** STOP button on Dashboard (via Gist)

---

## 10. Common Errors & Solutions

### Compiler Errors

| Error | Cause | Solution |
|-------|-------|----------|
| CS1525 "Invalid expression term '.'" | Null-conditional `?.` used | Use explicit null check |
| CS0841 "Local variable used before declaration" | Function not forward-declared | Move function definition up |
| CS0841 "Log" | Log() called in forward declaration | Remove Log from forward-declared functions |

### Runtime Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Yetersiz veri" | Chart window not open | Open daily charts for TCELL, TTKOM, CIMSA, OYAKC |
| VIOP null in Recovery | API connection issue | Wait for connection, auto-recovery triggers |
| API returns NULL | Rate limiting | Use 2-second cache wrapper functions |

---

## 11. Conventions for AI Assistants

### When Modifying Code

1. **Always specify affected functions** before making changes
2. **Avoid over-engineering** - only change what's requested
3. **Respect existing patterns** - use same null-check style, same logging format
4. **Don't add features** beyond what's requested
5. **Don't add comments/docstrings** to unchanged code

### When Analyzing Code

1. **Use `*` at end of prompt** for analysis-only (no code changes)
2. **Provide file links** for code references: `ArbCodeFormVersion_v19_4.txt:712`
3. **Check CLAUDE_TALIMAT_v19_4.md** for detailed Turkish documentation

### Naming Conventions

- Variables/functions: camelCase (`snPos`, `nfEnabled`)
- Constants: UPPERCASE (`VIOP_BUDGET`, `BB_PERIOD`)
- Prefixes: `daily` for per-day stats, `total` for cumulative
- Turkish names are common (`teminat`, `yatırım`)

---

## 12. Version History

| Version | Date | Changes |
|---------|------|---------|
| v19.4 | 2026-02-01 | Capital increase protection system |
| v19.3 | 2026-01-31 | Dividend protection, opening volatility, async Gist |
| v19.2 | 2026-01-31 | Pair rollover fixes, Z-Score dashboard |
| v19.1 | 2026-01-31 | Tiered budget allocation for Pair Trading |
| v19 | 2026-01-30 | Pair Trading strategy introduction |
| v18 | 2026-01-30 | Dividend management, early exit formula |

---

## 13. Build & Deploy

There is no traditional build system. The workflow is:

1. Edit code in `ArbCodeFormVersion_v19_4.txt`
2. Copy entire file content
3. Paste into IdealData Portföy script editor
4. IdealData compiles and runs the C# code

### Testing

- Set `IS_TEST = true` for test mode (uses static balance data)
- Test mode avoids real broker operations

---

## 14. Configuration Files

### settings.json

Runtime configuration including:
- Budget allocations (VIOP, SPOT, NF, Pair)
- Strategy parameters (BB period, thresholds)
- Margin warning levels
- Market hours and timeouts

### positions.json

Operational state:
- Open SN, NF, and Pair positions
- Orphan legs (incomplete positions)
- Pending orders
- Contract code mappings

### gist_credentials.json

**SENSITIVE - Never commit real tokens**
- GitHub Gist token
- Gist ID for dashboard

---

## Quick Commands Reference

```bash
# View recent log
tail -100 D:\arbit\logs\arb.log

# Run dividend scraper
python D:\arbit\Temettu_Scraper.py

# Run capital increase scraper
python D:\arbit\srm_scraper.py --force

# Stop robot remotely
# Update Gist file with STOP command
```

---

*For detailed Turkish documentation, see `CLAUDE_TALIMAT_v19_4.md`*
