# MCX DTE Logic Fix & Portfolio Configuration Resolution

## 📋 Overview

This document covers two major fixes implemented in the backtester system:

1. **MCX DTE (Days to Expiry) Logic Fix** - Fixed incorrect DTE calculation for MCX instruments
2. **Portfolio Configuration Issue Resolution** - Fixed multiple input files not being processed

---

## 🔧 Fix 1: MCX DTE Logic Implementation

### Problem Statement
The DTE (Days to Expiry) logic for MCX instruments like CRUDEOIL was not working correctly:
- **DTE = 1**: Only 1 day before expiry (correct)
- **DTE = 10**: Running on all days (incorrect)
- **DTE ≥ 30**: No proper handling for "run on all days" threshold

### Root Cause
1. **Missing DTE Parameter**: In `BTRUN/Util.py`, DTE was only passed to backend if DTE < 5
2. **Incorrect Logic**: MCX instruments needed different expiry calculation than NSE/BSE
3. **Missing Null Checks**: Database functions had missing return statements causing crashes

### Solution Implemented

#### 1. Fixed DTE Parameter Passing
**File**: `BTRUN/Util.py`
```python
# OLD CODE (lines 1432-1435)
if int(frontendStgyParameters['DTE']) < 5:
    stgyParaa.update({
        "dte": int(frontendStgyParameters['DTE'])
    })

# NEW CODE
# Always pass DTE to backend - no restriction on DTE value
stgyParaa.update({
    "dte": int(frontendStgyParameters['DTE'])
})
```

#### 2. Fixed MCX DTE Logic in Backtester
**File**: `app/backtester/__init__.py`
```python
# Handle DTE calculation differently for MCX vs NSE/BSE
if strategy.dte != None:
    if strategy.index.name in ["COPPER", "CRUDEOIL", "CRUDEOILM", "GOLD", "GOLDM", "NATGASMINI", "NATURALGAS", "NICKEL", "SILVER", "SILVERM", "ZINC"]:
        # MCX instruments: Use monthly expiry dates
        if strategy.dte >= 30:
            # DTE >= 30: Run on all trading days
            continue  # Don't skip any days
        else:
            # DTE < 30: Run only on specific days before expiry
            # Calculate exact DTE days before expiry
            target_expiry = get_monthly_expiry_date(strategy.index, current_date)
            if target_expiry is None:
                continue
            
            # Calculate DTE: days between current_date and target_expiry
            date_index = all_trading_dates.index(current_date)
            expiry_index = all_trading_dates.index(target_expiry)
            
            if strategy.dte != (expiry_index - date_index):
                continue  # Skip this day
```

#### 3. Fixed Portfolio DTE Logic
**File**: `app/pm/__init__.py`
```python
# Similar MCX-specific DTE logic for portfolio backtesting
if strategy.dte is not None:
    if strategy.index.name in ["COPPER", "CRUDEOIL", "CRUDEOILM", "GOLD", "GOLDM", "NATGASMINI", "NATURALGAS", "NICKEL", "SILVER", "SILVERM", "ZINC"]:
        # MCX instruments: Use monthly expiry dates
        if strategy.dte >= 30:
            # DTE >= 30: Run on all trading days
            continue  # Don't skip any days
        else:
            # DTE < 30: Run only on specific days before expiry
            # Calculate exact DTE days before expiry
            target_expiry = get_monthly_expiry_date(strategy.index, current_date)
            if target_expiry is None:
                continue
            
            # Calculate DTE: days between current_date and target_expiry
            date_index = all_trading_dates.index(current_date)
            expiry_index = all_trading_dates.index(target_expiry)
            
            if strategy.dte != (expiry_index - date_index):
                continue  # Skip this day
```

#### 4. Fixed Database Null Checks
**File**: `app/database/utils.py`
```python
# Fixed missing return statements
except KeyError:
    if time < MARKET_START_TIME[exchange]:
        return None  # Was: None
    else:
        return get_closest_valid_strike_price(...)
```

**File**: `app/evaluator/utils.py`
```python
# Added null checks for ATM price
atm = get_closest_valid_strike_price(index, current_date, trading_timestamp, leg_expiry, atm, leg.option_type, is_live)

# If ATM is None, skip this leg as data is not available
if atm is None:
    return None
```

### Expected Behavior After Fix

| DTE Value | Behavior |
|-----------|----------|
| **DTE = 1** | Trade only on 1 day before expiry |
| **DTE = 10** | Trade only on 10 days before expiry |
| **DTE ≥ 30** | Trade on ALL trading days in date range |

### Example Calculation
- **Expiry Date**: 16-6-2025
- **DTE = 1**: Trade only on **15-6-2025**
- **DTE = 10**: Trade only on **6-6-2025** (10 trading days before expiry)
- **DTE = 35**: Trade on **ALL days** in the specified date range

---

## 🔧 Fix 2: Portfolio Configuration Issue Resolution

### Problem Statement
When running portfolio backtest with multiple input files, only 1 file was processed instead of all 9 configured files.

### Root Cause Analysis
**File**: `BTRUN/INPUT SHEETS/INPUT PORTFOLIO CRUDEOIL.xlsx`

**PortfolioSetting Sheet** (✅ Correct):
- 9 portfolios all enabled with "YES"

**StrategySetting Sheet** (❌ Problem):
- Only 1 strategy had "Enabled: YES"
- 8 strategies had "Enabled: nan" (empty/null values)

### Solution Implemented

#### 1. Diagnostic Script
Created `check_portfolio_config.py` to analyze the issue:
```python
# Check enabled strategies
enabled_strategies = strategy_df[strategy_df['Enabled'].str.upper() == 'YES']
print(f"✅ ENABLED STRATEGIES: {len(enabled_strategies)}")

# Check valid strategies (enabled AND file exists)
valid_strategies = strategy_df[(strategy_df['Enabled'].str.upper() == 'YES') & (strategy_df['fileExists'] == True)]
print(f"🎯 VALID STRATEGIES (Enabled + File Exists): {len(valid_strategies)}")
```

#### 2. Fix Script
Created `fix_portfolio_config.py` to resolve the issue:
```python
# Fix the Enabled column - set all to "YES"
strategy_df['Enabled'] = 'YES'

# Save the updated file
with pd.ExcelWriter(portfolio_file, engine='openpyxl', mode='a', if_sheet_exists='replace') as writer:
    strategy_df.to_excel(writer, sheet_name='StrategySetting', index=False)
```

### Results

**Before Fix:**
- ✅ Enabled strategies: **1**
- ❌ Valid strategies: **1**
- ⚠️ Only "CRUDE 1601-1702" was processed

**After Fix:**
- ✅ Enabled strategies: **9**
- ✅ Valid strategies: **9**
- 🎯 All 9 strategies now processed:
  - CRUDE 1601-1702
  - CRUDE 1604-1924  
  - CRUDE 1703-1803
  - CRUDE 1804-1904
  - CRUDE 1905-2005
  - CRUDE 1928-2325
  - CRUDE 2006-2106
  - CRUDE 2107-2207
  - CRUDE 2209-2325

---

## 📁 Files Modified

### DTE Logic Fix
1. **`BTRUN/Util.py`** - Fixed DTE parameter passing
2. **`app/backtester/__init__.py`** - Fixed MCX DTE logic
3. **`app/pm/__init__.py`** - Fixed portfolio DTE logic
4. **`app/database/utils.py`** - Fixed missing return statements
5. **`app/evaluator/utils.py`** - Added null checks for ATM price

### Portfolio Configuration Fix
1. **`BTRUN/INPUT SHEETS/INPUT PORTFOLIO CRUDEOIL.xlsx`** - Updated StrategySetting sheet
2. **`BTRUN/INPUT SHEETS/INPUT PORTFOLIO CRUDEOIL_backup.xlsx`** - Created backup

---

## 🚀 Usage Instructions

### Running Portfolio Backtest
```bash
python BTRUN/BTRunPortfolio.py
```

**Expected Output:**
- ✅ Loaded **9 portfolio(s)** for backtesting
- 🔄 Each strategy processed individually
- 📊 Multiple Excel output files generated

### DTE Testing
Test different DTE values for MCX instruments:
- **DTE = 1**: Should trade only 1 day before expiry
- **DTE = 10**: Should trade only 10 days before expiry  
- **DTE = 35**: Should trade on all days in date range

---

## 🔍 Troubleshooting

### If DTE Still Not Working
1. Check if DTE parameter is being passed to backend
2. Verify MCX instrument names are in the supported list
3. Check database expiry data availability

### If Portfolio Still Processing Only 1 File
1. Verify "Enabled" column in StrategySetting sheet
2. Check if strategy Excel files exist at specified paths
3. Ensure both PortfolioSetting and StrategySetting sheets are configured

---

## 📊 Key Takeaways

1. **DTE Logic**: MCX instruments use monthly expiries, different from NSE/BSE weekly expiries
2. **Portfolio Configuration**: Both "PortfolioSetting" and "StrategySetting" sheets must be properly configured
3. **File Validation**: Strategy files must exist and be enabled to be processed
4. **Error Handling**: Added proper null checks to prevent crashes

---

## 📝 Notes

- **Backup Created**: Original portfolio file backed up as `INPUT PORTFOLIO CRUDEOIL_backup.xlsx`
- **MCX Instruments**: COPPER, CRUDEOIL, CRUDEOILM, GOLD, GOLDM, NATGASMINI, NATURALGAS, NICKEL, SILVER, SILVERM, ZINC
- **DTE Threshold**: 30 days is the threshold for "run on all days" vs "specific days"
- **Portfolio System**: Processes multiple strategies within a single portfolio configuration file

---

*Last Updated: July 12, 2025* 