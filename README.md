exportAll() {
  // Raw keys used to pull data
  const dataKeys = this.headersArray;

  // Pretty display headers for CSV
  const displayHeaders = dataKeys.map(h =>
    h.replace(/_/g, ' ')                      // underscores → spaces
     .replace(/\b\w/g, c => c.toUpperCase())   // capitalize each word
  );

  const rows = this.allDataArray; // or this.dataArray

  // CSV header row
  let csvContent = '\uFEFF' + displayHeaders.join(",") + "\n";

  // CSV data rows
  rows.forEach(row => {
    const rowStr = dataKeys.map(key => {
      let cell = row[key];

      if (cell === undefined || cell === null) cell = '';

      if (typeof cell === 'object') {
        cell = JSON.stringify(cell);
      }

      const s = String(cell);
      if (/[",\n\r]/.test(s)) {
        return `"${s.replace(/"/g, '""')}"`;
      }
      return s;
    }).join(",");

    csvContent += rowStr + "\n";
  });

  // Download file
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
