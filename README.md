{
  "id": "tpl-speaking-notes",
  "template_name": "Speaking Engagement Brief",
  "description": "Director/Vice Director speaking notes with formatted header and nested bullets.",
  "sections": [
    {
      "id": "tpl-speaking-header",
      "section_name": "Header",
      "template_id": "tpl-speaking-notes",
      "section_description": "Role, topic (blue), date.",
      "tone": "Neutral, clear, direct",
      "structure": "Three lines",
      "section_type": "text_block",
      "content": "DIRECTOR\n<span class=\"topic-blue\">{{topic}} (Speaking Engagement)</span>\n{{date}}"
    },
    {
      "id": "tpl-speaking-background",
      "section_name": "Background",
      "template_id": "tpl-speaking-notes",
      "section_description": "Brief engagement description and audience.",
      "tone": "Neutral, clear, direct",
      "structure": "Short paragraph",
      "section_type": "text_block",
      "content": "Background: A brief description of the engagement and who is in the audience."
    },
    {
      "id": "tpl-speaking-keypoints",
      "section_name": "Key Discussion Points",
      "template_id": "tpl-speaking-notes",
      "section_description": "Main points with specific spacing/font rules.",
      "tone": "Neutral, clear, direct",
      "structure": "Nested bullets",
      "section_type": "text_block",
      "content": "Key Discussion Points:\n• Discussion point #1\n  o <strong>May bold important parts for emphasis.</strong>\n  o Spacing between bullets should be 3pt.\n  o Spacing between sections should be 6pt.\n• Discussion point #2\n• ● Solid bullet\n  o ○ Hollow bullet\n     ▪ Square bullet\n• Discussion point #3\n  o Maintain Arial 11pt or 12pt throughout the document.\n• Page numbers should identify the page number including the total number of pages.\n  o When consolidating several topics, number per topic (e.g., 1 of 3, not 1 of 20).\n• Discussion point #4\n  o The header keeps the topic in blue; role and dates in black.\n• Discussion point #5\n  o Dates use civilian formatting (Month DD, YYYY).\n  o Always use Oxford comma rules."
    },
    {
      "id": "tpl-speaking-closing",
      "section_name": "Closing Discussion Point",
      "template_id": "tpl-speaking-notes",
      "section_description": "Final takeaway.",
      "tone": "Neutral, clear, direct",
      "structure": "Single bullet",
      "section_type": "text_block",
      "content": "Closing Discussion Point:\n• Final, “get off the stage” point."
    },
    {
      "id": "tpl-speaking-avoid",
      "section_name": "Topics to Avoid",
      "template_id": "tpl-speaking-notes",
      "section_description": "Items to avoid mentioning.",
      "tone": "Neutral",
      "structure": "Bulleted",
      "section_type": "text_block",
      "content": "Topics to Avoid:\n• Topic #1 to avoid\n• Topic #2 to avoid"
    }
  ]
}
