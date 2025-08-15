# pip install pypdf
from pypdf import PdfReader

def read_with_pypdf(path: str, password: str | None = None):
    reader = PdfReader(path)
    if reader.is_encrypted and password:
        # If the PDF is encrypted, try to decrypt
        reader.decrypt(password)

    print(f"Pages: {len(reader.pages)}")
    if reader.metadata:
        print("Metadata:")
        for k, v in reader.metadata.items():
            print(f"  {k}: {v}")

    page_texts = []
    for i, page in enumerate(reader.pages, start=1):
        text = page.extract_text() or ""
        page_texts.append(text)
        print(f"\n--- Page {i} ---\n{text[:500]}")  # preview first 500 chars

    return page_texts

if __name__ == "__main__":
    read_with_pypdf("your_file.pdf", password=None)


# pip install pdfplumber
import csv
import pdfplumber

def read_with_pdfplumber(path: str, password: str | None = None, export_tables_csv: str | None = None):
    page_texts = []
    all_tables = []  # list of (page_number, table_rows)

    with pdfplumber.open(path, password=password) as pdf:
        print(f"Pages: {len(pdf.pages)}")

        for i, page in enumerate(pdf.pages, start=1):
            # Text
            text = page.extract_text() or ""
            page_texts.append(text)
            print(f"\n--- Page {i} ---\n{text[:500]}")  # preview

            # Tables (works best on text-based, ruled tables)
            tables = page.extract_tables()
            if tables:
                for t in tables:
                    all_tables.append((i, t))

    # Optional: export all detected tables to one CSV (with page separators)
    if export_tables_csv and all_tables:
        with open(export_tables_csv, "w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            for page_no, rows in all_tables:
                writer.writerow([f"__PAGE_{page_no}__"])
                writer.writerows(rows)
        print(f"\nTables exported to: {export_tables_csv}")

    return page_texts, all_tables

if __name__ == "__main__":
    read_with_pdfplumber("your_file.pdf", password=None, export_tables_csv="tables.csv")
