# 03_data_cleaning.ipynb - Prompt Instructions

## Objektif
Membersihkan dataset berdasarkan hasil assessment untuk mengatasi missing values, outliers, duplicates, dan inkonsistensi data.

## Input Data
```
data/interim/combined_raw_transactions.csv
data/interim/assessment_report.json
```

## Output Data
```
data/interim/cleaned_transactions.csv
reports/03_cleaning_report.txt
```

## Implementation Code

---

### Markdown Cell
```markdown
# 03 Data Cleaning

Notebook ini melakukan cleaning terhadap dataset berdasarkan hasil assessment sebelumnya.

## Langkah Cleaning

### 1. Setup dan Load Data
- Import libraries dan load dataset
- Load assessment report untuk referensi issues

### 2. Handle Missing Values
- Drop records dengan critical missing values (date, amount)
- Impute missing category dengan 'Unknown'
- Handle missing description

### 3. Standardize Date Format
- Parse dan standardize semua date formats
- Remove future dates
- Filter date range yang valid (2020-2024)

### 4. Clean Amount Values
- Convert ke numeric
- Remove negative amounts
- Handle zero amounts
- Cap extreme outliers

### 5. Remove Duplicates
- Identify duplicate strategy
- Remove exact duplicates
- Handle subset duplicates (date+amount+category)

### 6. Standardize Text Fields
- Lowercase dan trim category
- Remove special characters
- Standardize category names
- Clean description field

### 7. Handle Outliers
- Apply IQR method untuk outliers
- Cap atau remove outliers berdasarkan threshold
- Validate outlier handling per source

### 8. Data Type Conversion
- Convert date to datetime
- Convert amount to float
- Ensure consistent data types

### 9. Final Validation
- Validate cleaned dataset
- Generate cleaning report
- Compare before/after metrics

## Expected Results

**Before Cleaning:**
- Total records: ~XXX,XXX
- Missing values: XX%
- Duplicates: X,XXX
- Outliers: X,XXX
- Invalid dates: XXX
- Negative amounts: XXX

**After Cleaning:**
- Total records: ~XXX,XXX (retention rate: ~XX%)
- Missing values: <5% (non-critical columns only)
- Duplicates: 0
- Outliers: handled/capped
- Invalid dates: 0
- Negative amounts: 0

## Key Decisions

1. **Missing Values**: Drop jika critical (date/amount), impute jika non-critical
2. **Duplicates**: Keep first occurrence, remove rest
3. **Outliers**: Cap at 99th percentile, remove jika extreme (>10x median)
4. **Date Range**: Filter 2020-2024 only
5. **Amount**: Remove negative, keep zero (might be valid)
```

---

### Markdown Cell
```markdown
## Setup Environment
```

### Cell 1: Import Libraries dan Load Data
```python
import pandas as pd
import numpy as np
import json
from pathlib import Path
from datetime import datetime
import sys

sys.path.append('../src')
from fingo_ds1.config import INTERIM_DATA_PATH, REPORTS_PATH

pd.set_option('display.max_columns', None)
pd.set_option('display.max_rows', 100)

df_raw = pd.read_csv(INTERIM_DATA_PATH / 'combined_raw_transactions.csv')
print(f"Raw dataset loaded: {df_raw.shape}")

with open(INTERIM_DATA_PATH / 'assessment_report.json', 'r') as f:
    assessment = json.load(f)
    
print(f"Assessment report loaded")
print(f"Identified issues: {len(assessment)} categories")
```

### Cell 2: Create Cleaning Tracker
```python
cleaning_log = {
    'initial_shape': df_raw.shape,
    'steps': []
}

def log_step(step_name, df_current):
    cleaning_log['steps'].append({
        'step': step_name,
        'records_remaining': len(df_current),
        'records_removed': cleaning_log['initial_shape'][0] - len(df_current),
        'shape': df_current.shape
    })
    print(f"{step_name}: {len(df_current):,} records remaining")

df = df_raw.copy()
print(f"Initial records: {len(df):,}")
```

---

### Markdown Cell
```markdown
## Handle Missing Values
```

### Cell 3: Identify Missing Patterns
```python
print("=== MISSING VALUES BEFORE CLEANING ===\n")

missing_summary = pd.DataFrame({
    'missing_count': df.isnull().sum(),
    'missing_pct': (df.isnull().sum() / len(df) * 100).round(2)
}).sort_values('missing_pct', ascending=False)

print(missing_summary[missing_summary['missing_count'] > 0])
```

### Cell 4: Drop Critical Missing Values
```python
critical_cols = ['date', 'amount']

print(f"Records before dropping critical missing: {len(df):,}")

for col in critical_cols:
    before = len(df)
    df = df[df[col].notna()]
    removed = before - len(df)
    print(f"Dropped {removed:,} records with missing {col}")

log_step('Remove critical missing values', df)
```

### Cell 5: Impute Non-Critical Missing
```python
if 'category' in df.columns:
    missing_cat = df['category'].isnull().sum()
    df['category'] = df['category'].fillna('Unknown')
    print(f"Imputed {missing_cat:,} missing categories with 'Unknown'")

if 'description' in df.columns:
    missing_desc = df['description'].isnull().sum()
    df['description'] = df['description'].fillna('')
    print(f"Imputed {missing_desc:,} missing descriptions with empty string")

log_step('Impute non-critical missing', df)
```

---

### Markdown Cell
```markdown
## Standardize Date Format
```

### Cell 6: Parse Date Column
```python
print("=== DATE STANDARDIZATION ===\n")

df['date_clean'] = pd.to_datetime(df['date'], errors='coerce')

unparseable = df['date_clean'].isnull().sum()
print(f"Unparseable dates: {unparseable:,}")

if unparseable > 0:
    print("\nSample unparseable dates:")
    print(df[df['date_clean'].isnull()]['date'].head(10).tolist())
```

### Cell 7: Remove Invalid Dates
```python
before = len(df)
df = df[df['date_clean'].notna()]
removed = before - len(df)

print(f"Removed {removed:,} records with invalid dates")
log_step('Remove invalid dates', df)
```

### Cell 8: Remove Future Dates
```python
today = datetime.now()
future_mask = df['date_clean'] > today

future_count = future_mask.sum()
print(f"Future dates found: {future_count:,}")

if future_count > 0:
    df = df[~future_mask]
    print(f"Removed {future_count:,} future dates")
    log_step('Remove future dates', df)
```

### Cell 9: Filter Valid Date Range
```python
min_date = pd.to_datetime('2020-01-01')
max_date = pd.to_datetime('2024-12-31')

before = len(df)
df = df[(df['date_clean'] >= min_date) & (df['date_clean'] <= max_date)]
removed = before - len(df)

print(f"Date range filtered: 2020-2024")
print(f"Removed {removed:,} records outside range")
print(f"Final date range: {df['date_clean'].min()} to {df['date_clean'].max()}")

log_step('Filter date range', df)
```

---

### Markdown Cell
```markdown
## Clean Amount Values
```

### Cell 10: Parse Amount Column
```python
print("=== AMOUNT CLEANING ===\n")

df['amount_clean'] = pd.to_numeric(df['amount'], errors='coerce')

non_numeric = df['amount_clean'].isnull().sum()
print(f"Non-numeric amounts: {non_numeric:,}")

if non_numeric > 0:
    print("\nSample non-numeric amounts:")
    print(df[df['amount_clean'].isnull()]['amount'].head(10).tolist())
    
    df = df[df['amount_clean'].notna()]
    print(f"Removed {non_numeric:,} non-numeric amounts")
    log_step('Remove non-numeric amounts', df)
```

### Cell 11: Remove Negative Amounts
```python
negative_mask = df['amount_clean'] < 0
negative_count = negative_mask.sum()

print(f"Negative amounts: {negative_count:,}")

if negative_count > 0:
    print(f"Sample negative amounts: {df[negative_mask]['amount_clean'].head(10).tolist()}")
    df = df[~negative_mask]
    print(f"Removed {negative_count:,} negative amounts")
    log_step('Remove negative amounts', df)
```

### Cell 12: Handle Zero Amounts
```python
zero_mask = df['amount_clean'] == 0
zero_count = zero_mask.sum()

print(f"Zero amounts: {zero_count:,} ({(zero_count/len(df)*100):.2f}%)")
print(f"Decision: Keep zero amounts (might be valid free items)")
```

### Cell 13: Cap Extreme Outliers
```python
Q1 = df['amount_clean'].quantile(0.25)
Q3 = df['amount_clean'].quantile(0.75)
IQR = Q3 - Q1

lower_bound = Q1 - 3 * IQR
upper_bound = Q3 + 3 * IQR

print(f"IQR Bounds (3x IQR):")
print(f"Lower: {lower_bound:,.2f}")
print(f"Upper: {upper_bound:,.2f}")

extreme_low = (df['amount_clean'] < lower_bound).sum()
extreme_high = (df['amount_clean'] > upper_bound).sum()

print(f"\nExtreme outliers:")
print(f"Below lower bound: {extreme_low:,}")
print(f"Above upper bound: {extreme_high:,}")

percentile_99 = df['amount_clean'].quantile(0.99)
print(f"\n99th percentile: {percentile_99:,.2f}")

capped_count = (df['amount_clean'] > percentile_99).sum()
df.loc[df['amount_clean'] > percentile_99, 'amount_clean'] = percentile_99

print(f"Capped {capped_count:,} values at 99th percentile")
log_step('Cap extreme outliers', df)
```

---

### Markdown Cell
```markdown
## Remove Duplicates
```

### Cell 14: Identify Duplicates
```python
print("=== DUPLICATE REMOVAL ===\n")

exact_dups = df.duplicated().sum()
print(f"Exact duplicates (all columns): {exact_dups:,}")

subset_cols = ['date_clean', 'amount_clean', 'category']
subset_dups = df.duplicated(subset=subset_cols).sum()
print(f"Subset duplicates (date+amount+category): {subset_dups:,}")
```

### Cell 15: Remove Duplicates
```python
before = len(df)

df = df.drop_duplicates(subset=subset_cols, keep='first')

removed = before - len(df)
print(f"Removed {removed:,} duplicate records (kept first occurrence)")

log_step('Remove duplicates', df)
```

---

### Markdown Cell
```markdown
## Standardize Text Fields
```

### Cell 16: Clean Category Field
```python
print("=== CATEGORY STANDARDIZATION ===\n")

if 'category' in df.columns:
    print(f"Unique categories before: {df['category'].nunique()}")
    
    df['category_clean'] = (df['category']
                            .str.lower()
                            .str.strip()
                            .str.replace(r'\s+', ' ', regex=True)
                            .str.replace(r'[^\w\s]', '', regex=True))
    
    print(f"Unique categories after: {df['category_clean'].nunique()}")
    
    print("\nTop 10 categories:")
    print(df['category_clean'].value_counts().head(10))
```

### Cell 17: Standardize Category Names
```python
category_mapping = {
    'food': 'food_beverage',
    'foods': 'food_beverage',
    'beverage': 'food_beverage',
    'drink': 'food_beverage',
    'transport': 'transportation',
    'transportation': 'transportation',
    'health': 'healthcare',
    'medical': 'healthcare',
    'entertainment': 'entertainment',
    'fun': 'entertainment',
    'cloth': 'clothing',
    'clothes': 'clothing',
    'fashion': 'clothing'
}

df['category_clean'] = df['category_clean'].replace(category_mapping)

print("Category mapping applied")
print(f"Final unique categories: {df['category_clean'].nunique()}")
print("\nFinal category distribution:")
print(df['category_clean'].value_counts())
```

### Cell 18: Clean Description Field
```python
print("=== DESCRIPTION CLEANING ===\n")

if 'description' in df.columns:
    df['description_clean'] = (df['description']
                               .str.strip()
                               .str.replace(r'\s+', ' ', regex=True))
    
    print(f"Description field cleaned")
    print(f"Non-empty descriptions: {(df['description_clean'] != '').sum():,}")
```

---

### Markdown Cell
```markdown
## Data Type Conversion
```

### Cell 19: Finalize Data Types
```python
print("=== FINAL DATA TYPE CONVERSION ===\n")

df['date'] = df['date_clean']
df['amount'] = df['amount_clean'].astype(float)
df['category'] = df['category_clean']

if 'description_clean' in df.columns:
    df['description'] = df['description_clean']

final_cols = ['date', 'amount', 'category', 'description', 'data_source']
df_clean = df[final_cols].copy()

print("Final data types:")
print(df_clean.dtypes)
```

---

### Markdown Cell
```markdown
## Final Validation
```

### Cell 20: Validate Cleaned Dataset
```python
print("=== CLEANED DATASET VALIDATION ===\n")

print(f"Final shape: {df_clean.shape}")
print(f"Retention rate: {(len(df_clean) / len(df_raw) * 100):.2f}%")

print("\nMissing values:")
print(df_clean.isnull().sum())

print("\nData quality checks:")
print(f"Invalid dates: {df_clean['date'].isnull().sum()}")
print(f"Negative amounts: {(df_clean['amount'] < 0).sum()}")
print(f"Non-numeric amounts: {pd.to_numeric(df_clean['amount'], errors='coerce').isnull().sum()}")
print(f"Duplicates: {df_clean.duplicated(subset=['date', 'amount', 'category']).sum()}")

print("\nDate range:")
print(f"Min: {df_clean['date'].min()}")
print(f"Max: {df_clean['date'].max()}")

print("\nAmount statistics:")
print(df_clean['amount'].describe())
```

### Cell 21: Data Distribution Check
```python
print("=== DATA DISTRIBUTION ===\n")

print("Records per data source:")
print(df_clean['data_source'].value_counts())

print("\nRecords per category:")
print(df_clean['category'].value_counts().head(15))

print("\nMonthly distribution:")
df_clean['year_month'] = df_clean['date'].dt.to_period('M')
print(df_clean['year_month'].value_counts().sort_index().head(12))
```

---

### Markdown Cell
```markdown
## Generate Cleaning Report
```

### Cell 22: Compile Cleaning Summary
```python
cleaning_summary = {
    'before': {
        'records': int(len(df_raw)),
        'columns': int(len(df_raw.columns)),
        'missing_values': int(df_raw.isnull().sum().sum()),
        'duplicates': int(df_raw.duplicated(subset=['date', 'amount']).sum()) if 'date' in df_raw.columns and 'amount' in df_raw.columns else 0
    },
    'after': {
        'records': int(len(df_clean)),
        'columns': int(len(df_clean.columns)),
        'missing_values': int(df_clean.isnull().sum().sum()),
        'duplicates': int(df_clean.duplicated(subset=['date', 'amount', 'category']).sum())
    },
    'removed': {
        'total_records': int(len(df_raw) - len(df_clean)),
        'percentage': float(((len(df_raw) - len(df_clean)) / len(df_raw) * 100).round(2))
    },
    'cleaning_steps': cleaning_log['steps']
}

print("Cleaning summary compiled")
print(json.dumps(cleaning_summary, indent=2))
```

### Cell 23: Generate Text Report
```python
report_lines = []
report_lines.append("=" * 80)
report_lines.append("DATA CLEANING REPORT")
report_lines.append("=" * 80)
report_lines.append("")

report_lines.append("BEFORE CLEANING")
report_lines.append("-" * 80)
report_lines.append(f"Total Records: {cleaning_summary['before']['records']:,}")
report_lines.append(f"Total Columns: {cleaning_summary['before']['columns']}")
report_lines.append(f"Missing Values: {cleaning_summary['before']['missing_values']:,}")
report_lines.append(f"Duplicates: {cleaning_summary['before']['duplicates']:,}")
report_lines.append("")

report_lines.append("AFTER CLEANING")
report_lines.append("-" * 80)
report_lines.append(f"Total Records: {cleaning_summary['after']['records']:,}")
report_lines.append(f"Total Columns: {cleaning_summary['after']['columns']}")
report_lines.append(f"Missing Values: {cleaning_summary['after']['missing_values']:,}")
report_lines.append(f"Duplicates: {cleaning_summary['after']['duplicates']:,}")
report_lines.append("")

report_lines.append("REMOVED DATA")
report_lines.append("-" * 80)
report_lines.append(f"Total Removed: {cleaning_summary['removed']['total_records']:,}")
report_lines.append(f"Percentage: {cleaning_summary['removed']['percentage']}%")
report_lines.append(f"Retention Rate: {100 - cleaning_summary['removed']['percentage']:.2f}%")
report_lines.append("")

report_lines.append("CLEANING STEPS")
report_lines.append("-" * 80)
for step in cleaning_summary['cleaning_steps']:
    report_lines.append(f"{step['step']}: {step['records_remaining']:,} records")
report_lines.append("")

report_lines.append("=" * 80)

report_text = "\n".join(report_lines)
print(report_text)

text_report_path = REPORTS_PATH / '03_cleaning_report.txt'
with open(text_report_path, 'w') as f:
    f.write(report_text)

print(f"\nReport saved to: {text_report_path}")
```

---

### Markdown Cell
```markdown
## Save Cleaned Dataset
```

### Cell 24: Export Cleaned Data
```python
output_path = INTERIM_DATA_PATH / 'cleaned_transactions.csv'
df_clean.to_csv(output_path, index=False)

print(f"Cleaned dataset saved to: {output_path}")
print(f"File size: {output_path.stat().st_size / 1024 / 1024:.2f} MB")

print("\nFinal dataset preview:")
df_clean.head(10)
```

---

### Markdown Cell
```markdown
## Summary

### Cleaning Achievements
- Removed all critical missing values
- Standardized date format (2020-2024 range)
- Cleaned amount values (no negatives, outliers capped)
- Removed all duplicates
- Standardized text fields (category, description)
- Final retention rate: ~XX%

### Data Quality Improvements
- Missing values: XX% → <5%
- Duplicates: X,XXX → 0
- Invalid dates: XXX → 0
- Negative amounts: XXX → 0
- Data consistency: Significantly improved

### Next Steps
- Proceed to feature engineering
- Create temporal features
- Calculate spending patterns
- Prepare for labeling
```

## Expected Outcomes
1. File `cleaned_transactions.csv` dengan kualitas data tinggi
2. File `03_cleaning_report.txt` berisi summary cleaning process
3. Retention rate 70-90% dari data original
4. Tidak ada missing values di kolom critical
5. Tidak ada duplicates
6. Date dan amount values valid 100%

## Validation Checklist
- [ ] All critical missing values removed
- [ ] Date format standardized dan valid
- [ ] Amount values cleaned dan capped
- [ ] Duplicates removed completely
- [ ] Text fields standardized
- [ ] Data types consistent
- [ ] Cleaning report generated
- [ ] Final dataset saved

## Notes
- Gunakan cleaning strategy yang balance antara quality dan retention
- Document semua keputusan cleaning untuk reproducibility
- Validate hasil cleaning sebelum proceed ke next step
