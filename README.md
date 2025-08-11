exportAll() {
  // Keys coming from the table, in the order you want
  const keys: string[] = this.headersArray;

  // Exact labels you want in the CSV header row
  const keyToLabel: Record<string, string> = {
    document_name: 'Document Name',
    file_name: 'File Name',
    'file-name': 'File Name',     // just in case
    fiscal_year: 'Fiscal Year',
    'fiscal-year': 'Fiscal Year', // just in case
    link: 'link',
    last_updated: 'Last Updated',
    'last-updated': 'Last Updated'// just in case
  };

  const headerLabels = keys.map(k => keyToLabel[k] ?? k);

  const rows = this.allDataArray ?? [];

  let csvContent = headerLabels.join(",") + "\n";

  rows.forEach((row: any) => {
    const line = keys.map(k => {
      const cell = row?.[k] ?? '';
      const s = String(cell);
      // Quote if comma, quote, or newline; escape quotes
      return /[",\n\r]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
    }).join(",");
    csvContent += line + "\n";
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
