exportAll() {
  // Raw keys (used for pulling data)
  const dataKeys = this.headersArray;

  // Convert raw keys to display format: underscores → spaces, capitalize each word
  const displayHeaders = dataKeys.map(h =>
    h
      .replace(/_/g, ' ')                      // underscores → spaces
      .replace(/\b\w/g, c => c.toUpperCase())   // capitalize each word
  );

  const rows = this.allDataArray; // export all rows, not just visible

  // Start CSV with formatted header row
  let csvContent = '\uFEFF' + displayHeaders.join(",") + "\n";

  // Add each row
  rows.forEach(row => {
    const rowStr = dataKeys.map(key => {
      let cell = row[key] ?? '';

      // Convert objects/arrays to JSON
      if (typeof cell === 'object') {
        cell = JSON.stringify(cell);
      }

      const s = String(cell);
      // Escape if needed
      if (/[",\n\r]/.test(s)) {
        return `"${s.replace(/"/g, '""')}"`;
      }
      return s;
    }).join(",");

    csvContent += rowStr + "\n";
  });

  // Download CSV
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
