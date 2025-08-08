updateSectionType(section, type){
  console.log(section, type);

  this.selectedSectionType = type;
  section.section_settings = type;

  // NEW: only act on the special templates
  if (type?.section_type === 'speaking_notes') {
    this.applySpeakingNotesTemplateOnce();
  }
  if (type?.section_type === 'information_paper') {
    // reserved for later wiring; no-op now
  }
}
