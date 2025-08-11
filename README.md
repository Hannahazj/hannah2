// table.component.ts (inside class)
private formatHeader(key: string): string {
  // "document_name" -> "Document Name"
  return key
    .replace(/_/g, ' ')
    .replace(/\b\w/g, (c) => c.toUpperCase());
}

exportAll() {
  const headers = this.headersArray;
  const rows = this.allDataArray;

  // Use formatted labels for the CSV header row
  let csvContent = headers.map(h => this.formatHeader(h)).join(",") + "\n";

  rows.forEach(row => {
    const rowStr = headers.map(h => {
      const cell = row[h] ?? '';
      if (typeof cell === 'string' && (cell.includes(',') || cell.includes('"'))) {
        return `"${cell.replace(/"/g, '""')}"`;
      }
      return cell;
    }).join(",");
    csvContent += rowStr + "\n";
  });

  const blob = new Blob([csvContent], { type: "text/csv" });
  const url = window.URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = "documents_export.csv";
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  window.URL.revokeObjectURL(url);
}
