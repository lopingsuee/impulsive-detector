# 02_data_assessing.ipynb - Prompt Instructions

## Objektif
Melakukan assessment menyeluruh terhadap dataset gabungan untuk mengidentifikasi masalah data quality, missing values, outliers, dan anomali yang perlu ditangani di tahap cleaning.

## Input Data
```
data/interim/combined_raw_transactions.csv
```

## Output Data
```
data/interim/assessment_report.json
reports/02_data_quality_report.txt
```

## Implementation Code

---

### Markdown Cell
```markdown
# 02 Data Assessing

Notebook ini melakukan assessment menyeluruh terhadap dataset gabungan hasil dari data gathering.

**Fokus Assessment:**
- Data quality issues
- Missing values pattern
- Outliers detection
- Data consistency
- Duplicate records
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
import sys

sys.path.append('../src')
from fingo_ds1.config import INTERIM_DATA_PATH, REPORTS_PATH

pd.set_option('display.max_columns', None)
pd.set_option('display.max_rows', 100)

df = pd.read_csv(INTERIM_DATA_PATH / 'combined_raw_transactions.csv')
print(f"Dataset loaded: {df.shape}")
df.head()
```

---

### Markdown Cell
```markdown
## Dataset Overview
```

### Cell 2: Basic Information
```python
print("=== DATASET BASIC INFO ===\n")
print(f"Total records: {len(df):,}")
print(f"Total columns: {len(df.columns)}")
print(f"Memory usage: {df.memory_usage(deep=True).sum() / 1024**2:.2f} MB")

print("\nColumns:")
for i, col in enumerate(df.columns, 1):
    print(f"{i}. {col} ({df[col].dtype})")
```

### Cell 3: Data Types Check
```python
print("=== DATA TYPES ASSESSMENT ===\n")

dtypes_summary = df.dtypes.value_counts()
print("Data types distribution:")
print(dtypes_summary)

print("\nColumns by type:")
for dtype in dtypes_summary.index:
    cols = df.select_dtypes(include=[dtype]).columns.tolist()
    print(f"\n{dtype}:")
    print(cols)
```

---

### Markdown Cell
```markdown
## Missing Values Analysis
```

### Cell 4: Missing Values Overview
```python
print("=== MISSING VALUES ANALYSIS ===\n")

missing = df.isnull().sum()
missing_pct = (missing / len(df) * 100).round(2)

missing_df = pd.DataFrame({
    'column': df.columns,
    'missing_count': missing.values,
    'missing_pct': missing_pct.values,
    'present_count': len(df) - missing.values
}).sort_values('missing_pct', ascending=False)

print(missing_df)
print(f"\nColumns with missing values: {(missing_df['missing_pct'] > 0).sum()}")
```

### Cell 5: Missing Pattern by Source
```python
print("=== MISSING PATTERN BY DATA SOURCE ===\n")

for source in df['data_source'].unique():
    print(f"\n{source.upper()}:")
    df_source = df[df['data_source'] == source]
    missing_source = df_source.isnull().sum()
    missing_pct_source = (missing_source / len(df_source) * 100).round(2)
    
    missing_info = missing_pct_source[missing_pct_source > 0].sort_values(ascending=False)
    if len(missing_info) > 0:
        print(missing_info)
    else:
        print("No missing values")
```

### Cell 6: Critical Columns Missing Check
```python
critical_cols = ['date', 'amount', 'category', 'data_source']

print("=== CRITICAL COLUMNS ASSESSMENT ===\n")

for col in critical_cols:
    missing_count = df[col].isnull().sum()
    missing_pct = (missing_count / len(df) * 100).round(2)
    print(f"{col}:")
    print(f"  Missing: {missing_count:,} ({missing_pct}%)")
    print(f"  Present: {len(df) - missing_count:,}")
```

---

### Markdown Cell
```markdown
## Data Quality Issues
```

### Cell 7: Date Column Assessment
```python
print("=== DATE COLUMN ASSESSMENT ===\n")

df['date_parsed'] = pd.to_datetime(df['date'], errors='coerce')

print(f"Original date column type: {df['date'].dtype}")
print(f"Unparseable dates: {df['date_parsed'].isnull().sum():,}")

print("\nDate range:")
print(f"Min: {df['date_parsed'].min()}")
print(f"Max: {df['date_parsed'].max()}")
print(f"Span: {(df['date_parsed'].max() - df['date_parsed'].min()).days} days")

print("\nSample invalid dates:")
invalid_dates = df[df['date_parsed'].isnull()]['date'].head(10)
print(invalid_dates.tolist())
```

### Cell 8: Future Dates Detection
```python
from datetime import datetime

today = datetime.now()
df['is_future_date'] = df['date_parsed'] > today

future_count = df['is_future_date'].sum()
print(f"Records with future dates: {future_count:,}")

if future_count > 0:
    print("\nSample future dates:")
    print(df[df['is_future_date']][['date', 'date_parsed', 'data_source']].head(10))
```

### Cell 9: Amount Column Assessment
```python
print("=== AMOUNT COLUMN ASSESSMENT ===\n")

df['amount_parsed'] = pd.to_numeric(df['amount'], errors='coerce')

print("Amount statistics:")
print(df['amount_parsed'].describe())

print("\nAmount issues:")
print(f"Non-numeric: {df['amount_parsed'].isnull().sum():,}")
print(f"Negative values: {(df['amount_parsed'] < 0).sum():,}")
print(f"Zero values: {(df['amount_parsed'] == 0).sum():,}")
print(f"Extremely high (>1M): {(df['amount_parsed'] > 1_000_000).sum():,}")
```

### Cell 10: Amount Distribution by Source
```python
print("=== AMOUNT DISTRIBUTION BY SOURCE ===\n")

for source in df['data_source'].unique():
    print(f"\n{source.upper()}:")
    amounts = df[df['data_source'] == source]['amount_parsed'].dropna()
    print(amounts.describe())
```

---

### Markdown Cell
```markdown
## Duplicate Detection
```

### Cell 11: Duplicate Records Check
```python
print("=== DUPLICATE RECORDS ASSESSMENT ===\n")

print(f"Total duplicates (all columns): {df.duplicated().sum():,}")

subset_cols = ['date', 'amount', 'category']
print(f"Duplicates (date+amount+category): {df.duplicated(subset=subset_cols).sum():,}")

print("\nDuplicate distribution by source:")
for source in df['data_source'].unique():
    df_source = df[df['data_source'] == source]
    dup_count = df_source.duplicated(subset=subset_cols).sum()
    dup_pct = (dup_count / len(df_source) * 100).round(2)
    print(f"{source}: {dup_count:,} ({dup_pct}%)")
```

### Cell 12: Duplicate Records Sample
```python
duplicates = df[df.duplicated(subset=subset_cols, keep=False)].sort_values(subset_cols)

if len(duplicates) > 0:
    print(f"Total duplicate groups: {len(duplicates):,}")
    print("\nSample duplicate records:")
    print(duplicates.head(20))
else:
    print("No duplicates found")
```

---

### Markdown Cell
```markdown
## Outliers Detection
```

### Cell 13: Statistical Outliers (IQR Method)
```python
def detect_outliers_iqr(series):
    Q1 = series.quantile(0.25)
    Q3 = series.quantile(0.75)
    IQR = Q3 - Q1
    lower_bound = Q1 - 1.5 * IQR
    upper_bound = Q3 + 1.5 * IQR
    return (series < lower_bound) | (series > upper_bound)

print("=== OUTLIERS DETECTION (IQR METHOD) ===\n")

outliers_mask = detect_outliers_iqr(df['amount_parsed'].dropna())
outliers = df[df['amount_parsed'].notna()][outliers_mask]

print(f"Total outliers: {len(outliers):,}")
print(f"Percentage: {(len(outliers) / len(df) * 100):.2f}%")

print("\nOutlier statistics:")
print(outliers['amount_parsed'].describe())

print("\nOutliers by source:")
print(outliers['data_source'].value_counts())
```

### Cell 14: Extreme Values Analysis
```python
print("=== EXTREME VALUES ANALYSIS ===\n")

percentiles = [0.01, 0.05, 0.10, 0.90, 0.95, 0.99]
print("Amount percentiles:")
for p in percentiles:
    value = df['amount_parsed'].quantile(p)
    print(f"P{int(p*100)}: {value:,.2f}")

print("\nTop 10 highest amounts:")
top_amounts = df.nlargest(10, 'amount_parsed')[['date', 'amount_parsed', 'category', 'data_source']]
print(top_amounts)
```

---

### Markdown Cell
```markdown
## Consistency Check
```

### Cell 15: Category Consistency
```python
print("=== CATEGORY CONSISTENCY CHECK ===\n")

print(f"Unique categories: {df['category'].nunique()}")
print("\nCategory distribution:")
print(df['category'].value_counts().head(20))

print("\nCategories with low frequency (<10):")
low_freq = df['category'].value_counts()
print(low_freq[low_freq < 10])
```

### Cell 16: Category Case Sensitivity Issues
```python
print("=== CASE SENSITIVITY ISSUES ===\n")

df['category_lower'] = df['category'].str.lower().str.strip()
print(f"Original unique categories: {df['category'].nunique()}")
print(f"After lowercase: {df['category_lower'].nunique()}")
print(f"Potential duplicates: {df['category'].nunique() - df['category_lower'].nunique()}")

if df['category'].nunique() != df['category_lower'].nunique():
    print("\nSample case duplicates:")
    for cat in df['category_lower'].unique()[:5]:
        variants = df[df['category_lower'] == cat]['category'].unique()
        if len(variants) > 1:
            print(f"{cat}: {variants}")
```

### Cell 17: Text Quality Issues
```python
print("=== TEXT QUALITY ISSUES ===\n")

text_cols = ['category', 'description']

for col in text_cols:
    if col in df.columns:
        print(f"\n{col.upper()}:")
        
        has_special = df[col].str.contains(r'[^a-zA-Z0-9\s]', na=False).sum()
        has_numbers = df[col].str.contains(r'\d', na=False).sum()
        has_whitespace = df[col].str.contains(r'\s{2,}', na=False).sum()
        
        print(f"  Contains special chars: {has_special:,}")
        print(f"  Contains numbers: {has_numbers:,}")
        print(f"  Multiple whitespaces: {has_whitespace:,}")
        
        print(f"  Sample values:")
        print(f"  {df[col].dropna().head(5).tolist()}")
```

---

### Markdown Cell
```markdown
## Data Completeness by Source
```

### Cell 18: Completeness Score per Source
```python
print("=== DATA COMPLETENESS BY SOURCE ===\n")

for source in df['data_source'].unique():
    df_source = df[df['data_source'] == source]
    print(f"\n{source.upper()}:")
    print(f"Total records: {len(df_source):,}")
    
    completeness = {}
    for col in ['date', 'amount', 'category', 'description']:
        if col in df_source.columns:
            complete_pct = (df_source[col].notna().sum() / len(df_source) * 100).round(2)
            completeness[col] = complete_pct
    
    for col, pct in completeness.items():
        print(f"  {col}: {pct}%")
```

---

### Markdown Cell
```markdown
## Generate Assessment Report
```

### Cell 19: Compile Assessment Summary
```python
assessment_report = {
    'dataset_info': {
        'total_records': int(len(df)),
        'total_columns': int(len(df.columns)),
        'memory_mb': float(df.memory_usage(deep=True).sum() / 1024**2),
        'date_range': {
            'min': str(df['date_parsed'].min()),
            'max': str(df['date_parsed'].max())
        }
    },
    'missing_values': {
        'columns_with_missing': int((missing_df['missing_pct'] > 0).sum()),
        'critical_missing': {
            col: {
                'count': int(df[col].isnull().sum()),
                'percentage': float((df[col].isnull().sum() / len(df) * 100).round(2))
            }
            for col in critical_cols
        }
    },
    'data_quality': {
        'unparseable_dates': int(df['date_parsed'].isnull().sum()),
        'future_dates': int(df['is_future_date'].sum()),
        'negative_amounts': int((df['amount_parsed'] < 0).sum()),
        'zero_amounts': int((df['amount_parsed'] == 0).sum()),
        'non_numeric_amounts': int(df['amount_parsed'].isnull().sum())
    },
    'duplicates': {
        'exact_duplicates': int(df.duplicated().sum()),
        'subset_duplicates': int(df.duplicated(subset=subset_cols).sum())
    },
    'outliers': {
        'iqr_outliers': int(len(outliers)),
        'percentage': float((len(outliers) / len(df) * 100).round(2))
    },
    'consistency': {
        'unique_categories': int(df['category'].nunique()),
        'case_duplicates': int(df['category'].nunique() - df['category_lower'].nunique())
    }
}

print("Assessment report compiled")
print(json.dumps(assessment_report, indent=2))
```

### Cell 20: Save Assessment Report
```python
report_path = INTERIM_DATA_PATH / 'assessment_report.json'
with open(report_path, 'w') as f:
    json.dump(assessment_report, f, indent=2)

print(f"Assessment report saved to: {report_path}")
```

### Cell 21: Generate Text Report
```python
report_lines = []
report_lines.append("=" * 80)
report_lines.append("DATA QUALITY ASSESSMENT REPORT")
report_lines.append("=" * 80)
report_lines.append("")

report_lines.append("DATASET OVERVIEW")
report_lines.append("-" * 80)
report_lines.append(f"Total Records: {assessment_report['dataset_info']['total_records']:,}")
report_lines.append(f"Total Columns: {assessment_report['dataset_info']['total_columns']}")
report_lines.append(f"Memory Usage: {assessment_report['dataset_info']['memory_mb']:.2f} MB")
report_lines.append(f"Date Range: {assessment_report['dataset_info']['date_range']['min']} to {assessment_report['dataset_info']['date_range']['max']}")
report_lines.append("")

report_lines.append("MISSING VALUES")
report_lines.append("-" * 80)
report_lines.append(f"Columns with Missing: {assessment_report['missing_values']['columns_with_missing']}")
for col, info in assessment_report['missing_values']['critical_missing'].items():
    report_lines.append(f"  {col}: {info['count']:,} ({info['percentage']}%)")
report_lines.append("")

report_lines.append("DATA QUALITY ISSUES")
report_lines.append("-" * 80)
report_lines.append(f"Unparseable Dates: {assessment_report['data_quality']['unparseable_dates']:,}")
report_lines.append(f"Future Dates: {assessment_report['data_quality']['future_dates']:,}")
report_lines.append(f"Negative Amounts: {assessment_report['data_quality']['negative_amounts']:,}")
report_lines.append(f"Zero Amounts: {assessment_report['data_quality']['zero_amounts']:,}")
report_lines.append("")

report_lines.append("DUPLICATES")
report_lines.append("-" * 80)
report_lines.append(f"Exact Duplicates: {assessment_report['duplicates']['exact_duplicates']:,}")
report_lines.append(f"Key Column Duplicates: {assessment_report['duplicates']['subset_duplicates']:,}")
report_lines.append("")

report_lines.append("OUTLIERS")
report_lines.append("-" * 80)
report_lines.append(f"IQR Outliers: {assessment_report['outliers']['iqr_outliers']:,} ({assessment_report['outliers']['percentage']}%)")
report_lines.append("")

report_lines.append("=" * 80)

report_text = "\n".join(report_lines)
print(report_text)

text_report_path = REPORTS_PATH / '02_data_quality_report.txt'
with open(text_report_path, 'w') as f:
    f.write(report_text)

print(f"\nText report saved to: {text_report_path}")
```

---

### Markdown Cell
```markdown
## Summary and Next Steps
```

### Cell 22: Issues Priority Summary
```python
print("=== PRIORITY ISSUES FOR CLEANING ===\n")

priority_issues = []

if assessment_report['missing_values']['critical_missing']['date']['percentage'] > 10:
    priority_issues.append("HIGH: Critical missing values in date column")

if assessment_report['data_quality']['unparseable_dates'] > 0:
    priority_issues.append("HIGH: Unparseable date formats need standardization")

if assessment_report['data_quality']['negative_amounts'] > 0:
    priority_issues.append("MEDIUM: Negative amounts need investigation")

if assessment_report['duplicates']['subset_duplicates'] > len(df) * 0.05:
    priority_issues.append("MEDIUM: Significant duplicate records")

if assessment_report['outliers']['percentage'] > 5:
    priority_issues.append("MEDIUM: High percentage of outliers")

if assessment_report['consistency']['case_duplicates'] > 0:
    priority_issues.append("LOW: Category case inconsistency")

if priority_issues:
    for i, issue in enumerate(priority_issues, 1):
        print(f"{i}. {issue}")
else:
    print("No critical issues found")
```

## Expected Outcomes
1. File `assessment_report.json` berisi summary kuantitatif semua quality issues
2. File `02_data_quality_report.txt` berisi laporan readable untuk dokumentasi
3. Identifikasi clear tentang issues yang perlu di-handle di cleaning phase
4. Prioritas cleaning berdasarkan severity issues

## Validation Checklist
- [ ] Semua kolom critical di-assess
- [ ] Missing values pattern teridentifikasi per source
- [ ] Outliers terdeteksi dan dikuantifikasi
- [ ] Duplicate records terhitung
- [ ] Date dan amount quality issues terdokumentasi
- [ ] Assessment report tersimpan
- [ ] Priority issues list dibuat

## Notes
- Tidak ada cleaning yang dilakukan di notebook ini
- Fokus hanya pada identifikasi dan dokumentasi issues
- Assessment report akan menjadi acuan untuk cleaning strategy
