import { Component, ChangeDetectorRef, ViewChild, ElementRef, input } from '@angular/core';
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
  reportId: any = null;

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

  // -------------------- speaking_notes only state --------------------
  speakingNotesApplied = false;       // prevents duplicate insertion
  speakingModeActive = false;         // gates the header editor UI
  headerRole = 'Director';            // Director | Vice Director
  headerTopic = '';                   // topic text
  headerDate: string | null = null;   // yyyy-mm-dd (from input[type=date]) or null for today
  // -------------------------------------------------------------------

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
    console.log('updateSectionType: selected', type);

    this.selectedSectionType = type;
    section.section_settings = type;

    // robust normalization to catch speaking_notes casing/format variants
    const label = (type?.section_type || type?.template_name || '').toString().trim().toLowerCase();
    const isSpeaking =
      label === 'speaking_notes' ||
      label === 'speaking-notes' ||
      label === 'speaking notes' ||
      /speaking[_\-\s]?notes/.test(label);

    const isInfoPaper =
      label === 'information_paper' ||
      label === 'information-paper' ||
      label === 'information paper' ||
      /information[_\-\s]?paper/.test(label);

    this.speakingModeActive = isSpeaking;

    if (isSpeaking) {
      console.log('[speaking_notes] detected; applying template once…');
      this.applySpeakingNotesTemplateOnce();
    }
    if (isInfoPaper) {
      // reserved for later wiring
    }
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

      // Allow download
      this.downloadReportAvailable = true; 

      // (optionally) select the new section here
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

      // Select new section and set focus
      this.selectSection(newSectionIndex, null);

      // Allow download
      this.downloadReportAvailable = true; 

    })
  }

  moveElementInArray(arr, old_index, new_index) {
    // Ensure indices are within valid bounds
    if (old_index < 0 || old_index >= arr.length || new_index < 0 || new_index >= arr.length) {
      console.error("Invalid index provided.");
      return arr; // Return original array if indices are invalid
    }
  
    const [elementToMove] = arr.splice(old_index, 1); 
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


  createNewResponse(event){
    console.log('new response')
    // Enter create new instance mode
    this.createNewInstanceMode = true;
    
    this.currentInstancePosition = null;

    this.updateSelectedDocuments(event);

    this.cdr.detectChanges();

  }
