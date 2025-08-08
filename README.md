// --- speaking notes only ---
speakingNotesApplied = false;       // prevents duplicate insertion
headerRole = 'Director';            // UI fields (no new reactive form => no nested form issues)
headerTopic = '';
headerDate: string | null = null;   // "YYYY-MM-DD" or null (auto today)


private formatCivilianDate(d?: string | null) {
  if (!d) return new Date().toLocaleDateString(undefined, { month: 'long', day: '2-digit', year: 'numeric' });
  // handle <input type="date">
  const parts = d.split('-');
  if (parts.length === 3) {
    const dt = new Date(+parts[0], +parts[1]-1, +parts[2]);
    return dt.toLocaleDateString(undefined, { month: 'long', day: '2-digit', year: 'numeric' });
  }
  return d;
}

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
  // Order: Background, Key Discussion Points, Closing Discussion Point, Topics to Avoid
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
  <!-- Add more as needed -->
</ul>
</div>`;

  return [
    { name: 'Background', content: `<div class="speaking-notes">Background: A brief description of the engagement and who is in the audience.</div>` },
    { name: 'Key Discussion Points', content: keyPointsHTML },
    { name: 'Closing Discussion Point', content: closingHTML },
    { name: 'Topics to Avoid', content: avoidHTML },
  ];
}
