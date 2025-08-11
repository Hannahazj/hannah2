table.component.html "<!-- <div class="form-group">
  <input
    type="search"
    class="form-control"
    [(ngModel)]="filter"
    placeholder="Filter results"
    tabindex="0"
    [attr.aria-label]="'Filter documents- current filter value is: ' + filter"
  />
</div> -->
<!-- TODO: build out pagination or load more for the table. -->

<!-- TODO: Add a navigation bar here -->
 
 <!--moved to document list componenent -->
<!-- <div class="form-group export-all-container">
  <button
    class="export-all-btn"
    (click)="exportAll()"
    tabindex="0"
    aria-label="Export all documents"
  >
    Export Corpus
  </button>
</div> -->


<div *ngIf="sortColumn" class="sort-indicator">
  <div class="sorted-by">
    Sorted by <b>{{ sortColumn | removeUnderscoreUppercase }}</b>
    <span *ngIf="sortAsc">({{ getSortLabel(sortColumn, true) }})</span>
    <span *ngIf="!sortAsc">({{ getSortLabel(sortColumn, false) }})</span>
  </div>

  <div class="total-documents">
    Total Documents: {{ allDataArray?.length || 0 }}
  </div>
</div>
<div class ="table-scroll-wrapper">
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
           *ngIf="col !== 'link'"
            class="column-sort-btn"
            (click)="onColumnSortClick(col, $event)"
            tabindex="0"
            [attr.aria-label]="'Sort by ' + (col | removeUnderscoreUppercase)"
          >
              <!-- default sort icon -->
              <fa-icon
                *ngIf="sortColumn !== col"
                [icon]="faSort"
              ></fa-icon>

              <!-- up arrow when active & ascending -->
              <fa-icon
                *ngIf="sortColumn === col && sortAsc"
                [icon]="faArrowUp"
              ></fa-icon>

              <!-- down arrow when active & descending -->
              <fa-icon
                *ngIf="sortColumn === col && !sortAsc"
                [icon]="faArrowDown"
              ></fa-icon>
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
<div class="alpha-index">
  <button
    *ngFor="let letter of alphaIndex"
    (click)="scrollToLetter(letter)"
    [class.active]="selectedAlpha === letter"
    aria-label="'Jump to ' + letter "
  >
    {{ letter }}
  </button>
</div>
</div>

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
      </tr> -->
"

table.component.ts"import { Component, Input, ViewChild } from "@angular/core";
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
import {faSort, faArrowDown, faArrowUp} from "@fortawesome/free-solid-svg-icons"; 
import { FontAwesomeModule } from "@fortawesome/angular-fontawesome";
import { CdkVirtualScrollViewport } from '@angular/cdk/scrolling';


/**
 * Table component
 */
@Component({
  selector: "app-table",
  standalone: true,
  imports: [
    FontAwesomeModule,
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
@ViewChild('verticalViewport') viewport!: CdkVirtualScrollViewport;


  // icons for sorting
  faSort = faSort; 
  faArrowDown = faArrowDown;
  faArrowUp = faArrowUp;

    // jump-index state:
    alphaIndex: string[] = [];
    private letterToIndex: Record<string, number> = {};
    // for selected letter
    selectedAlpha = '';
  

  // unfilterd data array for export
  @Input() allDataArray = [];
  @Input() headersArray = [];
  private _dataArray = [];
  @Input() set dataArray(val) {
    this._dataArray = val || [];
    this.applySort(); // Always sort when new data comes in
    this.buildAlphaIndex(); // build the A-Z map
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

    private buildAlphaIndex() {
      this.alphaIndex = [];
      this.letterToIndex = {};
  
      this._dataArray.forEach((row, i) => {
        // group by document_name
        const key = this.headersArray[0];
        let label = (row[key] || '').toString().charAt(0).toUpperCase();
  
        // normalize to A–Z or “#” for anything else
        if (!/[A-Z]/.test(label)) label = '#';
  
        // if we haven’t seen this letter yet, record it
        if (this.letterToIndex[label] === undefined) {
          this.letterToIndex[label] = i;
          this.alphaIndex.push(label);
        }
      });
    }
  
    scrollToLetter(letter: string) {
      this.selectedAlpha = letter;
      this.viewport.scrollToIndex(this.letterToIndex[letter], 'smooth');
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
    

}"

table.component.scss "
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
    justify-content: flex-end;
    width: 90%;
    margin: auto 10px;
  
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

}

.sort-indicator {
  width: 90%;
  margin: 8px 0;
  font-size: 1em;
  color: $color-gray-8;
}


/* ensure the wrapper lays out viewport + sidebar side by side */
.table-scroll-wrapper {
  display: flex;
  width: 100%;
  height: calc(100vh - 250px);  // or whatever your viewport height was
}

/* let the viewport grow to fill available space */
.viewport {
  flex: 1;
}

/* style the A–Z sidebar */
.alpha-index {
  width: 30px;
  margin-left: 8px;
  display: flex;
  flex-direction: column;
  align-items: center;

  button {
    background: transparent;
    border: none;
    padding: 2px 0;
    font-size: 0.8em;
    cursor: pointer;
    color: $color-gray-6;

    &:hover {
      color: $color-2;
    }
    &.active {
      color: $color-2;
      font-weight: bold;
    }
  }
}

/* strip every possible border/shadow/outline from the sort buttons */
button.column-sort-btn,
button.column-sort-btn:focus,
button.column-sort-btn:hover {
  /* kill the browser’s default button chrome */
  appearance: none !important;
  -webkit-appearance: none !important;
  -moz-appearance: none !important;

  /* remove any background or border */
  background: none !important;
  border: none     !important;

  /* remove any shadow/focus glow */
  box-shadow: none !important;
  outline: none    !important;
}


.sort-indicator {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  margin: 8px 0 0 0;   
  padding-right: 50px; /* added so that the total document counter is moved to the left */     
  color: $color-gray-8;      
  border: none;              
}

"
