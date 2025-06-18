tablle.components. html "<!-- <div class="form-group">
  <input
    type="search"
    class="form-control"
    [(ngModel)]="filter"
    placeholder="Filter results"
    tabindex="0"
    [attr.aria-label]="'Filter documents- current filter value is: ' + filter"
  />
</div> -->

<div class="form-group export-all-container">
  <button
    class="export-all-btn"
    (click)="exportAll()"
    tabindex="0"
    aria-label="Export all documents"
  >
    Export All
  </button>
</div>


<div *ngIf="sortColumn" class="sort-indicator">
  Sorted by <b>{{ sortColumn | removeUnderscoreUppercase }}</b>
  <span *ngIf="sortAsc">({{ getSortLabel(sortColumn, true) }})</span>
  <span *ngIf="!sortAsc">({{ getSortLabel(sortColumn, false) }})</span>
</div>


<cdk-virtual-scroll-viewport
  #verticalViewport
  itemSize="100"
  minBufferPx="200"
  maxBufferPx="600"
  class="viewport"
>
<table class="table">
  <thead>
    <tr>
      <th
        scope="col"
        *ngFor="let col of headersArray"
        [ngClass]="{ selectedColumn: sortColumn === col }"
      >
        <div class="header-cell">
          {{ col | removeUnderscoreUppercase }}
          <button
            class="column-sort-btn"
            (click)="onColumnSortClick(col, $event)"
            tabindex="0"
            [attr.aria-label]="'Sort by ' + (col | removeUnderscoreUppercase)"
          >
            <span *ngIf="sortColumn !== col">⇅</span>
            <span *ngIf="sortColumn === col">
              <span *ngIf="sortAsc">▲</span>
              <span *ngIf="!sortAsc">▼</span>
            </span>
          </button>
        </div>
      </th>
    </tr>
  </thead>
    <tbody>
      <span *ngIf="(dataArray | documentsFilter : filter).length == 0"
        >No items to be displayed</span
      >
      <tr
        *cdkVirtualFor="let rowData of dataArray | documentsFilter : filter"
        class="grid-row"
      >
        <ng-container >
          <th scope="row" *ngFor="let item of headersArray; let i = index"  [@enterItems]="{ value: '', params: { delay: i * 50 } }" >
            <span
              class="link"
              *ngIf="item == 'link'"
              class="link"
              (click)="navigateToPdf(rowData['file_name'], null)"
              [attr.aria-label]="item + ': ' + rowData[item]"
            >
              <app-open-button
                [ariaLabel]="'Link to document: ' + rowData['document_name']"
              ></app-open-button>
            </span>
            <span
              *ngIf="item != 'link'"
              tabindex="0"
              [attr.aria-label]="rowData[item]"
              [attr.aria-label]="
                (item | removeUnderscoreUppercase) + ': ' + rowData[item]
              "
              >{{ rowData[item] }}</span
            >
          </th>
        </ng-container>
      </tr>
    </tbody>
  </table>
</cdk-virtual-scroll-viewport>
<!-- 
      <tr *ngFor="let rowData of dataArray | documentsFilter:filter " [ngClass]="">
        <ng-container *ngFor="let item of headersArray">
            <th scope="row">
              
              <span class="link" *ngIf="item == 'link'"class="link" [routerLink]="['/auth/pdf', rowData['file_name']]">
                <app-open-button></app-open-button>
              </span>
              <span *ngIf="item != 'link'">{{rowData[item]}}</span>
            </th>
            
        </ng-container>
      </tr> -->" table.component.scss "
@import "../../assets/styles/designsystem.scss";


.form-group {
    margin: 0px 0px 20px;
    width: 300px;
    align-self: flex-end;
}

.viewport {
    height: calc(100vh - 250px);
    width: 100%;

}
.table {
    //border-collapse: separate !important;
    

    thead {
        
        tr {
            

            th {
                background-color: $color-gray-2;
                color: $color-gray-8;
                font-weight: $font-weight-5;
                padding-left: 15px;
                cursor: pointer;
                
            }

            th:hover {
               color: $color-gray-8; 
            }

            .selectedColumn {
                color: $black !important;
            }
    
            th:first-of-type {
                border-top-left-radius: $border-radius-2;
                min-width: 350px;
                max-width: 500px;
                
            }

            th:not(first-of-type) {
                text-align: center;
                table-layout: fixed;
            }

            th:last-of-type {
                border-top-right-radius: $border-radius-2;
                //border-right: 1px solid $color-gray-5;
                //border-top: 1px solid $color-gray-5;
            }
        }



    }

    tbody {



        tr:nth-child(even) th {
            background-color: #F4F7FC;
        }
        
        tr {
            th:first-of-type {
                border-left: 1px solid $color-gray-5;
                width: 8em;
                min-width: 8em;
                max-width: 8em;
                word-break: break-all;
                
            }

            th {
                font-weight: $font-weight-3;
                padding-left: 15px;
                border-right: 1px solid $color-gray-5;
                max-width: 200px;
                overflow: hidden;

            }

            th:not(first-of-type) {
                text-align: center
            }
        }
    }

}

.table-toolbar {
    display: flex;
    justify-content: flex-end;
    margin-bottom: 10px;
    width: 90%;
    margin: 0 auto 10px auto;
    .filter-btn {
      padding: 8px 14px;
      background: $color-gray-2;
      color: $black;
      border: none;
      border-radius: $border-radius-2;
      font-size: $font-size-3;
      cursor: pointer;
      &:hover {
        background: $color-gray-4;
      }
    }
  }
  
  .export-all-container {
    display: flex;
    justify-content: flex-end;
    width: 90%;
    margin: 0 auto 10px;
  
    .export-all-btn {
      padding: 8px 14px;
      background-color: $color-2;
      color: $white;
      border: none;
      border-radius: $border-radius-2;
      font-size: $font-size-3;
      cursor: pointer;
      &:hover { background-color: $color-2-lighten-2; }
    }
  }
  
  .header-cell {
    display: flex;
    align-items: center;
    justify-content: space-between;
  
    .column-filter-btn {
      background: transparent;
      border: none;
      cursor: pointer;
      font-size: $font-size-3;
      padding: 0 4px;
      &:hover { color: $color-2; }
    }
  }
  
  .export-all-container {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  width: 90%;
  margin: 0 auto 10px;

  .export-all-btn {
    /* existing styles */
  }

  .sort-options {
    margin-left: 16px;
    display: flex;
    align-items: center;

    label {
      margin-right: 6px;
      font-size: $font-size-3;
      color: $color-gray-8;
    }
    select {
      padding: 6px 8px;
      font-size: $font-size-3;
      border: 1px solid $color-gray-5;
      border-radius: $border-radius-2;
      cursor: pointer;
      background: $white;
    }
  }

  .selectedColumn {
    color: $color-2 !important;
    font-weight: bold;
    background: #f5f9ff;
  }
  .column-sort-btn {
    background: transparent;
    border: none;
    cursor: pointer;
    font-size: 1.1em;
    padding: 0 4px;
    margin-left: 5px;
    line-height: 1;
  }
  .sort-indicator {
    margin-bottom: 8px;
    font-size: 1em;
    color: $color-gray-8;
    padding-left: 10px;
  }
  
}" table.component.ts "import { Component, Input } from "@angular/core";
import { NgFor, NgClass, NgIf } from "@angular/common";
import { OpenButtonComponent } from "../open-button/open-button.component";
import { KeyValuePipe } from "@angular/common";
import { Router, RouterModule } from "@angular/router";
import { VirtualScrollerModule } from "@iharbeck/ngx-virtual-scroller";
import { ScrollingModule } from "@angular/cdk/scrolling";
import { RemoveUnderscoreUppercasePipe } from "../pipes/remove-underscore-uppercase.pipe";
import { DocumentsFilterPipe } from "../pipes/documents-filter.pipe";
import { NgModel, FormsModule } from "@angular/forms";
import { BaseComponent } from "../base/base.component";
import { slideInUpOnEnterAnimation } from "angular-animations";

/**
 * Table component
 */
@Component({
  selector: "app-table",
  standalone: true,
  imports: [
    NgFor,
    NgClass,
    NgIf,
    KeyValuePipe,
    FormsModule,
    DocumentsFilterPipe,
    ScrollingModule,
    VirtualScrollerModule,
    RemoveUnderscoreUppercasePipe,
    OpenButtonComponent,
    RouterModule,
  ],
  providers: [VirtualScrollerModule],
  templateUrl: "./table.component.html",
  styleUrl: "./table.component.scss",
  animations: [
    slideInUpOnEnterAnimation({ anchor: 'enterItems', duration: 600})

  ]
})
export class TableComponent extends BaseComponent {

  @Input() headersArray = [];
  private _dataArray = [];
  @Input() set dataArray(val) {
    this._dataArray = val || [];
    this.applySort(); // Always sort when new data comes in
  }
  get dataArray() {
    return this._dataArray;
  }
  filter: string;
  sortColumn = 'document_name'; // Default
  sortAsc = true;               // Default

    /**
   * Flag for it sort column is a boolean value
   */
    booleanValue = false;


  // Map columns to their sort type
  columnTypes = {
    document_name: 'string',
    file_name: 'string',
    fiscal_year: 'number',
    last_updated: 'date'
  };


    // Sort on load
    ngOnInit() {
      this.applySort();
    }

  /**
   * Sort by the column passed in
   * @param colName 
   * @param boolean 
   */
  sort(colName, boolean) {

    this.sortColumn = colName;

    if (boolean == true) {
      this.dataArray.sort((a, b) =>
        a[colName] < b[colName] ? 1 : a[colName] > b[colName] ? -1 : 0
      );
      this.booleanValue = !this.booleanValue;
    } else {
      this.dataArray.sort((a, b) =>
        a[colName] > b[colName] ? 1 : a[colName] < b[colName] ? -1 : 0
      );
      this.booleanValue = !this.booleanValue;
    }
  }

  /**
   * Function for export all
   */
  exportAll() {
    // 1. Get headers and data
    const headers = this.headersArray;
    const rows = this._dataArray; // or this.dataArray
  
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
  


  
    onColumnSortClick(colName: string, event: MouseEvent) {
      event.stopPropagation();
      if (this.sortColumn === colName) {
        this.sortAsc = !this.sortAsc;
      } else {
        this.sortColumn = colName;
        this.sortAsc = true;
      }
      this.applySort();
    }
  
    private applySort() {
      const col = this.sortColumn;
      const asc = this.sortAsc;
      const type = this.columnTypes[col] || 'string';
      const sorted = [...this._dataArray].sort((a, b) => {
        let diff: number;
        if (type === 'number') {
          diff = Number(a[col]) - Number(b[col]);
        } else if (type === 'date') {
          const dateA = a[col] ? new Date(a[col]).getTime() : 0;
          const dateB = b[col] ? new Date(b[col]).getTime() : 0;
          diff = dateA - dateB;
        } else {
          diff = (a[col] || '').toString().localeCompare((b[col] || '').toString());
        }
        // For fiscal_year and last_updated, "ascending" means most recent first
        if (col === 'fiscal_year' || col === 'last_updated') {
          return asc ? -diff : diff;
        }
        return asc ? diff : -diff;
      });
      this._dataArray = sorted;
    }

    // helper function for getting the sort label
    getSortLabel(col: string, asc: boolean): string {
      if (col === 'fiscal_year' || col === 'last_updated') {
        return asc ? 'Most Recent → Oldest' : 'Oldest → Most Recent';
      }
      return asc ? 'A → Z' : 'Z → A';
    }
    

}" document-list.component.html "
<h1 class="page-title" tabindex="0" [attr.aria-label]="'Page title: Documents'">Documents</h1>

<div class="toolbar-row">
  <input
    type="search"
    class="form-control"
    [(ngModel)]="filter"
    (ngModelChange)="updateFilter()"
    placeholder="Search Bar"
    tabindex="0"
    [attr.aria-label]="'Filter documents- current filter value is: ' + filter"
  />
  <button class="sort-topics-btn" (click)="sortByTopics()">Sort by Topics</button>
</div>

<div class="entities-container">
  <div class="selected-list-entity" 
    
    (click)="submitEntity(ent)" *ngFor="let ent of selectedEntitiesList">
    {{ent}}
  </div>

  <div class="entity" 
    [ngClass]="{'selected-entity': ent === selectedEntity }"
    (click)="submitEntity(ent)" *ngFor="let ent of entitiesList">
    {{ent}}
  </div>
</div>


<div *ngIf="loading" class="loading-container" [attr.aria-label]="'documents page is loading'">
    <app-loading></app-loading>
  </div>

<app-table 
    *ngIf="headersArray.length > 0 && dataArray.length > 0"
    class="table-container"

    [headersArray]="headersArray" 
    [dataArray]="dataArray">

</app-table>

<!-- <div *ngIf="headersArray.length == 0 || dataArray.length == 0">
    An error occured
</div> -->" document=list.component.scss "@import "../../assets/styles/designsystem.scss";


:host {
    background-color: $white;
    min-height: calc(100%);
    overflow: auto;
    padding-bottom: 20px;
    overflow: hidden;

    display: flex;
    flex-direction: column;
    flex: 1;
}

.entities-container {
    display: block;
    flex-direction: row;
    flex: 1;
    width: 90%;
    margin: 10px auto;

    .selected-list-entity {
        display: inline-block;
        background-color: $color-2;
        color: $white;
        font-size: $font-size-3;
        padding: 10px ;
        border-radius: $border-radius-2; 
        margin-right: 10px;
        margin-bottom: 10px;
        cursor: pointer;

        &:hover {
            background-color: $color-2-lighten-2;
            color: $white;
        }
    }

    .entity {
        display: inline-block;
        background-color: $color-gray-2;
        font-size: $font-size-3;
        padding: 10px ;
        border-radius: $border-radius-2; 
        margin-right: 10px;
        margin-bottom: 10px;
        cursor: pointer;

        &:hover {
            background-color: $color-2;
            color: $white;
        }
    }

    .selected-entity {
        background-color: $color-gray-4;
    }

}

.form-group {
    margin: 0px 80px 0px;
    width: 300px;
    align-self: flex-end;
}

.page-title {
    display: block;
    width: 90%;
    margin: 40px auto 20px;
    flex-direction: row;

    font-size: $font-size-6;
    font-weight: $font-weight-4;
    color: $color-13;
}

.table-container {
    display: flex;
    flex-direction: column;
    width: 90%;
    margin: 20px auto;
}


.toolbar-row {
    display: flex;
    flex-direction: row;
    align-items: center;
    margin: 0px 80px 10px 0px;
    width: 90%;
    margin: 0 auto 10px auto;
  
    .form-control {
      width: 300px;
      margin-right: 12px;
    }
    .sort-topics-btn {
      padding: 8px 14px;
      background: $color-2;
      color: $white;
      border: none;
      border-radius: $border-radius-2;
      font-size: $font-size-3;
      cursor: pointer;
      &:hover {
        background: $color-2-lighten-2;
      }
    }
  }
  " document-list.compoent.ts "import { Component } from '@angular/core';
import { TableComponent } from '../table/table.component';
import { BaseComponent } from '../base/base.component';
import { ApiService } from '../../services/api-service.service';
import { NgClass, NgFor, NgIf } from '@angular/common';
import { DocumentsFilterPipe } from '../pipes/documents-filter.pipe';
import { LoadingComponent } from '../loading/loading.component';
import { NgModel, FormsModule } from "@angular/forms";
import Debounce from "debounce-decorator";

/**
 * Document List Component
 * Parent component used to display a table of documents available
 */
@Component({
  selector: 'app-documents-list',
  standalone: true,
  imports: [BaseComponent, TableComponent, NgIf, NgFor, NgClass, FormsModule, DocumentsFilterPipe, LoadingComponent],
  templateUrl: './documents-list.component.html',
  styleUrl: './documents-list.component.scss'
})
export class DocumentsListComponent extends BaseComponent{

  /**
   * String used to track the filter value
   */
  filter: string; 

  /**
   * Array of header names to be used
   * REMOVING DOCUMENT TYPE AND FOCUS AREA FOR THE TIME BEING
   */
  //headersArray = ['document_name', 'document_type', 'file_name', 'fiscal_year', 'focus_area' , 'link'];
  headersArray = ['document_name',  'file_name', 'fiscal_year',  'link', 'last_updated'];

  /**
   * Data array to be used for the documents table
   */
  dataArray = [
  ];
  /**
   * Data object 
   */
  data = {};

  /**
   * Selected Entities List
   */
  selectedEntitiesList = [];

  /**
   * List of entitites 
   */
  entitiesList = [];

  selectedEntity = null;

  /**
   * loading flag
   */
  loading = false;


  /**
   * Constructor function
   * @param apiService 
   */
  constructor (
    private apiService: ApiService){
      super();
    }

  /**
   * NgOnInit function
   * Runs when the component is loaded
   */
  ngOnInit(){
    this.loading = true;
    this.getDocumentInfo();
    
  }

  ngAfterViewInit(){
    
  }

  /**
   * Gets the column headers from the data object
   * @returns 
   */
  getColumnHeaders(){
    return Object.keys(this.data[0]);
  }


  updateFilter(){
    //console.log('Update filter', this.filter);
    this.getDocumentInfo();
  }

  // sorts documents but has yet to be implemented
  sortByTopics() {
    // Sorting logic goes here, if needed
    alert('Sort by Topics not implemented yet.');
  }

  /**
   * Get the documents available from the server
   */
  @Debounce(500)
  getDocumentInfo(){
    this.data = {};

    console.log(this.filter);
    this.apiService.postGetDocs(this.filter, this.selectedEntitiesList, 400).subscribe(result => {
      this.data = result.documents;

      this.entitiesList = result.entities;
      //console.log(this.entitiesList);
      //console.log(this.data);
      //this.headersArray = this.getColumnHeaders();
      // Sort docuements by A - Z
      this.dataArray = result.documents.sort((a, b) =>
        a.document_name.toString().localeCompare(b.document_name.toString())
    );
      this.loading = false;
    });
  }


  submitEntity(entity){

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
  

}"
