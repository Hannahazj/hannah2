trackByDoc = (_: number, row: any) => row.file_name ?? row.document_name ?? row.id ?? _;


private applySort() {
  if (!this.userHasSorted || !this.sortColumn) return;

  const col = this.sortColumn;
  const asc = this.sortAsc;
  const type = this.columnTypes[col] || 'string';

  const sorted = [...this._dataArray].sort((a, b) => {
    let diff: number;
    if (type === 'number') {
      diff = Number(a?.[col]) - Number(b?.[col]);
    } else if (type === 'date') {
      const dateA = a?.[col] ? new Date(a[col]).getTime() : 0;
      const dateB = b?.[col] ? new Date(b[col]).getTime() : 0;
      diff = dateA - dateB;
    } else {
      diff = (a?.[col] ?? '').toString().localeCompare((b?.[col] ?? '').toString());
    }
    if (col === 'fiscal_year' || col === 'last_updated') {
      return asc ? -diff : diff;
    }
    return asc ? diff : -diff;
  });

  this._dataArray = sorted;

  // 👉 ensure the viewport shows the newly sorted order from the top
  this.viewport?.scrollToIndex(0, 'auto');
  // give Angular a tick, then force remeasure
  setTimeout(() => this.viewport?.checkViewportSize(), 0);
}


if (!this.userHasSorted || !this.sortColumn) return;
