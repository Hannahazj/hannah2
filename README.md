{
  "section_type": "information_paper",
  "section_description": "DLA Information Paper format",
  "tone": "neutral, clear, direct",
  "structure": "Date, Subject, Purpose, Background, Discussion/Key Points (bulleted), Recommendation/Way Ahead, Prepared by, Approved by",
  "template_markdown": "## Information Paper\n\n{{date}}\n\n**SUBJECT:** Clearly and succinctly specify the issue the paper discusses. Use specific description that summarizes the content, avoiding vague, one-word subjects. Clarifying the subject can help in organizing and presenting the most relevant information clearly. Do not introduce acronyms in the subject line. (1-2 lines)\n\n**PURPOSE:** State what this information paper seeks to do (1 sentence)\n\n**BACKGROUND:** Clearly state germane background information on the issue. (3-4 sentences max)\n\n**DISCUSSION / KEY POINTS:**\n1. Clearly and succinctly present information that the reader needs to know about the subject. Explain why it is important for the recipient to have this information.\n2. Structure main points and supporting ideas in complete but succinct bulleted paragraphs. The bullets indicate divisions and relationships among concepts.\n   - Use sub-bullets to illustrate significant supporting ideas that expand on the main bullet paragraph.\n3. The organization of information should flow from the subject, audience, and purpose. Organize the information by presenting the most important information first unless information is necessary for the reader to understand the main point. Each bulleted paragraph should logically flow to the next.\n4. Use short, concise sentences in an active voice. The tone should be neutral, clear, and direct in nature. Limit sentences to one thought. Use short, simple words. Avoid using acronyms, abbreviations, and jargon.\n\n**RECOMMENDATION / WAY AHEAD:** Clearly state recommendations and the way ahead.\n\n---\n\n*Prepared by:* Full name, e-mail, and phone number of action officer  \n*Approved by:* Full name of approving official (O-6/GS-15+ for actions to DLA Director)\n"
}


// near the top of report-gen.component.ts
interface SectionType {
  section_type: string;
  section_description?: string;
  tone?: string;
  structure?: string;
  template_markdown?: string; // <-- new
}

sectionTypes: SectionType[] = [];
selectedSectionType: SectionType | null = null;


private ensureTempInstanceWithContent(content: string, sources: any[] = []) {
  const sec = this.report.sections[this.currentSection];
  const exists = sec.content_array.findIndex(i => i.id === 'temp-instance-id');

  if (exists === -1) {
    sec.content_array.push({
      id: 'temp-instance-id',
      instance_description: '',
      content,
      favorite: false,
      sources
    });
  } else {
    sec.content_array[exists].content = content;
  }

  sec.selected_instance_id = 'temp-instance-id';
  this.currentInstancePosition = sec.content_array.findIndex(i => i.id === 'temp-instance-id');
  this.cdr.detectChanges();
}


updateSectionType(section, type: SectionType){
  this.selectedSectionType = type;
  section.section_settings = type;

  // Optionally set the prompt to guide generation
  if (type.section_type === 'information_paper') {
    this.queryString = `Format the response as an Information Paper. Use the exact headings and guidance. Date line should be formatted as 'MMMM DD, YYYY'. Avoid acronyms, use neutral, clear, direct tone.`;
  }
}


<div class="form-row" *ngIf="selectedSectionType?.template_markdown">
  <button
    type="button"
    class="btn btn-outline-secondary settings-dropdown"
    (click)="insertSelectedTemplate()"
  >
    Insert template into this section
  </button>
</div>


insertSelectedTemplate(){
  if (!this.selectedSectionType?.template_markdown) return;

  // Replace {{date}} with today in the template
  const today = new Date().toLocaleDateString(undefined, { month: 'long', day: '2-digit', year: 'numeric' });
  const md = this.selectedSectionType.template_markdown.replace('{{date}}', today);

  this.ensureTempInstanceWithContent(md);

  // Enter "new instance" mode so the user can submit against this scaffold
  this.createNewInstanceMode = true;
}


