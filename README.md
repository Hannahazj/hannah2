exportAll() {
  // 1. Use the visible column names from the table
  const headers = this.headersArray; 
  const rows = this.allDataArray; 

  // 2. Convert to CSV
  let csvContent = headers.join(",") + "\n";
  rows.forEach(row => {
    const rowStr = this.headersArray.map(h => {
      // Find the actual key in the object that matches this header if needed
      const key = Object.keys(row).find(k => 
        k.replace(/_/g, ' ').toLowerCase() === h.toLowerCase()
      ) || h;

      const cell = row[key] !== undefined && row[key] !== null ? row[key] : '';
      if (typeof cell === 'string' && (cell.includes(',') || cell.includes('"'))) {
        return `"${cell.replace(/"/g, '""')}"`;
      }
      return cell;
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
