exportAll() {
  const headers = this.headersArray;            // ['document_name','file_name','fiscal_year','link','last_updated']
  const rows = this.allDataArray;               // or this.dataArray

  // 1) Build human-readable header labels (keep 'link' lowercase)
  const headerLabels = headers.map(h =>
    h === 'link'
      ? 'link'
      : h
          .replace(/[_-]/g, ' ')                // underscores/hyphens -> spaces
          .replace(/\b\w/g, m => m.toUpperCase()) // Title Case
  );

  // 2) Start CSV with the pretty headers
  let csvContent = headerLabels.join(",") + "\n";

  // 3) Append rows using the original keys
  rows.forEach(row => {
    const rowStr = headers.map(h => {
      const cell = row[h] ?? '';
      // Quote if it contains comma, quote, or newline; escape quotes
      if (typeof cell === 'string' && /[",\n\r]/.test(cell)) {
        return `"${cell.replace(/"/g, '""')}"`;
      }
      return cell;
    }).join(",");
    csvContent += rowStr + "\n";
  });

  // 4) Download
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
