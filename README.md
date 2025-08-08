private async applySpeakingNotesTemplateOnce() {
  if (this.speakingNotesApplied || !this.report) return;

  const role = this.headerRole || 'Director';
  const topic = this.headerTopic || 'Topic';
  const dateCivilian = this.formatCivilianDate(this.headerDate);

  // Insert starting after the current section
  let insertAt = (this.currentSection ?? this.report.sections.length - 1) + 1;

  // 1) Header first (editable via the small controls below)
  {
    this.downloadReportAvailable = false;
    const res: any = await new Promise(r => this.reportsService.createSection(this.reportId, this.report, insertAt).subscribe(r));
    this.report = res.db_report;

    const newSectionId = res.section_id;
    const i = this.report.sections.findIndex(s => s.id === newSectionId);
    const sec = this.report.sections[i];
    sec.section_name = 'Header';
    sec.section_settings = { section_type: 'text_block', section_description: '', tone: 'Professional', structure: '' };

    this.ensureTempInstanceWithContent(i, this.buildSpeakingHeader(role, topic, dateCivilian));
    await new Promise(r => this.reportsService.updateSection(this.report.id, sec.id, sec).subscribe(r));

    insertAt++;
    this.downloadReportAvailable = true;
  }

  // 2) Then the required order sections
  const blocks = this.buildSpeakingSections();
  for (const b of blocks) {
    this.downloadReportAvailable = false;
    const res: any = await new Promise(r => this.reportsService.createSection(this.reportId, this.report, insertAt).subscribe(r));
    this.report = res.db_report;

    const newSectionId = res.section_id;
    const i = this.report.sections.findIndex(s => s.id === newSectionId);
    const sec = this.report.sections[i];

    sec.section_name = b.name;
    sec.section_settings = { section_type: 'text_block', section_description: '', tone: 'Professional', structure: '' };
    this.ensureTempInstanceWithContent(i, b.content);

    await new Promise(r => this.reportsService.updateSection(this.report.id, sec.id, sec).subscribe(r));
    insertAt++;
    this.downloadReportAvailable = true;
  }

  this.speakingNotesApplied = true;
  this.selectSection((this.currentSection ?? -1) + 1, null); // jump to Header
  this.cdr.detectChanges();
}

applySpeakingHeaderFromControls() {
  const i = this.report.sections.findIndex(s => (s.section_name || '').toLowerCase() === 'header');
  if (i === -1) return;
  const role = this.headerRole || 'Director';
  const topic = this.headerTopic || 'Topic';
  const dateCivilian = this.formatCivilianDate(this.headerDate);

  const html = this.buildSpeakingHeader(role, topic, dateCivilian);
  this.ensureTempInstanceWithContent(i, html);
  this.reportsService.updateSection(this.report.id, this.report.sections[i].id, this.report.sections[i])
    .subscribe(() => this.cdr.detectChanges());
}
