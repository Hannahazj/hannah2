given my rempot gen component.html "




<div class="modal-container" [ngClass]="{'new-section-documents-modal': createNewInstanceMode}" 
    *ngIf="selectSourcesModal" [@bounceInUpOnEnter] [@bounceOutDownOnLeave]>
    <div class="header-row">
      <div class="modal-title" > 
        Research Document Selection
      </div>
      
      <fa-icon class="fa-icon" [icon]="faXmark" (click)="documentSelectionClosed($event)" ></fa-icon>
  </div>
    
    
  <div class="output-flex-row">

    <report-documents-selection 
        [incomingSelectedDocuments]="report?.sections[currentSection]?.content_array[currentInstancePosition]?.sources"
        [createNewInstanceMode]="createNewInstanceMode"
        (selectedDocumentsChange)="updateSelectedDocuments($event)" 
        (closeModalEmit)="documentSelectionClosed($event)"
        class="flex-col-2">
    </report-documents-selection>
  </div>
</div>

<div class="no-report" *ngIf="showNoReportSelected" [@fadeOutOnLeave]>
    <app-items-page></app-items-page>

</div>


<div class="body-row" *ngIf="!showNoReportSelected" [@fadeOutOnLeave]>



    <div class="primary-col"  >
        <div class="top-row">
            <div class="items-link" [routerLink]="['/write']" ><fa-icon class="fa-icon" [icon]="faFile"></fa-icon></div>
            <!-- <div class="word-count">{{report?.sections[currentSection]?.content_array[currentInstance]?.content | wordCount}}<span> Words</span> ---- {{currentSection}} </div> -->
            <div class="flex-column-spacer">
                <div class="header-report-title" *ngIf="showHeaderReportTitle" [@fadeInOnEnter] [@fadeOutOnLeave]>{{report?.report_name}}</div>

            </div>
            
            <button class="read-only-button" 
                [ngClass]="{'read-only-mode-on': readOnlyMode, 'read-only-mode-off': !readOnlyMode}"
                (click)="toggleReadOnlyMode()"> 
                <span *ngIf="!readOnlyMode">Read Only</span>
                <span *ngIf="readOnlyMode">Exit Read Only</span>
            </button>
            
            <button class="download-button" [disabled]="!downloadReportAvailable" [ngClass]="{'disabled-download-button': !downloadReportAvailable}" (click)="downloadReport(report?.id, report?.report_name)"><!-- <fa-icon class="fa-icon" [icon]="faPrint"></fa-icon> --> Download Report</button>
        </div>



        <div class="primary-content-container" [ngClass]="{'read-only-primary-content-container': readOnlyMode}" #scrollContainer>
            <div class="section-container-row">
                <div class="controls-column" *ngIf="!readOnlyMode" ></div>
                <div class="section-column">
                    <div class="section-card" [ngClass]="{'section-card-edit': !readOnlyMode, 'section-card-read': readOnlyMode}" >
                        <div class="section-title" id="report-title">
                            <span *ngIf="!editReportTitle">{{report?.report_name}}  </span>
                            <fa-icon *ngIf="!editReportTitle && !readOnlyMode"(click)="openEditReportTitle(); $event.stopPropagation()" class="fa-icon" [icon]="faPen"></fa-icon>
                            <input (click)="$event.stopPropagation()" id="edit-report-title"  [(ngModel)]="tempReportTitle" class="edit-section-title-input" *ngIf="editReportTitle" type="text" placeholder="{{report?.report_name}}">
                            <fa-icon *ngIf="editReportTitle" class="check-icon" [icon]="faCheck" (click)="changeReportTitle(); $event.stopPropagation()" ></fa-icon>
                        
                        </div>

                        <div class="section-content-placeholder" *ngIf="report?.report_details?.report_description">{{report?.report_details?.report_description}}</div>
                    </div>
                </div>
            </div>

<!--             <div class="section-container">
                <div class="controls-column"  *ngIf="!readOnlyMode" [@slideInLeftOnEnter]></div>
                <div class="section-column">
                    <div class="section-card" >
                        <div class="section-title" id="report-subtitle">
                            <span *ngIf="!editReportSubtitle && report?.report_subtitle">{{report?.report_subtitle}}</span>
                            <span *ngIf="!editReportSubtitle && !report?.report_subtitle">Sub Title</span> 
                            <fa-icon *ngIf="!editReportSubtitle" (click)="openEditReportSubtitle(); $event.stopPropagation()" class="fa-icon" [icon]="faPen"></fa-icon>
                            <input (click)="$event.stopPropagation()"  [(ngModel)]="tempReportSubtitle" class="edit-section-title-input" *ngIf="editReportSubtitle" type="text" 
                                placeholder="{{report?.report_subtitle}}">
                            <fa-icon *ngIf="editReportSubtitle" class="check-icon" [icon]="faCheck" (click)="changeReportSubtitle(); $event.stopPropagation()" ></fa-icon>
                        
                        </div>
                    </div>
                </div>
            </div> -->

            <div class="section-container-row" *ngIf="report?.sections.length == 0">
                <div class="controls-column"  *ngIf="!readOnlyMode"></div>
                <div class="section-column">
                    <div class="section-card" [ngClass]="{'section-card-edit': !readOnlyMode, 'section-card-read': readOnlyMode}" (click)="addNewSection()">
                       <!--  <div class="section-title">
                            <span >This report is blank, add a new section to get started  </span>

                        </div> -->
                        <span >This report is blank, add a new section to get started  </span>
                    </div>
                </div>

            </div>

            <div class="section-container-row" *ngIf="report?.report_details?.created_from_template">
                <div class="controls-column"  *ngIf="!readOnlyMode" ></div>
                <div class="section-column">
                    <div class="section-card-message" >
                       <!--  <div class="section-title">
                            <span >This report is blank, add a new section to get started  </span>

                        </div> -->
                        <span >This report was created from template: {{report?.report_details?.created_from_template}} </span>
                    </div>
                </div>

            </div>


            <div class="section-container" *ngFor="let section of report?.sections; let sectionIndex = index" [id]="'section-' + sectionIndex">

<!--                 <div class="flex">
                    <div class="add-section-row" *ngIf="sectionIndex === currentSection">

                        <button class="minimal-button add-section  tooltip-parent" *ngIf="!readOnlyMode" (click)="addNewSection('above')">
                            <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon>
                            Add section above
                            <span class="tooltiptext">Add a new section above</span>
                        </button>
                    </div>
                </div> -->

                <div class="flex">
                    <div class="controls-column"  *ngIf="!readOnlyMode" >

                        <div class="section-details"  *ngIf="sectionIndex === currentSection" >
                            <div class="section-title">
                                <fa-icon class="fa-icon" [icon]="faFileLines"></fa-icon>
                                    <ng-container *ngIf="createNewInstanceMode">Selected sources for generating response</ng-container> 
                                    <ng-container *ngIf="!createNewInstanceMode">Sources used for generated response</ng-container> 
                            </div>
    
    <!--                         <div *ngIf="section?.content_array?.length > 0 && section?.content_array[currentInstancePosition].sources?.length == 0" class="source" [@fadeInOnEnter] [@fadeOutOnLeave]>
                                Utilizing all available sources  
                            </div> -->
    
                            
                            <ng-container *ngFor="let instance of section?.content_array">
                                <ng-container *ngIf="instance.id === section?.selected_instance_id">

                                    <div class="source" [@fadeInOnEnter] 
                                        *ngFor="let source of instance?.sources">
                                        <ng-container *ngIf="source?.title">{{source?.title}}</ng-container>
                                        <ng-container *ngIf="source?.document_name">{{source?.document_name}}</ng-container>
                                   
                                    </div>
                                </ng-container>
                            </ng-container>
    
        
    
    
                            <div class="position-bottom">
                                <button class="select-source-button tooltip-parent" 
                                    [ngClass]="{'select-source-button-new-instance': createNewInstanceMode}"
                                    *ngIf="sectionIndex === currentSection" (click)="selectSourcesModal = true">
                                   <!--  <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon> -->
                                    Select Specific Sources
                                    <span class="tooltiptext">Add or remove sources for this section</span>
                                </button>
                            </div>
                        </div>
    
    
                    <!--  Future when we display multiple responses -->
    
                        <div class="section-previous-responses"  *ngIf="sectionIndex === currentSection" > 
                            <div class="section-title" *ngIf="!createNewInstanceMode">
                                <fa-icon class="fa-icon" [icon]="faCodeBranch"></fa-icon>
                                Previous Responses
                                <span>{{section?.content_array.length}} responses</span>
                            </div>
                            
                            <ng-container *ngIf="!createNewInstanceMode">
                                <ng-container *ngFor="let response of section?.content_array">
                                    <div class="previous-response"  (click)="selectInstance(response?.id, true); $event.stopPropagation();" [@fadeInOnEnter] >
                                        <fa-icon class="fa-icon" [icon]="faRegCircle" *ngIf="response?.id !== section?.selected_instance_id"></fa-icon>
                                        <fa-icon class="fa-icon" [icon]="faCircle" *ngIf="response?.id === section?.selected_instance_id"></fa-icon>
                                        {{response?.instance_description}}
                                        <!-- <fa-icon *ngIf="response?.favorite" class="fa-icon favorite-fa-icon" [icon]="faHeart"></fa-icon> -->
                                    </div>
                                </ng-container>
                            </ng-container>

                            
                            <button
                                *ngIf="!createNewInstanceMode"
                                class="create-new-response"
                                tabindex="0"
                                (click)="createNewResponse($event)"
                                [attr.aria-label]="'Submit query to populate section'"
                                >
                                <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon> Create a new version
                            </button>
    
                        </div> 
    
    
    
                    </div>
                    
                    <div class="section-column">
                        <div class="section-card"  (click)="selectSection(sectionIndex, $event)"  
                            [ngClass]="{'selected-section-card': sectionIndex === currentSection, 
                                        'section-card-new-edit': sectionIndex === currentSection && createNewInstanceMode && !readOnlyMode, 
                                        'section-card-edit': !createNewInstanceMode && !readOnlyMode, 
                                        'section-card-read': readOnlyMode}">
    
                            <!-- [@dataChange]="currentSection" [@slideInRightOnEnter] [@slideOutRightOnLeave] -->
    
                            <div class="section-title" *ngIf="sectionIndex !== currentSection"> 
                                <span>{{section?.section_name}}  </span>
                             </div>
    
                            <div class="section-title" *ngIf="sectionIndex === currentSection" >
                                <span *ngIf="!editSectionTitle">{{section?.section_name}}  </span>
                                <fa-icon *ngIf="!editSectionTitle" (click)="openEditSectionTitle(); $event.stopPropagation()" class="fa-icon" [icon]="faPen"></fa-icon>
                                <input (click)="$event.stopPropagation()"  [(ngModel)]="tempSectionTitle" class="edit-section-title-input" *ngIf="editSectionTitle" type="text" placeholder="{{section?.section_name}}">
                                <fa-icon *ngIf="editSectionTitle" class="check-icon" [icon]="faCheck" (click)="changeSectionTitle(); $event.stopPropagation()" ></fa-icon>
                            
                            </div>
    
                            <div class="section-content-placeholder" *ngIf="section?.section_settings?.section_description">{{section?.section_settings?.section_description}}</div>

                            <div class="section-content-placeholder" *ngIf="sectionIndex !== currentSection && section?.section_settings?.section_type">Section Type: {{section?.section_settings?.section_type}}</div>
                            <div class="section-content-placeholder" *ngIf="sectionIndex !== currentSection && section?.section_settings?.structure">Structure: {{section?.section_settings?.structure}}</div>
                            <div class="section-content-placeholder" *ngIf="sectionIndex !== currentSection && section?.section_settings?.tone">Tone: {{section?.section_settings?.tone}}</div>


    
                            <div class="section-content-placeholder" *ngIf="!section?.section_settings?.section_description && !section?.content_array[section?.content_array?.length -1]?.content" >
                                Describe what this section is about...
                            </div>
    
                            <ng-container *ngFor="let instance of section?.content_array">
                                <div class="section-content" 
                                    *ngIf="instance.id === section?.selected_instance_id" 
                                    [innerHTML]="instance?.content | markdown">
    
                                </div>
                            </ng-container>
    
    
                            
    
                            <div class="query-row" *ngIf="sectionIndex === currentSection">
                                <form class="input-container" 
                                    [ngClass]="{'new-response-container': createNewInstanceMode}" >

                                    <div >
                                            <span class="form-title" *ngIf="!createNewInstanceMode">Build upon this response for this section</span>
                                            <span class="form-title" *ngIf="createNewInstanceMode">Create a new response for this section</span>

                                    </div>

                                    <textarea
                                    type="text"
                                    class="form-control"
                                    placeholder="Describe what this section is about.."
                                    id="query-input"
    
                                    (click)="$event.stopPropagation()"
                                    (keyup.enter)="submitSectionQuery(report?.id, report?.sections[currentSection]?.id ,queryString, report?.sections[currentSection])"
                                    autocomplete="off"
                                    [(ngModel)]="queryString"
                                    [ngModelOptions]="{standalone: true}"
                                    tabindex="0"
                                    [attr.aria-label]="'Text input query to populate section'"
                                    ></textarea>

                                    <form [formGroup]="sectionSettingsForm" (ngSubmit)="testForm()">


                                        <div ngbDropdown  class="d-inline-block">
                                            <span class="tooltip-parent">
                                                Section type: 
                                                <span class="tooltiptext">Describes the type of section being utilized</span>
                                            </span>
                                            <button type="button" class="btn btn-primary settings-dropdown" id="dropdownBasic2" ngbDropdownToggle      
                                                [ngClass]="{
                                                    'new-response-submit': createNewInstanceMode, 
                                                    'query-submit-button': !createNewInstanceMode
                                                }"
                                                (ngModelChange)="updateSectionType(section, selectedSectionType)">{{selectedSectionType?.section_type | removeUnderscoreUppercase}}</button>
                                            <div ngbDropdownMenu aria-labelledby="dropdownBasic2">
                                              
                                              <button ngbDropdownItem *ngFor="let item of sectionTypes" [value]="item" (click)="updateSectionType(section, item)">{{item?.section_type | removeUnderscoreUppercase}}</button>
                                            </div>
                                          </div>
                                        <div class="form-row" *ngIf="section?.section_settings?.section_type">
                                            <span class="type-item">
                                               Description: 
                                               {{section?.section_settings?.section_description}}
                                           </span>
                                       </div>

                                       <div class="form-row" *ngIf="section?.section_settings?.tone">
                                            <span class="type-item">
                                            Tone: 
                                            {{section?.section_settings?.tone}}
                                            </span>
                                        </div>

                                        <div class="form-row" *ngIf="section?.section_settings?.structure">
                                            <span class="type-item">
                                            Structure: 
                                            {{section?.section_settings?.structure}}
                                            </span>
                                        </div>
<!--                                     <div class="form-row" *ngIf="section?.section_settings?.section_type">
                                             <span class="tooltip-parent">
                                                Section type: 
                                                <span class="tooltiptext">Describes the type of section being utilized</span>
                                            </span>
                                            <input class="form-input" 
                                                type="text" 
                                                formControlName="type"
                                                placeholder="{{section?.section_settings?.section_type}}" 
                                                (click)="$event.stopPropagation()">
                                        </div>
                                        <div class="form-row" *ngIf="section?.section_settings?.section_type">
                                            <span class="tooltip-parent">
                                                Structure: 
                                                <span class="tooltiptext">Describes the structure desired in the response</span>
                                            </span>
                                            <input class="form-input" 
                                                type="text" 
                                                formControlName="structure"
                                                placeholder="{{section?.section_settings?.structure}}" 
                                                (click)="$event.stopPropagation()">
                                        </div>
                                        <div class="form-row" *ngIf="section?.section_settings?.tone">
                                            <span class="tooltip-parent">
                                                Tone: 
                                                <span class="tooltiptext">Describes the tone desired in the response</span>
                                            </span>
                                            <input class="form-input" 
                                                type="text" 
                                                formControlName="tone"
                                                placeholder="{{section?.section_settings?.tone}}" 
                                                (click)="$event.stopPropagation()">
                                        </div> -->
                                    </form>
                                    
                                    <div class="form-row align-self-end">
                                        <button
                                            type="submit"
                                            class="base-submit-button"
                                            [ngClass]="{
                                                'new-response-submit': createNewInstanceMode, 
                                                'query-submit-button': !createNewInstanceMode
                                            }"

                                            tabindex="0"
                                            (click)="submitSectionQuery(report?.id, report?.sections[currentSection]?.id ,queryString, report?.sections[currentSection] ); $event.stopPropagation()"
                                            [attr.aria-label]="'Submit query to populate section'"
                                            >
                                            

                                            <span *ngIf="createNewInstanceMode">Submit for new response</span>
                                            <span *ngIf="!createNewInstanceMode">Submit to modify response</span>
                                        </button>

                                    </div>

                                </form>
                            </div> 
    
    
    
                            <div class="section-controls-row" *ngIf="sectionIndex === currentSection">
    
    
                                <ng-container *ngIf="!section?.locked">
    <!--                                 <button class="control-button tooltip-parent" (click)="toggleLocked()">
                                        <fa-icon class="fa-icon" [icon]="faUnlock"></fa-icon>
                                        <span class="tooltiptext">Lock this section</span>
                                    </button> -->
    
                                    <button class="control-button tooltip-parent" (click)="copyToClipboard(standardTpl)">
                                        <fa-icon class="fa-icon" [icon]="faClone"></fa-icon>
                                        <span class="tooltiptext">Copy to clipboard</span>
                                    </button>
    
    <!--                                 <button class="control-button tooltip-parent" (click)="toggleFavorite()" *ngIf="section?.content_array[currentInstance]?.favorite">
                                        <fa-icon class="fa-icon" [icon]="faHeart"></fa-icon>
                                        <span class="tooltiptext">Unfavorite instance</span>
                                    </button>
    
                                    <button class="control-button tooltip-parent" 
                                        (click)="toggleFavorite()" 
                                        *ngIf="!section?.content_array[currentInstance]?.favorite">
                                        <fa-icon class="fa-icon" [icon]="faHeartRegular"></fa-icon>
                                        <span class="tooltiptext">Favorite instance</span>
                                    </button> -->
                                    
                                    <button class="control-button no-border-right tooltip-parent"
                                        [ngClass]="{'disabled': currentSection == 0}"
                                        [disabled]="currentSection === 0"
                                        (click)="moveSectionUp()">
                                        <fa-icon class="fa-icon" [icon]="faArrowUp"></fa-icon>
                                        <span class="tooltiptext">Move section up in report</span>
                                    </button>
                                    <button class="control-button tooltip-parent"
                                        (click)="moveSectionDown()"
                                        [ngClass]="{'disabled': currentSection === report?.sections.length -1}"
                                        [disabled]="currentSection === report?.sections.length -1"
                                        >
                                        <fa-icon class="fa-icon" [icon]="faArrowDown"></fa-icon>
                                        <span class="tooltiptext">Move section down in report</span>
                                    </button>
                                    <button class="control-button tooltip-parent"
                                        (click)="deleteSection(report?.id, report?.sections[currentSection]?.id )">
                                        <fa-icon class="fa-icon" [icon]="faTrashCan"></fa-icon>
                                        <span class="tooltiptext">Delete section</span>
                                    </button>
                                </ng-container>
    
                                <ng-container *ngIf="section?.locked">
                                    <button class="tooltip-parent control-button locked-button " (click)="toggleLocked()">
                                        <fa-icon class="fa-icon" [icon]="faLock" ></fa-icon>
                                        <span class="tooltiptext">Unlock this section</span>
                                    </button>
                                        
                                </ng-container>
    
                            </div>
    
    
                        </div>
    
    
    
               <!--          <button class="minimal-button add-section  tooltip-parent" (click)="addNewSection('below')">
                            <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon>
                        Add a new section
                        <span class="tooltiptext">Add addtional section</span>
                        </button> -->
                    </div>
                </div>

<!-- 
                <div class="flex">
                    <div class="add-section-row" *ngIf="sectionIndex === currentSection">

                        <button class="minimal-button add-section  tooltip-parent" *ngIf="!readOnlyMode" (click)="addNewSection('below')">
                            <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon>
                            Add section below
                            <span class="tooltiptext">Add a new section below</span>
                        </button>
                    </div>
                </div> -->



            </div>

            <div class="flex">
                <div class="add-section-row">

                    <button class="minimal-button add-section  tooltip-parent" *ngIf="!readOnlyMode" (click)="addNewSection('below')">
                        <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon>
                        Add section below
                        <span class="tooltiptext">Add a new section below</span>
                    </button>
                </div>
            </div> 

<!--             <div class="flex-row-justify-end" >
                <button class="minimal-button add-section  tooltip-parent" *ngIf="!readOnlyMode" (click)="addNewSection()">
                    <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon>
                    Add a new section
                    <span class="tooltiptext">Add addtional section</span>
                </button>
            </div> -->

        </div>
    </div>

    <div class="right-col">

        <div class="report-title">
            <img class="edit-icon" alt="edit title icon" src="../../assets/icons/edit.svg">
        </div>

<!--         <div class="report-title">
            <img class="edit-icon" alt="edit title icon" src="../../assets/icons/edit.svg">
        </div> -->

        <div class="section-tile" 
        *ngFor="let section of report?.sections; let i = index" 
        [id]="'nav-section-' + i"
        [ngClass]="{'selected-tile': currentSection === i &&  !createNewInstanceMode, 'selected-tile-empty': currentSection === i &&  createNewInstanceMode, }"
        (click)="selectSection(i, $event)">

            <div class="empty-title-and-content line-diagonal" *ngIf="section?.section_name == 'Section' && section?.content_array?.length == 0">
                <img class="edit-icon" alt="edit title icon" src="../../assets/icons/edit.svg">
                <fa-icon class="fa-icon doc-icon" [icon]="faFileLines"></fa-icon>
            </div>

            <div class="empty-title" *ngIf="section?.section_name == 'Section' && section?.content_array?.length > 0">
                <img class="edit-icon" alt="edit title icon" src="../../assets/icons/edit.svg">
            </div>

            <div class="empty-content" *ngIf="section?.section_name !== 'Section' && section?.content_array?.length == 0">
                <fa-icon class="fa-icon doc-icon" [icon]="faFileLines"></fa-icon>
            </div>

            <div class="line-container " *ngIf="section?.section_name !== 'Section' && section?.content_array[0]?.content?.length > 0">
                <div class="half-line"></div>
                <div class="full-line"></div>
                <div class="full-line"></div>
                <div class="full-line"></div>
                <div class="half-line"></div>

                <ng-content *ngIf="section?.content_array[0]?.content?.length > 700">
                    <div class="empty-line"></div>

                    <div class="half-line"></div>
                    <div class="full-line"></div>
                    <div class="full-line"></div>
                    <div class="full-line"></div>
                    <div class="half-line"></div>
                </ng-content>

            </div>

    </div>
       
    </div>
</div>



<ng-template #standardTpl></ng-template>" and my  report gen ocomponent.ts "import { Component, ChangeDetectorRef, ViewChild, ElementRef, input } from '@angular/core';
import { BaseComponent } from '../base/base.component';
import { 
  FormsModule, 
  Validators, 
  FormBuilder, 
  FormGroup, 
  ReactiveFormsModule, NgModel } from '@angular/forms';
import { NgClass, NgFor, NgIf } from '@angular/common';
import { Router, ActivatedRoute, RouterModule } from '@angular/router';
import { ReportDocumentsSelectionComponent } from '../report-documents-selection/report-documents-selection.component';
import { ItemsPageComponent } from '../items-page/items-page.component';
import { NgbDropdownModule } from '@ng-bootstrap/ng-bootstrap';


// Icons
import {  faCircle, faDotCircle, faHeart as faHeartRegular} from "@fortawesome/fontawesome-free-regular"
import { faCheck, faPen, faXmark, faFile, faUnlock, faLock, faHeart, faClone, faArrowUp, faArrowDown, faTrashCan, faFileLines, faPlus, faPrint, faCodeBranch   } from '@fortawesome/free-solid-svg-icons';
import { FaIconComponent } from "@fortawesome/angular-fontawesome";

// Pipes
import { WordCountPipe } from '../pipes/word-count.pipe';
import { MarkdownPipe } from '../pipes/markdown.pipe';
import { RemoveUnderscoreUppercasePipe } from '../pipes/remove-underscore-uppercase.pipe';

// Animations
import { trigger, state, style, animate, transition } from '@angular/animations';
import { fadeInOnEnterAnimation, fadeOutOnLeaveAnimation, bounceInUpOnEnterAnimation,  bounceOutDownOnLeaveAnimation, slideInRightOnEnterAnimation, slideInUpOnEnterAnimation, slideOutRightOnLeaveAnimation } from 'angular-animations';

// Services
import { ToastService } from "../../services/toast-service";
import { ReportsService } from '../../services/api-reports.service';

    
@Component({
  selector: 'app-report-gen',
  standalone: true,
  imports: [
    NgFor,
    NgIf, 
    NgClass,
    FaIconComponent,
    ItemsPageComponent,
    ReportDocumentsSelectionComponent,
    FormsModule,
    NgbDropdownModule,
    WordCountPipe,
    MarkdownPipe,
    RemoveUnderscoreUppercasePipe,
    ReactiveFormsModule,
    RouterModule
  ],
  templateUrl: './report-gen.component.html',
  styleUrl: './report-gen.component.scss',
  animations: [
    fadeInOnEnterAnimation({anchor: 'quickFadeIn', duration: 200, delay: 100}),
    fadeInOnEnterAnimation({duration: 200, delay: 200}),
    fadeOutOnLeaveAnimation({duration: 300}),
    slideInUpOnEnterAnimation({ translate: "100vh" }),
    bounceInUpOnEnterAnimation(),
    bounceOutDownOnLeaveAnimation(),
    slideInRightOnEnterAnimation({translate: "100vw"}),
    slideOutRightOnLeaveAnimation({translate: "100vw"}),
    trigger('dataChange', [
      state('*', style({
        opacity: 1,
        transform: 'scale(1)'
      })),
      transition('* => *', [
        animate('400ms ease-in-out', style({
          opacity: 0,
          transform: 'translateX(1000px) scale(0.5)'
        })),
        animate('400ms ease-in-out', style({
          opacity: 1,
          transform: 'scale(1)'
        }))
      ])
    ])
  ]
  
})
export class ReportGenComponent extends BaseComponent{

  /** Font Awesome Icon */
  faCheck = faCheck;
  /** Font Awesome Icon */
  faPen = faPen;

  /** Font Awesome Icon */
  faFile = faFile;
  /** Font Awesome Icon */
  faDotCircle = faDotCircle;
  /** Font Awesome Icon */
  faUnlock  = faUnlock;
  /** Font Awesome Icon */
  faLock = faLock;
  /** Font Awesome Icon */
  faClone = faClone;
  /** Font Awesome Icon */
  faHeartRegular = faHeartRegular;
  /** Font Awesome Icon */
  faHeart = faHeart;
  /** Font Awesome Icon */
  faArrowUp = faArrowUp;
  /** Font Awesome Icon */
  faArrowDown = faArrowDown;
  /** Font Awesome Icon */
  faTrashCan = faTrashCan;
  /** Font Awesome Icon */
  faFileLines = faFileLines;
  /** Font Awesome Icon */
  faPlus = faPlus;
  /** Font Awesome Icon */
  faCodeBranch = faCodeBranch;


  /**
   * used to track current section index
   */
  currentSection = 0;

  /**
   * 
   */
  currentInstancePosition = 0;

  /**
   * Report Id
   */
  reportId = null;

  /**
   * Flag to toggle the select sources modal
   */
  selectSourcesModal = false; 

  /**
   * Flag if section title is being edited
   */
  editSectionTitle = false;

  /**
   * Temporary Section title as the user is editing
   */
  tempSectionTitle = "";

  /**
   * Flag if the report title is being edited
   */
  editReportTitle = false;

  /**
   * Flag if the report subtitle is being edited
   */
  editReportSubtitle = false; 

  /**
   * Temp variable to track new report title
   */
  tempReportTitle = "";
  /**
   * Temp variable to track new report subtitle
   */
  tempReportSubtitle = "";

  /** 
   * Control read only view
   */
  readOnlyMode = false; 

  /**
   * Controls if the download report is available to the user
   */
  downloadReportAvailable = true; 

  /**
   * Primary Report Object
   */
  report: ReportObject = null;

  /**
   * Variable for user query input
   */
  queryString = "";

  /**
   * Controls when the header shows the report title
   * This is controlled by the users scroll position
   */
  showHeaderReportTitle = false; 

  /**
   * Used to show if no report is selected
   */
  showNoReportSelected = false; 

  /**
   * Mode to endter when user is creating a new instance without 
   */
  createNewInstanceMode = false;

  /**
   * Form group for section settings
   */
  sectionSettingsForm: FormGroup;

  /**
   * sectionTypes array to be populated from service
   */
  sectionTypes = [];

  selectedSectionType = null;

  /**
   * Watching the scroll container
   */
  @ViewChild('scrollContainer') scrollableElement!: ElementRef;

  /**
   * Constructor
   * @param fb 
   * @param toastService 
   * @param reportsService 
   * @param router 
   * @param cdr 
   */
  constructor(
      private fb: FormBuilder,
      private toastService: ToastService,
      public reportsService: ReportsService,
      private router: ActivatedRoute,
      private cdr: ChangeDetectorRef,
    ){
    super();

    this.sectionSettingsForm = this.fb.group({
      type: null,
      structure: null,
      tone: null
    })

  }

  /**
   * NgOnInit
   */
  ngOnInit(){

    // get section types
    this.reportsService.getSectionTypes().subscribe({
      next: (result) => {

        if (result && result.section_types){
          this.sectionTypes = result.section_types;
        }
        console.log(this.sectionTypes);
      }
    })

    // Check if there is a report id in the url
    this.router.queryParams.subscribe((params) => {
      this.reportId = params['id'];
      this.openReport(this.reportId);
    });
  }

  /**
   * ngAfterViewInit
   */
  ngAfterViewInit() {
    // If there is a report, set event listener to watch the scroll container
    if (this.report){
      //this.scrollableElement.nativeElement.addEventListener('scroll', this.onScroll.bind(this));
    }
  }

  testForm(){
    console.log(this.sectionSettingsForm);
  }

  /**
   * Show the header title in the top bar based on scroll position
   */
  onScroll(){
    if (this.scrollableElement.nativeElement.scrollTop > 100){
      this.showHeaderReportTitle = true; 
    } else {
      this.showHeaderReportTitle = false; 
    }
  }

  /**
   * Open report via the report id
   * @param reportId 
   * @returns 
   */
  openReport(reportId){
    // NO Report Id
    if (!reportId){
      this.showNoReportSelected = true;
      return; 
    } else {
      this.showNoReportSelected = false;

      this.reportsService.getReport(reportId).subscribe({
        next: (result) => {
          result = this.setSelectedInstancesIfNeeded(result);
          this.report = result;

          console.log(this.report);

          //
          if (this.report.sections.length > 0){
            this.selectSection(0, null);
          }

          // Add scroll detect when a report is loaded
          this.scrollableElement.nativeElement.addEventListener('scroll', this.onScroll.bind(this));
          this.cdr.detectChanges();
        }

      
      })
    }
  }

  /**
   * Request report from service for download
   * @param reportId 
   * @param fileName 
   */
  downloadReport(reportId, fileName){
    this.reportsService.downloadReport(reportId).subscribe({
      next: (result) => {
        this.base64ToDocx(result.docx_file, fileName)
      }
    })
  }
  
  /**
   * Tracking the 
   * @param event 
   */
  documentSelectionClosed(event){
    this.selectSourcesModal = false
    console.log(event);
  }

  /**
   * Convert base64 string to docx file
   * @param base64String 
   * @param filename 
   */
  base64ToDocx(base64String, filename) {
    const byteCharacters = atob(base64String);
    const byteArrays = [];
  
    for (let offset = 0; offset < byteCharacters.length; offset += 512) {
      const slice = byteCharacters.slice(offset, offset + 512);
  
      const byteNumbers = new Array(slice.length);
      for (let i = 0; i < slice.length; i++) {
        byteNumbers[i] = slice.charCodeAt(i);
      }
  
      const byteArray = new Uint8Array(byteNumbers);
      byteArrays.push(byteArray);
    }
  
    const blob = new Blob(byteArrays, { type: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document' });
  
    const link = document.createElement('a');
    link.href = URL.createObjectURL(blob);
    link.download = filename;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }

  /**
   * Set the selected instance variable 
   * @param reportObject 
   * @returns 
   */
  setSelectedInstancesIfNeeded(reportObject:ReportObject){

    reportObject.sections.forEach( section => {
      let matchFound = false; 
      section.content_array.forEach( instance => {
        if (instance.id === section.selected_instance_id){
            matchFound = true;
        } 
      })

      if (!matchFound && section.content_array.length > 0 ){
        section.selected_instance_id = section.content_array[section.content_array.length - 1].id;
      }
      
    })
    return reportObject;
  }

  /**
   * Open the report title edit field
   */
  openEditReportTitle(){
    this.editReportTitle = true; 

      setTimeout(() => {
        let inputSelector = 'edit-report-title';
  
        let inputEle = document.getElementById(inputSelector);
        if (inputEle instanceof HTMLElement) { inputEle.focus()}
      }, 5)

  }

  /**
   * Open the report subtitle edit
   * NOT COMPLETED
   */
  openEditReportSubtitle(){
    this.editReportSubtitle = true; 
  }

  /**
   * Submit the Updated report title
   */
  changeReportTitle(){

    this.editReportTitle = false;

    this.report.report_name = this.tempReportTitle;

    //Save changes
    this.sendUpdateReport(this.report);

  }

  /**
   * Submit the updated report subtitle
   * 
   */
  changeReportSubtitle(){

    this.editReportSubtitle = false;
    this.report.report_subtitle = this.tempReportSubtitle;

    //Save changes
    this.sendUpdateReport(this.report);

  }

  /**
   * Open the edit section title for a section
   */
  openEditSectionTitle(){
    this.editSectionTitle = true;

    setTimeout(() => {
      let inputSelector = '.edit-section-title-input';

      let inputEle = document.querySelector(inputSelector);
      if (inputEle instanceof HTMLElement) { inputEle.focus()}
    }, 50)


  }

  /**
   * Submit the section title change 
   */
  changeSectionTitle(){

    // Close the section title edit
    this.editSectionTitle = false;
  
    // Update the report object 
    this.report.sections[this.currentSection].section_name = this.tempSectionTitle;

    // Submit to server
    this.updateSection(this.report.id, this.report.sections[this.currentSection].id, this.report.sections[this.currentSection]);
  }

  updateSectionType(section, type){
    console.log(section, type);

    this.selectedSectionType = type;
    section.section_settings = type;

  }


  /**
   * Add a new section to the report
   */
  addNewSection(aboveORbelow = 'below'){

    console.log(this.currentSection, aboveORbelow);

    // Prevent download 
    this.downloadReportAvailable = false; 


    this.reportsService.createSection(this.reportId, this.report, -1).subscribe(result => {
      // Update report object
      this.report = result.db_report;

      // Need to select section 
/*       const newSectionId = result.section_id;

      let newSectionIndex = this.report.sections.findIndex(obj => obj.id === newSectionId)

      // Need to move item in the array
      console.log(newSectionIndex);

      let destinationIndex;

      if (aboveORbelow == 'above' ) {
        destinationIndex = 0;
        if (this.currentSection > 0){
          destinationIndex = this.currentSection; 
        }
      }  else {
        destinationIndex = this.currentSection ; 
      }
      
      this.moveElementInArray(this.report.sections, newSectionIndex, destinationIndex);

      newSectionIndex = this.report.sections.findIndex(obj => obj.id === newSectionId)

      // Select new section and set focus
      this.selectSection(newSectionIndex, null); */

      // Allow download
      this.downloadReportAvailable = true; 

      // Update report 
      //this.sendUpdateReport(this.report);
    })
  }



  /**
   * Add a new section to the report
   */
  addNewSectionNEW(aboveORbelow = 'below'){

    console.log(this.currentSection, aboveORbelow);

    // Prevent download 
    this.downloadReportAvailable = false; 

    
    let destinationIndex;

    if (aboveORbelow == 'above' ) {
      destinationIndex = 0;
      if (this.currentSection > 0){
        destinationIndex = this.currentSection; 
      }
    }  else {
      destinationIndex = this.currentSection + 1; 
    }

    console.log(destinationIndex);
    this.reportsService.createSection(this.reportId, this.report, destinationIndex).subscribe(result => {
      // Update report object
      this.report = result.db_report;

      // Need to select section 
      const newSectionId = result.section_id;

      let newSectionIndex = this.report.sections.findIndex(obj => obj.id === newSectionId)

      // Need to move item in the array
      console.log(newSectionIndex);

/*       let destinationIndex;

      if (aboveORbelow == 'above' ) {
        destinationIndex = 0;
        if (this.currentSection > 0){
          destinationIndex = this.currentSection; 
        }
      }  else {
        destinationIndex = this.currentSection + 1; 
      } */
      
     // this.moveElementInArray(this.report.sections, newSectionIndex, destinationIndex);

      //newSectionIndex = this.report.sections.findIndex(obj => obj.id === newSectionId)

      // Select new section and set focus
      this.selectSection(newSectionIndex, null);

      // Allow download
      this.downloadReportAvailable = true; 

      // Update report 
      //this.sendUpdateReport(this.report);
    })
  }

  moveElementInArray(arr, old_index, new_index) {
    // Ensure indices are within valid bounds
    if (old_index < 0 || old_index >= arr.length || new_index < 0 || new_index >= arr.length) {
      console.error("Invalid index provided.");
      return arr; // Return original array if indices are invalid
    }
  
    // 1. Remove the element from its old position
    // splice(startIndex, deleteCount) returns an array containing the removed elements
    const [elementToMove] = arr.splice(old_index, 1); 
  
    // 2. Insert the element at the new position
    // splice(startIndex, deleteCount, item1, item2, ...) inserts items without deleting
    arr.splice(new_index, 0, elementToMove); 
  
    return arr;
  }

  /**
   * Update section
   * @param reportId 
   * @param sectionId 
   * @param sectionObject 
   */
  updateSection(reportId, sectionId, sectionObject){

    // Prevent dowenload 
    this.downloadReportAvailable = false; 

    // Submit changes
    this.reportsService.updateSection(reportId, sectionId, sectionObject).subscribe(result => {
      this.report.sections[this.currentSection] = result;

      // Allow download
      this.downloadReportAvailable = true;

      // Set focus
      this.setFocusOnInput();

    })
  }


/*   keySubmitSelection(reportId, sectionId, query, sectionObject, $event){
    console.log("SUBMIT", $event, $event.key)
    if ($event && $event.key == "Enter"){
      $event.preventDefault();
      this.submitSectionQuery(reportId, sectionId, query, sectionObject);

    }
  } */


  createNewResponse(event){
    console.log('new response')
    // Enter create new instance mode
    this.createNewInstanceMode = true;
    
    this.currentInstancePosition = null;

    this.updateSelectedDocuments(event);

    //this.updateInstancePosition(null);

    this.cdr.detectChanges();

  }

  /**
   * Used to submit a new query for a section
   * @param reportId 
   * @param sectionId 
   * @param query 
   * @param sectionObject 
   */
  submitSectionQuery(reportId, sectionId, query, sectionObject){

    // Prevent Download 
    this.downloadReportAvailable = false; 

    this.reportsService.createInstance(reportId, sectionId, query, sectionObject).subscribe({

      next: (result) => {

        // Remove temp instance if found
        let tempInstanceIndex = this.report.sections[this.currentSection].content_array.findIndex(obj => obj.id === 'temp-instance-id');
        if (tempInstanceIndex !== -1){
          this.report.sections[this.currentSection].content_array.splice(tempInstanceIndex, 1);

        }

        let sectionIndexToUpdate =  this.report.sections.findIndex((obj) => obj.id === sectionId);

        this.report.sections[sectionIndexToUpdate].content_array.push(result);

        // Activate new instance
        this.report.sections[this.currentSection].selected_instance_id = result.id;

        // Allow download 
        this.downloadReportAvailable = true; 

        // Force page update
        this.cdr.detectChanges();

      } 
    })
  }


  /**
 * Used to set user focus on the input box
 * Need to use different classes for research and non-research view
 */
  setFocusOnInput() {
  
    setTimeout(() => {
      let inputSelector = 'query-input';

      let inputEle = document.getElementById(inputSelector);
      console.log(inputEle);
      if (inputEle instanceof HTMLElement) { inputEle.focus()}
    }, 50)

  }

  getSectionTypeByType(section_type){
    console.log(section_type);

    section_type == 'text_block' ? section_type = 'free_text' : section_type = section_type;
    console.log(this.sectionTypes);
    const fullSection = this.sectionTypes.find(type => {
      return type.section_type == section_type;
    })

    return fullSection;
  }


  /**
   * Change selected section
   * @param index 
   */
  selectSection(index, event){

    if (this.readOnlyMode) {
      return;
    }

    console.log(this.report.sections[index]);

    // Early exit if needed
    if (this.report.sections.length < index){
      return;
    }

    // Close section title edit
    this.editSectionTitle = false;
    this.tempSectionTitle = '';

    // Clear query box
    this.queryString = '';

    console.log(event);

    // Clear first for annimation purposes
    this.currentSection = null;

    //setTimeout(() => {
      this.currentSection = index;

      // Determine if there are no current instances for the section
      console.log(this.report.sections[index].content_array.length)
      if (this.report.sections[index].content_array.length == 0){
        console.log('should be new instance mode')
        this.createNewInstanceMode = true; 
      }

      // Set section type

      const sectionObject = this.getSectionTypeByType(this.report.sections[index].section_settings.section_type);
      if (sectionObject) { 
        this.report.sections[index].section_settings = sectionObject;
        this.selectedSectionType = sectionObject;
      }

      


      this.selectInstance(this.report.sections[index].selected_instance_id, false);
      this.scrollToPosition(index);
      this.setFocusOnInput();
   // }, 0)

    
    
  }

  /**
   * Scroll section into view
   * @param index 
   */
  scrollToPosition(index: number) {
    const element = document.getElementById('section-' + index);

    if (element) {
      element.scrollIntoView({ behavior: 'smooth', block: 'start' });

      const navElement = document.getElementById('nav-section-' + index);
      navElement.scrollIntoView({ behavior: 'smooth', block: 'start' })
    }
  }


  /**
   * Toggle Locked for the current section
   */
  toggleLocked(){
    this.report.sections[this.currentSection].locked = !this.report.sections[this.currentSection].locked;

    this.updateSection(this.reportId, this.report.sections[this.currentSection].id, {}  );
  }

  /**
   * Copy section response to clipboard
   * @param template 
   */
  copyToClipboard(template){

    navigator.clipboard.writeText(this.report.sections[this.currentSection].content_array[0].content);
    this.toastService.show({
      value: "Copied to clipboard",
      template,
      classname: "bg-success text-light",
    });
  }
  
  /**
   * Toggle favorite for the current section
   */
  toggleFavorite(){
    //this.report.sections[this.currentSection].favorite = !this.report.sections[this.currentSection].favorite;
    //this.report.sections[this.currentSection].content_array[this.currentInstance].favorite = !this.report.sections[this.currentSection].content_array[this.currentInstance].favorite 
  }

  /**
   * Move section down one position
   */
  moveSectionDown(){


    const temp = this.report.sections[this.currentSection];
    this.report.sections[this.currentSection] = this.report.sections[this.currentSection + 1];
    this.report.sections[this.currentSection + 1] = temp;

    // move the current selected
    this.currentSection++;
    this.sendUpdateReport(this.report);
  }

  /**
   * Move section up one position
   */
  moveSectionUp(){

    const temp = this.report.sections[this.currentSection -1];

    this.report.sections[this.currentSection- 1] = this.report.sections[this.currentSection];

    this.report.sections[this.currentSection] = temp;

    // move the current selected
    this.currentSection--;

    this.sendUpdateReport(this.report);
  }

  /**
   * Delete a section
   * @param reportId 
   * @param sectionId 
   */
  deleteSection(reportId, sectionId){

    // Prevent download
    this.downloadReportAvailable = false; 
    this.reportsService.deleteSection(reportId, sectionId).subscribe({

      next: (result) => {

        this.report.sections.splice(this.currentSection, 1);

        //check for last item
        if (this.currentSection > this.report.sections.length - 1){
          this.currentSection--;
        }

        // Allow download
        this.downloadReportAvailable = true; 
      }
    })
  }

  /**
   * submit update report
   * @param report 
   */
  sendUpdateReport(report){

    // Prevent download
    this.downloadReportAvailable = false; 

    this.reportsService.updateReport(report.id, report).subscribe({
      next: (result) => {

        // Allow download
        this.downloadReportAvailable = true; 
      }
    })
  }

  /**
   * Update selected documents for a section
   * MAY NEED MORE WORK HERE
   * @param event 
   */
  updateSelectedDocuments(event){

    // Find if temp instance exists
    console.log(this.report.sections[this.currentSection].content_array.findIndex(obj => obj.id === 'temp-instance-id'));

    console.log(this.report.sections[this.currentSection]);

    if (this.report.sections[this.currentSection].content_array.findIndex(obj => obj.id === 'temp-instance-id') == -1){
      this.report.sections[this.currentSection].content_array.push(
        {
          id: 'temp-instance-id',
          instance_description: '',
          content: 'Describe what this section is about...',
          favorite: false,
          sources: event
        }
      )

      this.currentInstancePosition = 0;
    }

    console.log(this.report.sections[this.currentSection].content_array)
    
    this.report.sections[this.currentSection].selected_instance_id = 'temp-instance-id';

    //this.selectInstance('temp-instance-id');

    console.log(this.report.sections[this.currentSection].content_array[this.report.sections[this.currentSection].content_array.length -1].sources)

    //this.report.sections[this.currentSection].content_array[this.report.sections[this.currentSection].content_array.length -1].sources = event;

    this.cdr.detectChanges();

  }

  /**
   * Select instance of 
   * @param instanceId 
   * @param updateSection 
   */
  selectInstance(instanceId, updateSection = false){

      // Exit create new instance mode
      
      if (this.report.sections[this.currentSection].content_array.length == 0){
        this.createNewInstanceMode = true; 
      } else {
        this.createNewInstanceMode = false; 
      }

      this.report.sections[this.currentSection].selected_instance_id = instanceId;

      this.updateInstancePosition(instanceId);

      // Need to set the current source list somewhere so I can pass to the modal

      // TURN SAVE BACK ON WHEN SERVICE ISSUE IS FIXED
      if (updateSection){
        this.updateSection(this.report.id, this.report.sections[this.currentSection].id, this.report.sections[this.currentSection]);
      }
     

  }


  /**
   * update instance position
   * which instance is active for a section
   * @param instanceId 
   */
  updateInstancePosition(instanceId){

    let index = this.report.sections[this.currentSection].content_array.findIndex(obj => obj.id === instanceId)
    
    this.sectionSettingsForm.reset();

    console.log(index);
    console.log(this.report.sections[index]);

    let section_settings = null;

    if (this.report.sections[index] && this.report.sections[index].section_settings){
      section_settings = this.report.sections[index].section_settings

      this.sectionSettingsForm.setValue({
        type: section_settings.section_type,
        structure: section_settings.structure,
        tone: section_settings.tone
      })
    }

    this.currentInstancePosition = index;

    // need case for not found
  
  }


  /**
   * Used to store current section value while in read only mode
   */
  temporaryCurrentSection = null;


  toggleReadOnlyMode(){
    this.readOnlyMode = !this.readOnlyMode;

    if (this.readOnlyMode){
      this.temporaryCurrentSection = this.currentSection;
      this.currentSection = null;
    } else {
      this.currentSection = this.temporaryCurrentSection;
      this.temporaryCurrentSection = null;
    }
  }



  
}
" and this format for " Information Paper
MMMM DD, YYYY
SUBJECT: Clearly and succinctly specify the issue the paper discusses. Use specific description that summarizes the content, avoiding vague, one-word subjects. Clarifying the subject can help in organizing and presenting the most relevant information clearly. Do not introduce acronyms in the subject line. (1-2 lines)
PURPOSE: State what this information paper seeks to do (1 sentence)
BACKGROUND: Clearly state germane background information on the issue. (3-4 sentences max)
DISCUSSION / KEY POINTS:
1.	Clearly and succinctly present information that the reader needs to know about the subject. Explain why it is important for the recipient to have this information.
2.	Structure main points and supporting ideas in complete but succinct bulleted paragraphs. The bullets indicate divisions and relationships among concepts.
A.	Use sub-bullets to illustrate significant supporting ideas that expand on the main bullet paragraph.
3.	The organization of information should flow from the subject, audience, and purpose. Organize the information by presenting the most important information first unless information is necessary for the reader to understand the main point. Each bulleted paragraph should logically flow to the next.
4.	Use short, concise sentences in an active voice. The tone should be neutral, clear, and direct in nature. Limit sentences to one thought. Use short, simple words. Avoid using acronyms, abbreviations, and jargon.
RECOMMENDATION / WAY AHEAD: Clearly state recommendations and the way ahead.








Prepared by: Full name, e-mail, and phone number of action officer
Approved by: Full name of approving official (should be O-6/GS-15 or above for actions going to the DLA Director)

" how can i modify the code to add the tempalte for that for that formate i selected. I ideally i want it to show up as an option for a template.
