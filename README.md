import { Component, Input, ViewChild } from "@angular/core";
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
    

}
