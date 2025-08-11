/**
 * Function for export all
 */
exportAll() {
  // 1. Get headers (transform to match table display)
  const headers = this.headersArray.map(h =>
    h.replace(/_/g, ' ')                      // underscores → spaces
     .replace(/\b\w/g, c => c.toUpperCase())   // capitalise each word
  );

  const rows = this.allDataArray; // or this.dataArray

  // 2. Convert to CSV
  let csvContent = '\uFEFF' + headers.join(",") + "\n"; // BOM for Excel UTF-8 compatibility

  rows.forEach(row => {
    const rowStr = this.headersArray.map(h => {
      let cell = row[h];

      // Handle null/undefined
      if (cell === undefined || cell === null) cell = '';

      // Handle objects/arrays
      if (typeof cell === 'object') {
        cell = JSON.stringify(cell);
      }

      const s = String(cell);
      // Quote if needed
      if (/[",\n\r]/.test(s)) {
        return `"${s.replace(/"/g, '""')}"`;
      }
      return s;
    }).join(",");

    csvContent += rowStr + "\n";
  });

  // 3. Trigger download
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
