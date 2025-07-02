# Data Extraction from Unstructured PDFs

*Author:* Ashish  
*Last Updated:* 01 May, 2025  
*Reading time:* 9 min read

Data Extraction is the process of extracting data from various sources such as CSV files, web, PDF, etc. Although in some files, data can be extracted easily (e.g., CSV), files like unstructured PDFs require additional steps to extract data using Python. In unstructured PDFs, data is stored randomly, so we need to convert it into a structured form before analysis.

---

## Table of Contents

- [PyMuPDF](#pymupdf)  
- [Code](#code)  
- [Data Cleaning and Data Processing](#data-cleaning-and-data-processing)  
- [How to Extract Data from Unstructured PDF Files with Python?](#how-to-extract-data-from-unstructured-pdf-files-with-python)  
- [How to extract information from pdf python?](#how-to-extract-information-from-pdf-python)  
- [Conclusion](#conclusion)  
- [Frequently Asked Questions](#frequently-asked-questions)  

---

## PyMuPDF

We’ll use the [PyMuPDF](https://pypi.org/project/PyMuPDF/) (aka `fitz`) library to:

- Open and navigate PDF pages  
- Read all “words” (with their bounding‐box coordinates)  
- Leverage PDF annotations (bounding boxes) to pinpoint regions of interest  

> **Terminology**  
> - **Word**: A sequence of characters without spaces (e.g. “ash”, “23”, “2,3”).  
> - **Annot**: A PDF annotation (note, image, or bounding‐box) associated with a location on a page. In this article, “annot” and “bounding box” are used interchangeably.

_Image examples of a PDF with red bounding boxes omitted._

---

## Code

### 1. Extract raw word‐tuples from the first page

```python
import fitz
import pandas as pd

# Open PDF and grab page 1
doc = fitz.open('Mansfield--70-21009048 - ConvertToExcel.pdf')
page1 = doc[0]

# Get all words: each entry is a tuple
words = page1.get_text("words")
````

### 2. Extract text from the first bounding box

```python
first_annots = []

# Get the rectangle of the first annotation
rec = page1.first_annot.rect

# Filter words whose bbox falls inside rec
mywords = [w for w in words if fitz.Rect(w[:4]) in rec]

# Convert tuples to a readable string
ann = make_text(mywords)
first_annots.append(ann)
```

### 3. Helper to assemble words in reading order

```python
def make_text(words):
    # Group words by their y-coordinate (rounded)
    line_dict = {}
    words.sort(key=lambda w: w[0])  # sort by x-coord

    for w in words:
        y1 = round(w[3], 1)         # y2 coordinate
        word = w[4]
        line_dict.setdefault(y1, []).append(word)

    # Reassemble lines top to bottom, left to right
    lines = sorted(line_dict.items())
    return "\n".join(" ".join(line) for (_, line) in lines)
```

### 4. Extract from every page and every annotation

```python
all_annots = []

for pageno in range(len(doc)):
    page = doc[pageno]
    words = page.get_text("words")

    for annot in page.annots() or []:
        rec     = annot.rect
        mywords = [w for w in words if fitz.Rect(w[:4]) in rec]
        ann     = make_text(mywords)
        all_annots.append(ann)
```

---

## Data Cleaning and Data Processing

### 1. Split into key/value pairs

```python
cont = [text.split('\n', 1) for text in all_annots]
```

### 2. Remove unwanted symbols

```python
liss = []
for pair in cont:
    cleaned = [s.replace('*','').replace('#','').replace(':','').strip() 
               for s in pair]
    liss.append(cleaned)
```

### 3. Normalize numeric values

```python
keys   = [p[0] for p in liss]
values = [p[1] for p in liss]

for i, v in enumerate(values):
    if all(ch.isdigit() or ch==' ' for ch in v):
        values[i] = v.replace(' ', '')
```

### 4. Build the final record

```python
report = dict(zip(keys, values))
# e.g., strip spaces from a VIN
report['VEHICLE IDENTIFICATION'] = report['VEHICLE IDENTIFICATION'].replace(' ', '')

# Further split fields like LOCALITY, MANNER OF CRASH, CRASH SEVERITY into sub-values
# … (see full example code)
```

### 5. Convert to DataFrame & export

```python
df = pd.DataFrame([report])
df.to_csv('final.csv', index=False)
```

---

## How to Extract Data from Unstructured PDF Files with Python?

A simpler, text-only approach using **PyPDF2** + **NLTK**:

1. **Install**

   ```bash
   pip install PyPDF2 nltk
   ```
2. **Extract all text**

   ```python
   import PyPDF2

   def extract_text_from_pdf(pdf_path):
       with open(pdf_path, 'rb') as f:
           reader = PyPDF2.PdfFileReader(f)
           text = ""
           for i in range(reader.numPages):
               text += reader.getPage(i).extract_text()
       return text
   ```
3. **Preprocess**

   ```python
   from nltk.tokenize import word_tokenize
   from nltk.corpus import stopwords

   def preprocess_text(text):
       tokens     = word_tokenize(text.lower())
       stop_words = set(stopwords.words('english'))
       tokens     = [t for t in tokens if t.isalnum() and t not in stop_words]
       return " ".join(tokens)
   ```
4. **Use**

   ```python
   raw_text      = extract_text_from_pdf('your_file.pdf')
   clean_text    = preprocess_text(raw_text)
   # … downstream NLP or regex-based extraction …
   ```

---

## How to extract information from pdf python?

Popular libraries and their strengths:

* **PyPDF2**: basic text & metadata extraction
* **PDFMiner**: complex layouts, font-level control
* **PyMuPDF (fitz)**: fast text + image + annotation handling
* **PDFQuery**: CSS-style selectors for targeted data
* **IronPDF**: commercial, simplified forms & tables

### Example: PyPDF2 snippet

```python
from PyPDF2 import PdfReader

reader = PdfReader('example.pdf')
print(len(reader.pages))           # number of pages
text = reader.pages[0].extract_text()
print(text)
```

---

## Conclusion

Extracting data from unstructured PDFs can be challenging. For form- or table-like PDFs, tools such as **PyPDF2**, **Camelot**, or **tabula-py** work well. For truly unstructured layouts, **PyMuPDF**’s bounding-box approach combined with custom cleaning yields reliable, structured output. Once in a DataFrame, you can export to CSV/Excel and proceed with analysis.

---

## Frequently Asked Questions

**Q1. How do I extract specific data from a PDF?**
Use bounding-box methods in PyMuPDF or targeted selectors in PDFQuery to isolate only the regions containing your data.

**Q2. How do I extract form data from a PDF?**
Libraries like PyPDF2 and pdfplumber can read PDF form fields and export their values programmatically.

**Q3. Can I extract data from PDF to Excel?**
Yes—extract into a pandas DataFrame, then use `df.to_csv()` or `df.to_excel()`.

**Q4. How to automatically extract data from PDF?**
Write a Python script combining extraction + cleanup, then schedule it via cron (Linux) or Task Scheduler (Windows).

