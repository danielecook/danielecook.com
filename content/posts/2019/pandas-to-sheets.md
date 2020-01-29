---
title: From Pandas to Google Sheets
author: Daniel Cook
date: 2019-10-25T01:15:53Z
tags:
  - python
  - google-sheets
---

I wrote the following snippet to post data files to google sheets. In order to get this to work you will need to [authorize google sheets access](https://gspread.readthedocs.io/en/latest/oauth2.html).

Then you can set the content of any google sheets worksheet to the data from a pandas dataframe by using the `pandas_to_sheets` function.

```python
#!/usr/bin/env python
import gspread
from oauth2client.service_account import ServiceAccountCredentials

def iter_pd(df):
    for val in list(df.columns):
        yield val
    for row in df.values:
        for val in list(row):
            if pd.isna(val):
                yield ""
            else:
                yield val

def pandas_to_sheets(pandas_df, sheet, clear = True):
    # Updates all values in a workbook to match a pandas dataframe
    if clear:
        sheet.clear()
    (row, col) = pandas_df.shape
    cells = sheet.range("A1:{}".format(gspread.utils.rowcol_to_a1(row + 1, col)))
    for cell, val in zip(cells, iter_pd(df)):
        cell.value = val
    sheet.update_cells(cells)

scope = ['https://spreadsheets.google.com/feeds',
         'https://www.googleapis.com/auth/drive']

credentials = ServiceAccountCredentials.from_json_keyfile_name('service.json', scope)

gc = gspread.authorize(credentials)

workbook = gc.open_by_key("<workbook id>")
sheet = workbook.worksheet("worksheet_name")

df = pd.read_csv("input_data.tsv")
pandas_to_sheets(df, workbook.worksheet("worksheet"))
```