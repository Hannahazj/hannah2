import { AfterViewInit, Component, ViewChild, ElementRef, Input, ChangeDetectorRef } from '@angular/core';
import { TableComponent } from '../table/table.component';
import { BaseComponent } from '../base/base.component';
import { ApiService } from '../../services/api-service.service';
import { NgClass, NgFor, NgIf } from '@angular/common';
import { DocumentsFilterPipe } from '../pipes/documents-filter.pipe';
import { LoadingComponent } from '../loading/loading.component';
import { NgModel, FormsModule } from "@angular/forms";
import Debounce from "debounce-decorator";
import { ActivatedRoute, Router, NavigationEnd } from '@angular/router';
//import { NodeGraphComponent } from '../node-graph/node-graph.component';
import { RouterModule } from '@angular/router';
import { NgbDropdownModule } from '@ng-bootstrap/ng-bootstrap';
import { NodeGraphComponent } from '../node-graph-chart/node-graph.component';
import { Subscription } from 'rxjs';
import { filter, take } from 'rxjs/operators';

/**
 * Document List Component
 * Parent component used to display a table of documents available
 */
@Component({
  selector: 'app-documents-list',
  standalone: true,
  imports: [NgbDropdownModule, RouterModule, BaseComponent, TableComponent, NgIf, NgFor, NgClass, FormsModule, DocumentsFilterPipe, LoadingComponent, NodeGraphComponent],
  templateUrl: './documents-list.component.html',
  styleUrl: './documents-list.component.scss'
})
export class DocumentsListComponent extends BaseComponent implements AfterViewInit {
@ViewChild('searchInput') searchInput!: ElementRef<HTMLInputElement>;

  /**
   * String used to track the filter value
   */
  filter: string = ''; 

  /**
   * Array of header names to be used
   * REMOVING DOCUMENT TYPE AND FOCUS AREA FOR THE TIME BEING
   */
  //headersArray = ['document_name', 'document_type', 'file_name', 'fiscal_year', 'focus_area' , 'link'];
  headersArray = ['document_name',  'file_name', 'fiscal_year',  'link','last_updated'];
   @Input() allDataArray = [];
  /**
   * Data array to be used for the documents table
   */
  dataArray = [
  ];
  /**
   * Data object 
   */
  data = {};

  networkData = {};

  /**
   * List of entitites 
   */
  entitiesList = [];

  // we don't use SelecedEntity anymore we use SelctedEntitiesList for filtering, some parts of the code will need to be cleaned up
  selectedEntity = '';
    /**
   * Selected Entities List 
   */
    selectedEntitiesList = [];

  /* Filtering tabs which are used to filter the documents */
  filteringTabs = [];
  selectedFilteringTabs = '';

  /**
   * loading flag
   */
  loading = false;

  childRoute = undefined;

  routeSubscription!: Subscription;

  /**
   * Constructor function
   * @param apiService 
   */
  constructor (
    private route: ActivatedRoute,
    private router: Router,
    private apiService: ApiService,
    private cdr: ChangeDetectorRef
  ){
      super();
    }

  // helper function for the tabs due to endpoint being missing
  private useStaticTabsFallback(reason='endpoint is missing from fastapi'): void{
    console.warn(`Using static tabs fallback because: ${reason}`);
  
  this.filteringTabs = ["ORGANIZATION", 
    "EVENT", 
    "PERSON", 
    "FINANCIAL CONCEPTS",
     "US LAWS",
     "AUDITING PRACTICES",
      "GEOGRAPHICAL LOCATION" ];

  }


  /**
   * NgOnInit function
   * Runs when the component is loaded
   */
  ngOnInit(){   
    // calling the sorting tab endpoint to get the filtering tabs for the dropdown filter
    this.apiService.postGetExploreSortingTabs().pipe(take(1)).subscribe({
      next: (response) => {
        // if there is an error then use the static tabs fallback
        if ((response as any)?.status == 404){
          this.useStaticTabsFallback('404 from api')
          return;
        }

        // comes back with tab_id and option we only need option
        const rawTabs = Array.isArray(response.explore_tabs) ? response.explore_tabs : [];
        this.filteringTabs = rawTabs.map((t: any) => t.tab_option);
        console.log('Filtering Tabs:', this.filteringTabs);
        console.log('Selected Filtering Tab:', this.selectedFilteringTabs);
      },
      error: (error) => console.error('Filtering Tabs Error:', error),
    }); 

    console.log(this.router);
    const temp = this.router.url.split('/');
    console.log(temp);
    this.childRoute = temp[temp.length -1];

    this.routeSubscription = this.router.events
    .pipe(filter((event) => event instanceof NavigationEnd))
    .subscribe((event: NavigationEnd) => {

      const temp = event.url.split('/');
      console.log(temp);
      this.childRoute = temp[temp.length -1];
      console.log(this.childRoute);

  });

    this.loading = true;
    this.getDocumentInfo();
  }

  ngAfterViewInit(){
    
    setTimeout(() => this.searchInput?.nativeElement?.focus(), 0);

  }

  ngOnDestroy(): void {
    if (this.routeSubscription) {
      this.routeSubscription.unsubscribe();
    }
  }

  /**
   * Gets the column headers from the data object
   * @returns 
   */
  getColumnHeaders(){
    return Object.keys(this.data[0]);
  }


  updateFilter(){
    console.log('Update filter', this.filter);
    this.selectedFilteringTabs = ''; // update the selected filtering tab to empty since theere is no mention of fitler tab for the taggingtabs endpoint
    this.getDocumentInfo();
  }

    // this function is no longer used anymore
    // sorts documents 
    onTopicSelected(entity: string) {
      if (!entity) {
        this.selectedEntitiesList = [];
      } else {
        this.selectedEntitiesList = [entity];
      }
      this.getDocumentInfo();
    }

    // function for filtering based off a tab selected
    onFilterTabSelected(tab: string) {
      this.selectedFilteringTabs= tab;
      this.selectedEntitiesList = []; // update the selected entities tag to be empty since there is no mention of entitieslist for the taggingtabs endpoint
      this.getDocumentInfo();
    }

 /**
   * Function for export all
   */
 exportAll() {
  // 1. Get headers and data
  const headers = this.headersArray;
  const rows = this.allDataArray; // or this.dataArray

  // 2. Convert to CSV
  let csvContent = headers.join(",") + "\n";
  rows.forEach(row => {
    const rowStr = headers.map(h => {
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

    

  /**
   * Update filter value
   */
/*   updateFilter(){
    //console.log('Update filter', this.filter);
    this.getDocumentInfo();
  } */

// the post-get-docs cosmos is not functioanlly corretly on the backend it is not pulling all the documents
// we are using a variable called useOld to flip from the old endpoint to the new one

  /**
   * Get the documents available from the server
   */
  @Debounce(500)
  getDocumentInfo() {
    this.loading = false; // might change this to loading = true since it is slow
    const txt = this.filter.trim();
    const tab = this.selectedFilteringTabs.trim();


    if (tab) {
      // Tagging-tabs branch: only has docs & entities so we need to have logic that handles network
      this.apiService
        .postGetTaggingTabs(tab)
        .pipe(take(1))
        .subscribe({
          next: res => {

            this.applyResultToTable({
              documents: res.documents,
              entities:  res.entities
            });
            this.loading = false;
          },
          error: err => {
            console.error('Filtering Tabs Error:', err);
            this.loading = false;
          }
        });
    } else {
      const docs$ = this.apiService.postGetDocsCosmos(txt, this.selectedEntitiesList, 800);

      docs$
        .subscribe({
          next: result => {
            this.applyResultToTable(result);
            this.loading = false;
          },
          error: err => {
            console.error('Documents fetch error', err);
            this.loading = false;
          }
        });
    }
  }


  /**  helper function to apply the result to the table */
  private applyResultToTable(result: {
    documents: any[];
    entities: string[];
    d3_graph?: any;
  }) {
    this.allDataArray  = result.documents;
    //  use tab entities if a tab is selected (category) otherwise use cosmos entities if no tab is selected
    const tab = this.selectedFilteringTabs.trim();
    this.entitiesList  = result.entities
    this.dataArray     = [...result.documents].sort((a, b) =>
      a.document_name.localeCompare(b.document_name)
    );
    this.loading = false

    if (result.d3_graph) {
      this.networkData = result.d3_graph;
      this.cdr.markForCheck();
    }
  }



  /**
   * Submit entity value
   * @param entity 
   */
  submitEntity(entity){

    this.selectedFilteringTabs = ''; // update the selected filtering tab to empty since there is no mention of fitler tab for the taggingtabs endpoint
    // Check if entity already selected
    let alreadySelected = this.selectedEntitiesList.indexOf(entity)

    if (alreadySelected == -1){
      // Add to array
      this.selectedEntitiesList.push(entity);
      this.getDocumentInfo();
    } else {
      this.selectedEntitiesList.splice(alreadySelected, 1)
      this.getDocumentInfo();
    }

  }
  

}
