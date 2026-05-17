# 01_data_gathering.ipynb - Prompt Instructions

## Objektif
Mengumpulkan dan menggabungkan semua dataset raw dari berbagai sumber menjadi satu dataset gabungan untuk analisis impulsive buying behavior.

## Input Data
```
data/raw/daily_household_transaction/
  - daily_household_transaction_clean
  - daily_household_transaction_raw_public
  - Daily_household_transactioncsv
data/raw/Indonesian_Ecommerce_sales/
  - Indonesian_Ecommerce_sales_clean
  - Indonesian_Ecommerce_sales_raw_public
data/raw/all_months_clean.csv
data/raw/personal_finance/Personal_Finance_Dataset.csv
```

## Output Data
```
data/interim/combined_raw_transactions.csv
```

## Implementation Code

### Cell 1: Setup Environment
```python
import pandas as pd
import numpy as np
from pathlib import Path
import sys

sys.path.append('../src')
from fingo_ds1.config import RAW_DATA_PATH, INTERIM_DATA_PATH

pd.set_option('display.max_columns', None)
pd.set_option('display.max_rows', 100)
```

### Cell 2: Load Household Transaction Data
```python
def load_household_data():
    path = RAW_DATA_PATH / 'daily_household_transaction'
    
    dfs = []
    for file in ['daily_household_transaction_clean', 
                 'daily_household_transaction_raw_public', 
                 'Daily_household_transactioncsv']:
        try:
            df = pd.read_csv(path / file)
            dfs.append(df)
        except Exception as e:
            print(f"Error loading {file}: {e}")
    
    return pd.concat(dfs, ignore_index=True) if dfs else pd.DataFrame()

df_household = load_household_data()
print(f"Household data: {len(df_household):,} records")
df_household.head()
```

### Cell 3: Load Ecommerce Data
```python
def load_ecommerce_data():
    path = RAW_DATA_PATH / 'Indonesian_Ecommerce_sales'
    
    dfs = []
    for file in ['Indonesian_Ecommerce_sales_clean', 
                 'Indonesian_Ecommerce_sales_raw_public']:
        try:
            df = pd.read_csv(path / file)
            dfs.append(df)
        except Exception as e:
            print(f"Error loading {file}: {e}")
    
    return pd.concat(dfs, ignore_index=True) if dfs else pd.DataFrame()

df_ecommerce = load_ecommerce_data()
print(f"Ecommerce data: {len(df_ecommerce):,} records")
df_ecommerce.head()
```

### Cell 4: Load Monthly and Personal Finance Data
```python
df_monthly = pd.read_csv(RAW_DATA_PATH / 'all_months_clean.csv')
print(f"Monthly data: {len(df_monthly):,} records")

df_personal = pd.read_csv(RAW_DATA_PATH / 'personal_finance/Personal_Finance_Dataset.csv')
print(f"Personal finance data: {len(df_personal):,} records")

df_monthly.head()
```

### Cell 5: Inspect Column Structure
```python
datasets = {
    'household': df_household,
    'ecommerce': df_ecommerce,
    'monthly': df_monthly,
    'personal': df_personal
}

for name, df in datasets.items():
    print(f"\n{name.upper()} columns:")
    print(df.columns.tolist())
    print(f"Shape: {df.shape}")
```

### Cell 6: Standardize Column Names
```python
def standardize_columns(df, source_name):
    df = df.copy()
    df.columns = df.columns.str.lower().str.strip().str.replace(' ', '_').str.replace('-', '_')
    df['data_source'] = source_name
    return df

df_household = standardize_columns(df_household, 'household_transaction')
df_ecommerce = standardize_columns(df_ecommerce, 'ecommerce_sales')
df_monthly = standardize_columns(df_monthly, 'monthly_clean')
df_personal = standardize_columns(df_personal, 'personal_finance')

print("Columns standardized")
```

### Cell 7: Identify Common Schema
```python
all_columns = set()
for df in [df_household, df_ecommerce, df_monthly, df_personal]:
    all_columns.update(df.columns)

print(f"Total unique columns: {len(all_columns)}")
print(f"\nAll columns:\n{sorted(all_columns)}")

required_cols = ['date', 'amount', 'category', 'description']
print(f"\nRequired columns for alignment: {required_cols}")
```

### Cell 8: Map and Align Columns
```python
def map_columns(df, source):
    df = df.copy()
    
    column_mapping = {
        'household_transaction': {
            'transaction_date': 'date',
            'total_amount': 'amount',
            'product_category': 'category',
            'product_name': 'description'
        },
        'ecommerce_sales': {
            'order_date': 'date',
            'price': 'amount',
            'product_category': 'category',
            'product_name': 'description'
        },
        'monthly_clean': {
            'date': 'date',
            'amount': 'amount',
            'category': 'category',
            'description': 'description'
        },
        'personal_finance': {
            'date': 'date',
            'amount': 'amount',
            'category': 'category',
            'description': 'description'
        }
    }
    
    mapping = column_mapping.get(source, {})
    if mapping:
        df = df.rename(columns=mapping)
    
    return df

df_household = map_columns(df_household, 'household_transaction')
df_ecommerce = map_columns(df_ecommerce, 'ecommerce_sales')
df_monthly = map_columns(df_monthly, 'monthly_clean')
df_personal = map_columns(df_personal, 'personal_finance')

print("Columns mapped")
```

### Cell 9: Create Unified Schema
```python
def align_schema(df, required_cols):
    df = df.copy()
    
    for col in required_cols:
        if col not in df.columns:
            df[col] = np.nan
    
    existing_cols = [col for col in required_cols if col in df.columns]
    extra_cols = [col for col in df.columns if col not in required_cols and col != 'data_source']
    
    return df[existing_cols + ['data_source'] + extra_cols]

df_household_aligned = align_schema(df_household, required_cols)
df_ecommerce_aligned = align_schema(df_ecommerce, required_cols)
df_monthly_aligned = align_schema(df_monthly, required_cols)
df_personal_aligned = align_schema(df_personal, required_cols)

print("Schema aligned")
print(f"\nHousehold: {df_household_aligned.shape}")
print(f"Ecommerce: {df_ecommerce_aligned.shape}")
print(f"Monthly: {df_monthly_aligned.shape}")
print(f"Personal: {df_personal_aligned.shape}")
```

### Cell 10: Combine All Datasets
```python
df_combined = pd.concat([
    df_household_aligned,
    df_ecommerce_aligned,
    df_monthly_aligned,
    df_personal_aligned
], ignore_index=True)

print(f"Combined dataset shape: {df_combined.shape}")
print(f"\nRecords per source:")
print(df_combined['data_source'].value_counts())
```

### Cell 11: Basic Validation
```python
print("=== DATA VALIDATION ===\n")

print(f"Total records: {len(df_combined):,}")
print(f"Total columns: {len(df_combined.columns)}")

print("\nMissing values per column:")
missing = df_combined.isnull().sum()
missing_pct = (missing / len(df_combined) * 100).round(2)
missing_df = pd.DataFrame({
    'missing_count': missing,
    'missing_pct': missing_pct
}).sort_values('missing_pct', ascending=False)
print(missing_df[missing_df['missing_count'] > 0])

print("\nData types:")
print(df_combined.dtypes)
```

### Cell 12: Date Range Validation
```python
df_combined['date'] = pd.to_datetime(df_combined['date'], errors='coerce')

print("Date range validation:")
print(f"Min date: {df_combined['date'].min()}")
print(f"Max date: {df_combined['date'].max()}")
print(f"Date range: {(df_combined['date'].max() - df_combined['date'].min()).days} days")

print("\nRecords with invalid dates:")
print(f"{df_combined['date'].isnull().sum():,} records")
```

### Cell 13: Amount Validation
```python
df_combined['amount'] = pd.to_numeric(df_combined['amount'], errors='coerce')

print("Amount statistics:")
print(df_combined['amount'].describe())

print("\nRecords with invalid amounts:")
print(f"Null: {df_combined['amount'].isnull().sum():,}")
print(f"Negative: {(df_combined['amount'] < 0).sum():,}")
print(f"Zero: {(df_combined['amount'] == 0).sum():,}")
```

### Cell 14: Preview Final Dataset
```python
print("=== FINAL DATASET PREVIEW ===\n")
print(df_combined.info())
print("\nFirst 10 records:")
df_combined.head(10)
```

### Cell 15: Save Combined Dataset
```python
output_path = INTERIM_DATA_PATH / 'combined_raw_transactions.csv'
df_combined.to_csv(output_path, index=False)

print(f"Dataset saved to: {output_path}")
print(f"File size: {output_path.stat().st_size / 1024 / 1024:.2f} MB")
```

## Expected Outcomes
1. File `combined_raw_transactions.csv` berisi minimal 4 sumber data
2. Kolom wajib: `date`, `amount`, `category`, `description`, `data_source`
3. Missing values < 30% untuk kolom critical
4. Date range masuk akal (2020-2024)
5. Amount values positif dan masuk akal

## Validation Checklist
- [ ] Semua 4 dataset berhasil di-load
- [ ] Tidak ada duplikasi record dalam satu sumber
- [ ] Column mapping berhasil untuk semua dataset
- [ ] Date format konsisten
- [ ] Amount format konsisten (numeric)
- [ ] Data source teridentifikasi jelas
- [ ] File output tersimpan di `data/interim/`

## Notes
- Jika ada error saat loading, skip dataset tersebut dan lanjutkan dengan yang lain
- Kolom tambahan (selain required) tetap dipertahankan untuk analisis lanjutan
- Tidak dilakukan cleaning di tahap ini, hanya gathering dan standardisasi struktur
