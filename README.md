private formatCivilianDate(d?: string | null) {
  if (!d) {
    return new Date().toLocaleDateString(undefined, { month: 'long', day: '2-digit', year: 'numeric' });
  }
  const parts = d.split('-');
  if (parts.length === 3) {
    const dt = new Date(+parts[0], +parts[1] - 1, +parts[2]);
    return dt.toLocaleDateString(undefined, { month: 'long', day: '2-digit', year: 'numeric' });
  }
  return d;
}

// upsert guard so TS/RT never balks at section_settings
private upsertSectionSettings(sec: any, partial: {
  section_type?: string;
  section_description?: string;
  tone?: string;
  structure?: string;
}) {
  if (!sec.section_settings) {
    sec.section_settings = { section_type: 'text_block', section_description: '', tone: '', structure: '' };
  }
  Object.assign(sec.section_settings, partial);
}

// reuse your existing temp instance pattern
private ensureTempInstanceWithContent(sectionIndex: number, content: string) {
  const sec = this.report.sections[sectionIndex];
  let idx = sec.content_array.findIndex(i => i.id === 'temp-instance-id');
  if (idx === -1) {
    sec.content_array.push({ id: 'temp-instance-id', instance_description: '', content, favorite: false, sources: [] });
    idx = sec.content_array.length - 1;
  } else {
    sec.content_array[idx].content = content;
  }
  sec.selected_instance_id = 'temp-instance-id';
  this.currentInstancePosition = idx;
}

private buildSpeakingHeader(role: string, topic: string, dateCivilian: string) {
  return `<div class="speaking-notes">
${role}
<span class="topic-blue">${topic} (Speaking Engagement)</span>
${dateCivilian}
</div>`;
}

private buildSpeakingSections() {
  const keyPointsHTML = `<div class="speaking-notes">
Key Discussion Points:
<ul>
  <li>Discussion point # 1
    <ul>
      <li><strong>May bold important parts for emphasis.</strong></li>
      <li>Spacing between bullets should be 3pt.</li>
      <li>Spacing between sections should be 6pt.</li>
    </ul>
  </li>
  <li>Discussion point # 2</li>
  <li>● Solid bullet</li>
  <li>○ Hollow bullet
    <ul><li>▪ Square bullet</li></ul>
  </li>
  <li>Discussion point # 3
    <ul><li>Maintain Arial 11pt or 12pt throughout the document.</li></ul>
  </li>
  <li>Page numbers should identify the page number including the total number of pages.</li>
  <li>When consolidating several topics, ensure the numbers correspond with the topic not the entire file (i.e., 1 of 3 not 1 of 20).</li>
  <li>Discussion point # 4
    <ul><li>The header should remain with the topic in blue with the Director or Vice Director and dates in black.</li></ul>
  </li>
  <li>Discussion point # 5
    <ul>
      <li>The dates are civilian formatting month, day, year.</li>
      <li>Always use Oxford comma rules.</li>
    </ul>
  </li>
</ul>
</div>`;

  const closingHTML = `<div class="speaking-notes">
Closing Discussion Point:
<ul><li>Final, “get off the stage” point.</li></ul>
</div>`;

  const avoidHTML = `<div class="speaking-notes">
Topics to Avoid:
<ul>
  <li>Topic # 1 to avoid</li>
</ul>
</div>`;

  // Order: Background → Key Discussion Points → Closing Discussion Point → Topics to Avoid
  return [
    { name: 'Background', content: `<div class="speaking-notes">Background: A brief description of the engagement and who is in the audience.</div>` },
    { name: 'Key Discussion Points', content: keyPointsHTML },
    { name: 'Closing Discussion Point', content: closingHTML },
    { name: 'Topics to Avoid', content: avoidHTML },
  ];
}

private async applySpeakingNotesTemplateOnce() {
  if (this.speakingNotesApplied || !this.report) return;

  const role = this.headerRole || 'Director';
  const topic = this.headerTopic || 'Topic';
  const dateCivilian = this.formatCivilianDate(this.headerDate);

  // Start inserting right after the current section
  let insertAt = (this.currentSection ?? this.report.sections.length - 1) + 1;

  // 1) Header
  {
    this.downloadReportAvailable = false;
    const res: any = await new Promise(r => this.reportsService.createSection(this.reportId, this.report, insertAt).subscribe(r));
    this.report = res.db_report;

    const newSectionId = res.section_id;
    const i = this.report.sections.findIndex(s => s.id === newSectionId);
    const sec = this.report.sections[i];

    sec.section_name = 'Header';
    this.upsertSectionSettings(sec, { section_type: 'text_block', section_description: '', tone: 'Professional', structure: '' });
    this.ensureTempInstanceWithContent(i, this.buildSpeakingHeader(role, topic, dateCivilian));

    await new Promise(r => this.reportsService.updateSection(this.report.id, sec.id, sec).subscribe(r));
    insertAt++;
    this.downloadReportAvailable = true;
  }

  // 2) Then required order blocks
  for (const b of this.buildSpeakingSections()) {
    this.downloadReportAvailable = false;
    const res: any = await new Promise(r => this.reportsService.createSection(this.reportId, this.report, insertAt).subscribe(r));
    this.report = res.db_report;

    const newSectionId = res.section_id;
    const i = this.report.sections.findIndex(s => s.id === newSectionId);
    const sec = this.report.sections[i];

    sec.section_name = b.name;
    this.upsertSectionSettings(sec, { section_type: 'text_block', section_description: '', tone: 'Professional', structure: '' });
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
