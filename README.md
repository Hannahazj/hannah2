// table.component.ts (inside the class)
resetSort(): void {
  this.userHasSorted = false;
  this.sortColumn = null;
  this.sortAsc = true;
  this.alphaIndex = [];
  this.letterToIndex = {};
}

@ViewChild(TableComponent) tableCmp!: TableComponent;


updateFilter(){
  this.tableCmp?.resetSort();         // ← reset on search text change
  this.selectedFilteringTabs = '';
  this.getDocumentInfo();
}

onFilterTabSelected(tab: string) {
  this.tableCmp?.resetSort();         // ← reset on topic change
  const mappedTab = this.tabNameMap[tab] || tab;
  this.selectedFilteringTabs = mappedTab;
  this.selectedEntitiesList = [];
  this.getDocumentInfo();
}

submitEntity(entity: string){
  this.tableCmp?.resetSort();         // ← reset on entity toggle
  this.selectedFilteringTabs = '';
  const alreadySelected = this.selectedEntitiesList.indexOf(entity);
  if (alreadySelected === -1) this.selectedEntitiesList.push(entity);
  else this.selectedEntitiesList.splice(alreadySelected, 1);
  this.getDocumentInfo();
}


if (this.childRoute === 'network') {
  this.tableCmp?.resetSort();   // hide "Sorted by..." & A–Z when leaving the table
}
