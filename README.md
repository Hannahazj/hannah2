import { Component, ChangeDetectorRef, ViewChild, ElementRef, input } from '@angular/core';
import { BaseComponent } from '../base/base.component';
import { FormsModule, Validators, FormBuilder, FormGroup, ReactiveFormsModule, NgModel } from '@angular/forms';
import { NgClass, NgFor, NgIf } from '@angular/common';
import { Router, ActivatedRoute, RouterModule } from '@angular/router';
import { ReportDocumentsSelectionComponent } from '../report-documents-selection/report-documents-selection.component';
import { ItemsPageComponent } from '../items-page/items-page.component';
import { NgbDropdownModule } from '@ng-bootstrap/ng-bootstrap';

// Icons
import { faCircle, faDotCircle, faHeart as faHeartRegular } from "@fortawesome/fontawesome-free-regular";
import { faCheck, faPen, faXmark, faFile, faUnlock, faLock, faHeart, faClone, faArrowUp, faArrowDown, faTrashCan, faFileLines, faPlus, faPrint, faCodeBranch } from '@fortawesome/free-solid-svg-icons';
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
      state('*', style({ opacity: 1, transform: 'scale(1)' })),
      transition('* => *', [
        animate('400ms ease-in-out', style({ opacity: 0, transform: 'translateX(1000px) scale(0.5)' })),
        animate('400ms ease-in-out', style({ opacity: 1, transform: 'scale(1)' }))
      ])
    ])
  ]
})
export class ReportGenComponent extends BaseComponent {

  faCheck = faCheck;
  faPen = faPen;
  faXmark = faXmark;

  faFile = faFile;
  faDotCircle = faDotCircle;
  faUnlock  = faUnlock;
  faLock = faLock;
  faClone = faClone;
  faHeartRegular = faHeartRegular;
  faHeart = faHeart;
  faArrowUp = faArrowUp;
  faArrowDown = faArrowDown;
  faTrashCan = faTrashCan;
  faFileLines = faFileLines;
  faPlus = faPlus;
  faCodeBranch = faCodeBranch;

  currentSection = 0;
  currentInstancePosition = 0;

  reportId: any = null;
  selectSourcesModal = false; 
  editSectionTitle = false;
  tempSectionTitle = "";
  editReportTitle = false;
  editReportSubtitle = false; 
  tempReportTitle = "";
  tempReportSubtitle = "";
  readOnlyMode = false; 
  downloadReportAvailable = true; 
  report: any = null; // ReportObject;
  queryString = "";
  showHeaderReportTitle = false; 
  showNoReportSelected = false; 
  createNewInstanceMode = false;

  sectionSettingsForm: FormGroup;
  sectionTypes: any[] = [];
  selectedSectionType: any = null;

  @ViewChild('scrollContainer') scrollableElement!: ElementRef;

  // ---- speaking_notes state (scoped) ----
  speakingModeActive = false;
  speakingNotesApplied = false;
  headerRole: 'Director' | 'Vice Director' = 'Director';
  headerTopic = '';
  headerDate: string | null = null; // yyyy-mm-dd
  private headerDebounce: any;
  // ---------------------------------------

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
    });
  }

  ngOnInit(){
    this.reportsService.getSectionTypes().subscribe({
      next: (result) => {
        if (result && result.section_types){
          this.sectionTypes = result.section_types;
        }
      }
    });

    this.router.queryParams.subscribe((params) => {
      this.reportId = params['id'];
      this.openReport(this.reportId);
    });
  }

  ngAfterViewInit() {
    // optionally add scroll listener after report loads
  }

  testForm(){
    console.log(this.sectionSettingsForm);
  }

  onScroll(){
    if (this.scrollableElement?.nativeElement?.scrollTop > 100){
      this.showHeaderReportTitle = true; 
    } else {
      this.showHeaderReportTitle = false; 
    }
  }

  openReport(reportId: any){
    if (!reportId){
      this.showNoReportSelected = true;
      return; 
    }

    this.showNoReportSelected = false;

    this.reportsService.getReport(reportId).subscribe({
      next: (result) => {
        result = this.setSelectedInstancesIfNeeded(result);
        this.report = result;

        // enable speaking notes behavior if template name matches
        this.speakingModeActive = this.isSpeakingNotesTemplate(this.report?.report_details?.created_from_template);
        if (this.speakingModeActive) this.applySpeakingNotesIfNeeded();

        if (this.report.sections?.length > 0){
          this.selectSection(0, null);
        }

        if (this.scrollableElement?.nativeElement){
          this.scrollableElement.nativeElement.addEventListener('scroll', this.onScroll.bind(this));
        }
        this.cdr.detectChanges();
      }
    });
  }

  downloadReport(reportId: any, fileName: string){
    this.reportsService.downloadReport(reportId).subscribe({
      next: (result) => this.base64ToDocx(result.docx_file, fileName)
    });
  }
  
  documentSelectionClosed(event: any){
    this.selectSourcesModal = false;
  }

  base64ToDocx(base64String: string, filename: string) {
    const byteCharacters = atob(base64String);
    const byteArrays: Uint8Array[] = [];
  
    for (let offset = 0; offset < byteCharacters.length; offset += 512) {
      const slice = byteCharacters.slice(offset, offset + 512);
      const byteNumbers = new Array(slice.length);
      for (let i = 0; i < slice.length; i++) byteNumbers[i] = slice.charCodeAt(i);
      byteArrays.push(new Uint8Array(byteNumbers));
    }
  
    const blob = new Blob(byteArrays, { type: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document' });
    const link = document.createElement('a');
    link.href = URL.createObjectURL(blob);
    link.download = filename;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }

  setSelectedInstancesIfNeeded(reportObject: any){
    reportObject.sections.forEach((section: any) => {
      let matchFound = false; 
      section.content_array.forEach((instance: any) => {
        if (instance.id === section.selected_instance_id) matchFound = true;
      });
      if (!matchFound && section.content_array.length > 0 ){
        section.selected_instance_id = section.content_array[section.content_array.length - 1].id;
      }
    });
    return reportObject;
  }

  openEditReportTitle(){
    this.editReportTitle = true; 
    setTimeout(() => {
      const inputEle = document.getElementById('edit-report-title');
      if (inputEle instanceof HTMLElement) inputEle.focus();
    }, 5);
  }

  openEditReportSubtitle(){
    this.editReportSubtitle = true; 
  }

  changeReportTitle(){
    this.editReportTitle = false;
    this.report.report_name = this.tempReportTitle;
    this.sendUpdateReport(this.report);
  }

  changeReportSubtitle(){
    this.editReportSubtitle = false;
    this.report.report_subtitle = this.tempReportSubtitle;
    this.sendUpdateReport(this.report);
  }

  openEditSectionTitle(){
    this.editSectionTitle = true;
    setTimeout(() => {
      const inputEle = document.querySelector('.edit-section-title-input');
      if (inputEle instanceof HTMLElement) inputEle.focus();
    }, 50);
  }

  changeSectionTitle(){
    this.editSectionTitle = false;
    this.report.sections[this.currentSection].section_name = this.tempSectionTitle;
    this.updateSection(this.report.id, this.report.sections[this.currentSection].id, this.report.sections[this.currentSection]);
  }

  updateSectionType(section: any, type: any){
    this.selectedSectionType = type;
    section.section_settings = type;
  }

  addNewSection(aboveORbelow = 'below'){
    this.downloadReportAvailable = false; 
    this.reportsService.createSection(this.reportId, this.report, -1).subscribe(result => {
      this.report = result.db_report;
      this.downloadReportAvailable = true; 
    });
  }

  addNewSectionNEW(aboveORbelow = 'below'){
    this.downloadReportAvailable = false; 
    let destinationIndex = aboveORbelow === 'above'
      ? (this.currentSection > 0 ? this.currentSection : 0)
      : this.currentSection + 1;

    this.reportsService.createSection(this.reportId, this.report, destinationIndex).subscribe(result => {
      this.report = result.db_report;
      const newSectionId = result.section_id;
      const newSectionIndex = this.report.sections.findIndex((obj: any) => obj.id === newSectionId);
      this.selectSection(newSectionIndex, null);
      this.downloadReportAvailable = true; 
    });
  }

  moveElementInArray(arr: any[], old_index: number, new_index: number) {
    if (old_index < 0 || old_index >= arr.length || new_index < 0 || new_index >= arr.length) {
      console.error("Invalid index provided.");
      return arr;
    }
    const [elementToMove] = arr.splice(old_index, 1); 
    arr.splice(new_index, 0, elementToMove); 
    return arr;
  }

  updateSection(reportId: any, sectionId: any, sectionObject: any){
    this.downloadReportAvailable = false; 
    this.reportsService.updateSection(reportId, sectionId, sectionObject).subscribe(result => {
      this.report.sections[this.currentSection] = result;
      this.downloadReportAvailable = true;
      this.setFocusOnInput();
    });
  }

  createNewResponse(event: any){
    this.createNewInstanceMode = true;
    this.currentInstancePosition = null;
    this.updateSelectedDocuments(event);
    this.cdr.detectChanges();
  }

  submitSectionQuery(reportId: any, sectionId: any, query: string, sectionObject: any){
    this.downloadReportAvailable = false; 
    this.reportsService.createInstance(reportId, sectionId, query, sectionObject).subscribe({
      next: (result) => {
        const tempIdx = this.report.sections[this.currentSection].content_array.findIndex((obj: any) => obj.id === 'temp-instance-id');
        if (tempIdx !== -1) this.report.sections[this.currentSection].content_array.splice(tempIdx, 1);

        const sectionIndexToUpdate = this.report.sections.findIndex((obj: any) => obj.id === sectionId);
        this.report.sections[sectionIndexToUpdate].content_array.push(result);

        this.report.sections[this.currentSection].selected_instance_id = result.id;

        this.downloadReportAvailable = true; 
        this.cdr.detectChanges();
      } 
    });
  }

  setFocusOnInput() {
    setTimeout(() => {
      const inputEle = document.getElementById('query-input');
      if (inputEle instanceof HTMLElement) inputEle.focus();
    }, 50);
  }

  getSectionTypeByType(section_type: string){
    const normalized = section_type === 'text_block' ? 'free_text' : section_type;
    return this.sectionTypes.find((type: any) => type.section_type === normalized);
  }

  selectSection(index: number, event: any){
    if (this.readOnlyMode) return;
    if (this.report.sections.length < index) return;

    this.editSectionTitle = false;
    this.tempSectionTitle = '';
    this.queryString = '';

    this.currentSection = null as any;
    this.currentSection = index;

    if (this.report.sections[index].content_array.length == 0){
      this.createNewInstanceMode = true; 
    }

    // Guard against missing section_settings
    const secSettings = this.report.sections[index]?.section_settings;
    if (secSettings?.section_type) {
      const sectionObject = this.getSectionTypeByType(secSettings.section_type);
      if (sectionObject) {
        this.report.sections[index].section_settings = sectionObject;
        this.selectedSectionType = sectionObject;
      }
    }

    this.selectInstance(this.report.sections[index].selected_instance_id, false);
    this.scrollToPosition(index);
    this.setFocusOnInput();
  }

  scrollToPosition(index: number) {
    const element = document.getElementById('section-' + index);
    if (element) {
      element.scrollIntoView({ behavior: 'smooth', block: 'start' });
      const navElement = document.getElementById('nav-section-' + index);
      navElement?.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  }

  toggleLocked(){
    this.report.sections[this.currentSection].locked = !this.report.sections[this.currentSection].locked;
    this.updateSection(this.reportId, this.report.sections[this.currentSection].id, {});
  }

  copyToClipboard(template: any){
    navigator.clipboard.writeText(this.report.sections[this.currentSection].content_array[0].content);
    this.toastService.show({ value: "Copied to clipboard", template, classname: "bg-success text-light" });
  }

  toggleFavorite(){}

  moveSectionDown(){
    const temp = this.report.sections[this.currentSection];
    this.report.sections[this.currentSection] = this.report.sections[this.currentSection + 1];
    this.report.sections[this.currentSection + 1] = temp;
    this.currentSection++;
    this.sendUpdateReport(this.report);
  }

  moveSectionUp(){
    const temp = this.report.sections[this.currentSection - 1];
    this.report.sections[this.currentSection - 1] = this.report.sections[this.currentSection];
    this.report.sections[this.currentSection] = temp;
    this.currentSection--;
    this.sendUpdateReport(this.report);
  }

  deleteSection(reportId: any, sectionId: any){
    this.downloadReportAvailable = false; 
    this.reportsService.deleteSection(reportId, sectionId).subscribe({
      next: () => {
        this.report.sections.splice(this.currentSection, 1);
        if (this.currentSection > this.report.sections.length - 1){
          this.currentSection--;
        }
        this.downloadReportAvailable = true; 
      }
    });
  }

  sendUpdateReport(report: any){
    this.downloadReportAvailable = false; 
    this.reportsService.updateReport(report.id, report).subscribe({
      next: () => this.downloadReportAvailable = true
    });
  }

  updateSelectedDocuments(event: any){
    const idx = this.report.sections[this.currentSection].content_array.findIndex((obj: any) => obj.id === 'temp-instance-id');
    if (idx === -1){
      this.report.sections[this.currentSection].content_array.push({
        id: 'temp-instance-id',
        instance_description: '',
        content: 'Describe what this section is about...',
        favorite: false,
        sources: event
      });
      this.currentInstancePosition = 0;
    }
    this.report.sections[this.currentSection].selected_instance_id = 'temp-instance-id';
    this.cdr.detectChanges();
  }

  selectInstance(instanceId: any, updateSection = false){
    if (this.report.sections[this.currentSection].content_array.length == 0){
      this.createNewInstanceMode = true; 
    } else {
      this.createNewInstanceMode = false; 
    }

    this.report.sections[this.currentSection].selected_instance_id = instanceId;
    this.updateInstancePosition(instanceId);

    if (updateSection){
      this.updateSection(this.report.id, this.report.sections[this.currentSection].id, this.report.sections[this.currentSection]);
    }
  }

  // FIX: read section_settings from active section, not by instance index
  updateInstancePosition(instanceId: any){
    const activeSection = this.report.sections[this.currentSection];
    const idx = activeSection.content_array.findIndex((obj: any) => obj.id === instanceId);
    this.sectionSettingsForm.reset();

    const section_settings = activeSection?.section_settings || null;
    if (section_settings){
      this.sectionSettingsForm.setValue({
        type: section_settings.section_type,
        structure: section_settings.structure,
        tone: section_settings.tone
      });
    }
    this.currentInstancePosition = idx;
  }

  temporaryCurrentSection: any = null;

  toggleReadOnlyMode(){
    this.readOnlyMode = !this.readOnlyMode;
    if (this.readOnlyMode){
      this.temporaryCurrentSection = this.currentSection;
      this.currentSection = null as any;
    } else {
      this.currentSection = this.temporaryCurrentSection;
      this.temporaryCurrentSection = null;
    }
  }

  // ---------------- speaking_notes helpers ----------------

  private isSpeakingNotesTemplate(name: any): boolean {
    const n = (name || '').toString().trim().toLowerCase();
    return /speaking[_\-\s]?notes/.test(n);
  }

  private formatCivilianDate(d?: string | null) {
    if (!d) {
      return new Date().toLocaleDateString(undefined, { month: 'long', day: '2-digit', year: 'numeric' });
    }
    const parts = d.split('-');
    if (parts.length === 3) {
      const dt = new Date(+parts[0], +parts[1]-1, +parts[2]);
      return dt.toLocaleDateString(undefined, { month: 'long', day: '2-digit', year: 'numeric' });
    }
    return d;
  }

  private ensureTempInstanceWithContent(sectionIndex: number, contentHtml: string) {
    const sec = this.report.sections[sectionIndex];
    let idx = sec.content_array.findIndex((i: any) => i.id === 'temp-instance-id');
    if (idx === -1) {
      sec.content_array.push({ id: 'temp-instance-id', instance_description: '', content: contentHtml, favorite: false, sources: [] });
      idx = sec.content_array.length - 1;
    } else {
      sec.content_array[idx].content = contentHtml;
    }
    sec.selected_instance_id = 'temp-instance-id';
    this.currentInstancePosition = idx;
  }

  private buildHeaderHtml(role: string, topic: string, dateCivilian: string) {
    const safeTopic = topic && topic.trim().length ? topic.trim() : 'Topic';
    return `${role}<br><span style="color:#1266f1;font-weight:600">${safeTopic} (Speaking Engagement)</span><br>${dateCivilian}`;
    // Director/Vice Director in black, topic in blue; date on its own line.
  }

  private speakingBlocks() {
    return [
      { name: 'Header', content: this.buildHeaderHtml(this.headerRole, this.headerTopic, this.formatCivilianDate(this.headerDate)) },
      { name: 'Background', content: `Background: A brief description of the engagement and who is in the audience.` },
      {
        name: 'Key Discussion Points',
        content: [
          `Key Discussion Points:`,
          `&#8226; Discussion point # 1`,
          `&nbsp;&nbsp;o <strong>May bold important parts for emphasis.</strong>`,
          `&nbsp;&nbsp;o Spacing between bullets should be 3pt.`,
          `&nbsp;&nbsp;o Spacing between sections should be 6pt.`,
          `&#8226; Discussion point # 2`,
          `&#8226; Solid bullet`,
          `&nbsp;&nbsp;o Hollow bullet`,
          `&nbsp;&nbsp;&nbsp;&nbsp;&#9633; Square bullet`,
          `&#8226; Discussion point # 3`,
          `&nbsp;&nbsp;o Maintain Arial 11pt or 12pt throughout the document.`,
          `&#8226; Page numbers should identify the page number including the total number of pages.`,
          `&nbsp;&nbsp;o When consolidating several topics, ensure the numbers correspond with the topic not the entire file (i.e., 1 of 3 not 1 of 20).`,
          `&#8226; Discussion point # 4`,
          `&nbsp;&nbsp;o The header should remain with the topic in blue with the Director or Vice Director and dates in black.`,
          `&#8226; Discussion point # 5`,
          `&nbsp;&nbsp;o The dates are civilian formatting month, day, year.`,
          `&nbsp;&nbsp;o Always use Oxford comma rules.`
        ].join('<br>')
      },
      {
        name: 'Closing Discussion Point',
        content: [
          `Closing Discussion Point:`,
          `&#8226; Final, “get off the stage” point.`
        ].join('<br>')
      },
      {
        name: 'Topics to Avoid',
        content: [
          `Topics to Avoid:`,
          `&#8226; Topic # 1 to avoid`
        ].join('<br>')
      }
    ];
  }

  private applySpeakingNotesIfNeeded() {
    if (this.speakingNotesApplied || !this.report?.sections) return;

    const existingNames = new Set(
      (this.report.sections || []).map((s: any) => (s.section_name || '').toLowerCase())
    );

    // Order: Background, Key Discussion Points, Closing Discussion Point, Topics to Avoid (Header is separate)
    const blocks = this.speakingBlocks();
    const needed = blocks.filter(b => !existingNames.has(b.name.toLowerCase()));
    if (needed.length === 0) {
      this.speakingNotesApplied = true;
      return;
    }

    const insertNext = (idx: number) => {
      if (idx >= needed.length) {
        this.speakingNotesApplied = true;
        this.cdr.detectChanges();
        return;
      }
      const blk = needed[idx];
      this.downloadReportAvailable = false;
      const destinationIndex = this.report.sections.length; // append

      this.reportsService.createSection(this.reportId, this.report, destinationIndex).subscribe((res: any) => {
        this.report = res.db_report;
        const newSectionId = res.section_id;
        const secIndex = this.report.sections.findIndex((s: any) => s.id === newSectionId);
        const sec = this.report.sections[secIndex];

        sec.section_name = blk.name;
        if (!sec.section_settings) {
          sec.section_settings = { section_type: 'text_block', section_description: '', tone: 'Professional', structure: '' };
        }

        this.ensureTempInstanceWithContent(secIndex, blk.content);

        this.reportsService.updateSection(this.report.id, sec.id, sec).subscribe(() => {
          this.downloadReportAvailable = true;
          insertNext(idx + 1);
        });
      });
    };

    insertNext(0);
  }

  onHeaderControlChange() {
    if (!this.speakingModeActive) return;
    if (this.headerDebounce) clearTimeout(this.headerDebounce);
    this.headerDebounce = setTimeout(() => this.applySpeakingHeaderFromControls(), 300);
  }

  applySpeakingHeaderFromControls() {
    if (!this.speakingModeActive || !this.report?.sections) return;
    const headerIdx = this.report.sections.findIndex((s: any) => (s.section_name || '').toLowerCase() === 'header');
    if (headerIdx === -1) return;

    const md = this.buildHeaderHtml(this.headerRole, this.headerTopic, this.formatCivilianDate(this.headerDate));
    this.ensureTempInstanceWithContent(headerIdx, md);
    const sec = this.report.sections[headerIdx];
    this.reportsService.updateSection(this.report.id, sec.id, sec).subscribe(() => this.cdr.detectChanges());
  }

  // ---------------- end speaking_notes helpers ----------------
}
