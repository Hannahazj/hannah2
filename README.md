exportAll() {
  // 1. Format headers: replace underscores with spaces & capitalize each word
  const headers = this.headersArray.map(h =>
    h
      .split('_')
      .map(word => word.charAt(0).toUpperCase() + word.slice(1))
      .join(' ')
  );

  const rows = this.allDataArray; // or this.dataArray

  // 2. Convert to CSV
  let csvContent = headers.join(",") + "\n";
  rows.forEach(row => {
    const rowStr = this.headersArray.map(h => {
      // Quote strings with commas or quotes
      const cell = row[h] !== undefined && row[h] !== null ? row[h] : '';
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
