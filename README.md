<div class="modal-container" [ngClass]="{'new-section-documents-modal': createNewInstanceMode}" 
    *ngIf="selectSourcesModal" [@bounceInUpOnEnter] [@bounceOutDownOnLeave]>
  <div class="header-row">
    <div class="modal-title">Research Document Selection</div>
    <fa-icon class="fa-icon" [icon]="faXmark" (click)="documentSelectionClosed($event)"></fa-icon>
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

  <div class="primary-col">
    <div class="top-row">
      <div class="items-link" [routerLink]="['/write']"><fa-icon class="fa-icon" [icon]="faFile"></fa-icon></div>
      <div class="flex-column-spacer">
        <div class="header-report-title" *ngIf="showHeaderReportTitle" [@fadeInOnEnter] [@fadeOutOnLeave]>{{report?.report_name}}</div>
      </div>

      <button class="read-only-button" 
        [ngClass]="{'read-only-mode-on': readOnlyMode, 'read-only-mode-off': !readOnlyMode}"
        (click)="toggleReadOnlyMode()">
        <span *ngIf="!readOnlyMode">Read Only</span>
        <span *ngIf="readOnlyMode">Exit Read Only</span>
      </button>

      <button class="download-button" [disabled]="!downloadReportAvailable" [ngClass]="{'disabled-download-button': !downloadReportAvailable}" (click)="downloadReport(report?.id, report?.report_name)">
        Download Report
      </button>
    </div>

    <div class="primary-content-container" [ngClass]="{'read-only-primary-content-container': readOnlyMode}" #scrollContainer>
      <div class="section-container-row">
        <div class="controls-column" *ngIf="!readOnlyMode"></div>
        <div class="section-column">
          <div class="section-card" [ngClass]="{'section-card-edit': !readOnlyMode, 'section-card-read': readOnlyMode}">
            <div class="section-title" id="report-title">
              <span *ngIf="!editReportTitle">{{report?.report_name}}</span>
              <fa-icon *ngIf="!editReportTitle && !readOnlyMode" (click)="openEditReportTitle(); $event.stopPropagation()" class="fa-icon" [icon]="faPen"></fa-icon>
              <input (click)="$event.stopPropagation()" id="edit-report-title" [(ngModel)]="tempReportTitle" class="edit-section-title-input" *ngIf="editReportTitle" type="text" placeholder="{{report?.report_name}}">
              <fa-icon *ngIf="editReportTitle" class="check-icon" [icon]="faCheck" (click)="changeReportTitle(); $event.stopPropagation()"></fa-icon>
            </div>

            <div class="section-content-placeholder" *ngIf="report?.report_details?.report_description">{{report?.report_details?.report_description}}</div>
          </div>
        </div>
      </div>

      <div class="section-container-row" *ngIf="report?.sections.length == 0">
        <div class="controls-column" *ngIf="!readOnlyMode"></div>
        <div class="section-column">
          <div class="section-card" [ngClass]="{'section-card-edit': !readOnlyMode, 'section-card-read': readOnlyMode}" (click)="addNewSection()">
            <span>This report is blank, add a new section to get started</span>
          </div>
        </div>
      </div>

      <div class="section-container-row" *ngIf="report?.report_details?.created_from_template">
        <div class="controls-column" *ngIf="!readOnlyMode"></div>
        <div class="section-column">
          <div class="section-card-message">
            <span>This report was created from template: {{report?.report_details?.created_from_template}}</span>
          </div>
        </div>
      </div>

      <div class="section-container" *ngFor="let section of report?.sections; let sectionIndex = index" [id]="'section-' + sectionIndex">

        <div class="flex">
          <div class="controls-column" *ngIf="!readOnlyMode">

            <div class="section-details" *ngIf="sectionIndex === currentSection">
              <div class="section-title">
                <fa-icon class="fa-icon" [icon]="faFileLines"></fa-icon>
                <ng-container *ngIf="createNewInstanceMode">Selected sources for generating response</ng-container>
                <ng-container *ngIf="!createNewInstanceMode">Sources used for generated response</ng-container>
              </div>

              <ng-container *ngFor="let instance of section?.content_array">
                <ng-container *ngIf="instance.id === section?.selected_instance_id">
                  <div class="source" [@fadeInOnEnter] *ngFor="let source of instance?.sources">
                    <ng-container *ngIf="source?.title">{{source?.title}}</ng-container>
                    <ng-container *ngIf="source?.document_name">{{source?.document_name}}</ng-container>
                  </div>
                </ng-container>
              </ng-container>

              <div class="position-bottom">
                <button class="select-source-button tooltip-parent" 
                        [ngClass]="{'select-source-button-new-instance': createNewInstanceMode}"
                        *ngIf="sectionIndex === currentSection" (click)="selectSourcesModal = true">
                  Select Specific Sources
                  <span class="tooltiptext">Add or remove sources for this section</span>
                </button>
              </div>
            </div>

            <div class="section-previous-responses" *ngIf="sectionIndex === currentSection">
              <div class="section-title" *ngIf="!createNewInstanceMode">
                <fa-icon class="fa-icon" [icon]="faCodeBranch"></fa-icon>
                Previous Responses
                <span>{{section?.content_array.length}} responses</span>
              </div>
              
              <ng-container *ngIf="!createNewInstanceMode">
                <ng-container *ngFor="let response of section?.content_array">
                  <div class="previous-response" (click)="selectInstance(response?.id, true); $event.stopPropagation();" [@fadeInOnEnter]>
                    <fa-icon class="fa-icon" [icon]="faRegCircle" *ngIf="response?.id !== section?.selected_instance_id"></fa-icon>
                    <fa-icon class="fa-icon" [icon]="faCircle" *ngIf="response?.id === section?.selected_instance_id"></fa-icon>
                    {{response?.instance_description}}
                  </div>
                </ng-container>
              </ng-container>

              <button *ngIf="!createNewInstanceMode"
                      class="create-new-response"
                      tabindex="0"
                      (click)="createNewResponse($event)"
                      [attr.aria-label]="'Submit query to populate section'">
                <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon> Create a new version
              </button>
            </div>

          </div>
          
          <div class="section-column">
            <div class="section-card" (click)="selectSection(sectionIndex, $event)"
                [ngClass]="{
                  'selected-section-card': sectionIndex === currentSection, 
                  'section-card-new-edit': sectionIndex === currentSection && createNewInstanceMode && !readOnlyMode, 
                  'section-card-edit': !createNewInstanceMode && !readOnlyMode, 
                  'section-card-read': readOnlyMode
                }">

              <div class="section-title" *ngIf="sectionIndex !== currentSection">
                <span>{{section?.section_name}}</span>
              </div>

              <div class="section-title" *ngIf="sectionIndex === currentSection">
                <span *ngIf="!editSectionTitle">{{section?.section_name}}</span>
                <fa-icon *ngIf="!editSectionTitle" (click)="openEditSectionTitle(); $event.stopPropagation()" class="fa-icon" [icon]="faPen"></fa-icon>
                <input (click)="$event.stopPropagation()" [(ngModel)]="tempSectionTitle" class="edit-section-title-input" *ngIf="editSectionTitle" type="text" placeholder="{{section?.section_name}}">
                <fa-icon *ngIf="editSectionTitle" class="check-icon" [icon]="faCheck" (click)="changeSectionTitle(); $event.stopPropagation()"></fa-icon>
              </div>

              <div class="section-content-placeholder" *ngIf="section?.section_settings?.section_description">{{section?.section_settings?.section_description}}</div>

              <div class="section-content-placeholder" *ngIf="sectionIndex !== currentSection && section?.section_settings?.section_type">Section Type: {{section?.section_settings?.section_type}}</div>
              <div class="section-content-placeholder" *ngIf="sectionIndex !== currentSection && section?.section_settings?.structure">Structure: {{section?.section_settings?.structure}}</div>
              <div class="section-content-placeholder" *ngIf="sectionIndex !== currentSection && section?.section_settings?.tone">Tone: {{section?.section_settings?.tone}}</div>

              <div class="section-content-placeholder" *ngIf="!section?.section_settings?.section_description && !section?.content_array[section?.content_array?.length -1]?.content">
                Describe what this section is about...
              </div>

              <ng-container *ngFor="let instance of section?.content_array">
                <div class="section-content" *ngIf="instance.id === section?.selected_instance_id">
                  <!-- speaking_notes uses raw HTML to avoid markdown auto-outline -->
                  <ng-container *ngIf="speakingModeActive; else normalMd">
                    <div [innerHTML]="instance?.content"></div>
                  </ng-container>
                  <ng-template #normalMd>
                    <div [innerHTML]="instance?.content | markdown"></div>
                  </ng-template>
                </div>
              </ng-container>

              <!-- Speaking Notes: Header editor (no bubbling) -->
              <div class="query-row"
                   *ngIf="speakingModeActive && section?.section_name === 'Header' && sectionIndex === currentSection"
                   (click)="$event.stopPropagation()">
                <form class="input-container" (click)="$event.stopPropagation()" [ngClass]="{'new-response-container': createNewInstanceMode}">
                  <div class="form-title">Speaking Notes Header</div>

                  <div class="form-row" (click)="$event.stopPropagation()">
                    <span class="type-item">Role:</span>
                    <select class="form-input"
                            [(ngModel)]="headerRole"
                            (ngModelChange)="onHeaderControlChange()"
                            [ngModelOptions]="{standalone: true}"
                            (click)="$event.stopPropagation()">
                      <option>Director</option>
                      <option>Vice Director</option>
                    </select>
                  </div>

                  <div class="form-row" (click)="$event.stopPropagation()">
                    <span class="type-item">Topic:</span>
                    <input class="form-input"
                          type="text"
                          placeholder="Topic"
                          [(ngModel)]="headerTopic"
                          (ngModelChange)="onHeaderControlChange()"
                          [ngModelOptions]="{standalone: true}"
                          (click)="$event.stopPropagation()">
                  </div>

                  <div class="form-row" (click)="$event.stopPropagation()">
                    <span class="type-item">Date:</span>
                    <input class="form-input"
                          type="date"
                          [(ngModel)]="headerDate"
                          (ngModelChange)="onHeaderControlChange()"
                          [ngModelOptions]="{standalone: true}"
                          (click)="$event.stopPropagation()">
                  </div>

                  <div class="form-row align-self-end">
                    <button type="button"
                            class="base-submit-button query-submit-button"
                            (click)="applySpeakingHeaderFromControls(); $event.stopPropagation()">
                      Apply Header
                    </button>
                  </div>
                </form>
              </div>
              <!-- /Header editor -->

              <div class="query-row" *ngIf="sectionIndex === currentSection">
                <form class="input-container" [ngClass]="{'new-response-container': createNewInstanceMode}">
                  <div>
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
                    [attr.aria-label]="'Text input query to populate section'"></textarea>

                  <form [formGroup]="sectionSettingsForm" (ngSubmit)="testForm()">
                    <div ngbDropdown class="d-inline-block">
                      <span class="tooltip-parent">
                        Section type: 
                        <span class="tooltiptext">Describes the type of section being utilized</span>
                      </span>
                      <button type="button" class="btn btn-primary settings-dropdown" id="dropdownBasic2" ngbDropdownToggle
                        [ngClass]="{'new-response-submit': createNewInstanceMode,'query-submit-button': !createNewInstanceMode}"
                        (ngModelChange)="updateSectionType(section, selectedSectionType)">
                        {{selectedSectionType?.section_type | removeUnderscoreUppercase}}
                      </button>
                      <div ngbDropdownMenu aria-labelledby="dropdownBasic2">
                        <button ngbDropdownItem *ngFor="let item of sectionTypes" [value]="item" (click)="updateSectionType(section, item)">{{item?.section_type | removeUnderscoreUppercase}}</button>
                      </div>
                    </div>

                    <div class="form-row" *ngIf="section?.section_settings?.section_type">
                      <span class="type-item">Description: {{section?.section_settings?.section_description}}</span>
                    </div>

                    <div class="form-row" *ngIf="section?.section_settings?.tone">
                      <span class="type-item">Tone: {{section?.section_settings?.tone}}</span>
                    </div>

                    <div class="form-row" *ngIf="section?.section_settings?.structure">
                      <span class="type-item">Structure: {{section?.section_settings?.structure}}</span>
                    </div>
                  </form>

                  <div class="form-row align-self-end">
                    <button type="submit"
                            class="base-submit-button"
                            [ngClass]="{'new-response-submit': createNewInstanceMode,'query-submit-button': !createNewInstanceMode}"
                            tabindex="0"
                            (click)="submitSectionQuery(report?.id, report?.sections[currentSection]?.id ,queryString, report?.sections[currentSection]); $event.stopPropagation()"
                            [attr.aria-label]="'Submit query to populate section'">
                      <span *ngIf="createNewInstanceMode">Submit for new response</span>
                      <span *ngIf="!createNewInstanceMode">Submit to modify response</span>
                    </button>
                  </div>
                </form>
              </div>

              <div class="section-controls-row" *ngIf="sectionIndex === currentSection">
                <ng-container *ngIf="!section?.locked">
                  <button class="control-button tooltip-parent" (click)="copyToClipboard(standardTpl)">
                    <fa-icon class="fa-icon" [icon]="faClone"></fa-icon>
                    <span class="tooltiptext">Copy to clipboard</span>
                  </button>
                  
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
                          [disabled]="currentSection === report?.sections.length -1">
                    <fa-icon class="fa-icon" [icon]="faArrowDown"></fa-icon>
                    <span class="tooltiptext">Move section down in report</span>
                  </button>
                  <button class="control-button tooltip-parent"
                          (click)="deleteSection(report?.id, report?.sections[currentSection]?.id)">
                    <fa-icon class="fa-icon" [icon]="faTrashCan"></fa-icon>
                    <span class="tooltiptext">Delete section</span>
                  </button>
                </ng-container>

                <ng-container *ngIf="section?.locked">
                  <button class="tooltip-parent control-button locked-button" (click)="toggleLocked()">
                    <fa-icon class="fa-icon" [icon]="faLock"></fa-icon>
                    <span class="tooltiptext">Unlock this section</span>
                  </button>
                </ng-container>
              </div>

            </div>
          </div>
        </div>

      </div>

      <div class="flex">
        <div class="add-section-row">
          <button class="minimal-button add-section tooltip-parent" *ngIf="!readOnlyMode" (click)="addNewSection('below')">
            <fa-icon class="fa-icon" [icon]="faPlus"></fa-icon>
            Add section below
            <span class="tooltiptext">Add a new section below</span>
          </button>
        </div>
      </div>

    </div>
  </div>

  <div class="right-col">
    <div class="report-title">
      <img class="edit-icon" alt="edit title icon" src="../../assets/icons/edit.svg">
    </div>

    <div class="section-tile" 
      *ngFor="let section of report?.sections; let i = index" 
      [id]="'nav-section-' + i"
      [ngClass]="{'selected-tile': currentSection === i &&  !createNewInstanceMode, 'selected-tile-empty': currentSection === i &&  createNewInstanceMode}"
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

      <div class="line-container" *ngIf="section?.section_name !== 'Section' && section?.content_array[0]?.content?.length > 0">
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

<ng-template #standardTpl></ng-template>
