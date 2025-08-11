private tabNameMap: Record<string, string> = {
  "GEOGRAPHICAL LOCATION": "GEO",
  "US LAWS": "US GOVERNMENT LAWS"
};


private useStaticTabsFallback(reason='endpoint is missing from fastapi'): void {
  console.warn(`Using static tabs fallback because: ${reason}`);

  const rawTabs = [
    "ORGANIZATION", 
    "EVENT", 
    "PERSON", 
    "FINANCIAL CONCEPTS",
    "US LAWS",
    "AUDITING PRACTICES",
    "GEOGRAPHICAL LOCATION"
  ];

  // Apply mapping to display names
  this.filteringTabs = rawTabs.map(tab => this.tabNameMap[tab] || tab);
}


this.filteringTabs = rawTabs.map((t: any) => {
  const tabName = t.tab_option;
  return this.tabNameMap[tabName] || tabName;
});


onFilterTabSelected(tab: string) {
  const mappedTab = this.tabNameMap[tab] || tab;
  this.selectedFilteringTabs = mappedTab;
  this.selectedEntitiesList = [];
  this.getDocumentInfo();
}
