  exportAll() {
    // 1) Source headers (keys) and data
    const headers = this.headersArray as string[];
    const rows = this.allDataArray; // or this.dataArray

    // 2) Build display headers exactly as table shows (via the same pipe)
    const displayHeaders = headers.map(h => this.headerPipe.transform(h));

    // 3) Convert to CSV
    // Optional BOM for better Excel compatibility
    let csvContent = '\uFEFF' + displayHeaders.join(",") + "\n";

    rows.forEach(row => {
      const rowStr = headers.map(h => {
        let cell = row[h];

        // Normalize undefined/null
        if (cell === undefined || cell === null) cell = '';

        // Handle objects/arrays safely
        if (typeof cell === 'object') {
          cell = JSON.stringify(cell);
        }

        const s = String(cell);
        // Quote if needed (commas, quotes, newlines)
        if (/[",\n\r]/.test(s)) {
          return `"${s.replace(/"/g, '""')}"`;
        }
        return s;
      }).join(",");

      csvContent += rowStr + "\n";
    });

    // 4) Trigger download
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
}
