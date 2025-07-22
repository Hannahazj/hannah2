report_db.py "import uuid
from typing import Optional

from internal.report_gen.report_db_setup import (
    ReportDetails,
    ReportSection,
    ReportTemplateKeyTable,
    ReportUsersKeyTable,
    SectionInstanceSource,
    SectionResponseInstance,
    TemplateDetails,
    TemplateSection,
)
from internal.report_gen.report_models import (
    ReportDetailsSchema,
    ReportSectionSchema,
    SectionResponseInstanceSchema,
)
from internal.report_gen.template_models import (
    TemplateDetailsSchema,
    TemplateSectionSchema,
)
from sqlalchemy import desc
from sqlalchemy.orm import Session


def get_reports(
    db: Session, skip: int = 0, limit: int = 10, user_id: Optional[str] = None
):
    if user_id:
        # search document_ownership table for rows where user_id = user_id and
        all_reports_in_table = db.query(ReportUsersKeyTable.report_id).all()
        print(all_reports_in_table)
        report_ids = (
            db.query(ReportUsersKeyTable.report_id)
            .filter(ReportUsersKeyTable.user_id == user_id)
            .all()
        )
        report_ids = [r[0] for r in report_ids]
        print(report_ids)
        # # Step 3: Get the list of report details for all reports belonging to the user
        return db.query(ReportDetails).filter(ReportDetails.id.in_(report_ids)).all()
        # pass
    # return db.query((ReportDetails.offset(skip).limit(limit).all(), )
    return db.query(ReportDetails).offset(skip).limit(limit).all()


def get_report(db: Session, report_id: str):
    return (
        db.query(ReportDetails).filter(ReportDetails.id == uuid.UUID(report_id)).first()
    )


def insert_user_report_access(db: Session, user_id, report_id, access_type="owner"):
    try:
        # Create a new UserReportAccess object
        new_access = ReportUsersKeyTable(
            user_id=user_id, report_id=report_id, access_level=access_type
        )

        # Add the new entry to the session
        db.add(new_access)

        # Commit the session to save the entry to the database
        db.commit()
        print("New access entry added successfully.")
        return new_access

    except Exception as e:
        print(f"An error occurred: {e}")
        db.rollback()
        return None

    # finally:
    # db.close()


def create_report(
    db: Session, report: ReportDetailsSchema, user_id: Optional[str] = None
):
    if not report.report_details:
        report.report_details = {}
    db_report = ReportDetails(
        id=uuid.uuid4(),
        report_name=report.report_name,
        sections=report.sections,  # , report_id=report.report_id
        report_details=report.report_details,
    )
    db.add(db_report)
    db.commit()

    if user_id:
        insert_user_report_access(db=db, user_id=user_id, report_id=db_report.id)
        # pass
    db.refresh(db_report)
    return db_report


def delete_report(db: Session, report_id: str):
    db_report = get_report(db, report_id)
    if db_report:
        db.query(ReportUsersKeyTable).filter(
            ReportUsersKeyTable.report_id == report_id
        ).delete()
        db.delete(db_report)
        db.commit()
        return True
    return False


def update_report(db: Session, report_id: str, report_update: ReportDetailsSchema):
    db_report = get_report(db, report_id)
    if db_report:
        if report_update.report_name:
            db_report.report_name = report_update.report_name
        if report_update.sections:
            # updated_sections = []
            db_report.sections = []  # updated_sections
            for section in report_update.sections:
                # update_section(db, report_id, str(section.id.hex), section)
                # if section exists, get it
                # print("looking for:", str(section.id))
                # db_section = get_section(db, report_id, str(section.id))
                # print("found: ", db_section)
                # if db_section:
                #     db_report.add(db_section)
                # else:
                add_section_to_report(db, report_id, section, use_old_id=True)
                # if it doesnt exist, create it
                db.commit()
                db.refresh(db_report)
                # updated_sections.append(section)

        # if report_update.sections:
        #     db_report.sections = report_update.sections
        db.commit()
        return db_report
    return None


# Get sections
def get_sections(db: Session, report_id: str, skip: int = 0, limit: int = 10):
    db_report = get_report(db=db, report_id=report_id)
    return db_report.sections


# Get section
def get_section(
    db: Session, report_id: str, section_id: str, last_section: bool = False
):
    # if last_section:
    #     return (
    #         db.query(ReportSection)
    #         .filter(
    #             ReportSection.report_id == uuid.UUID(report_id),
    #             ReportSection.id == uuid.UUID(section_id),
    #         )
    #         .order_by(desc(ReportSection.order_number))
    #         .first()
    #     )
    return (
        db.query(ReportSection)
        .filter(
            ReportSection.report_id == uuid.UUID(report_id),
            ReportSection.id == uuid.UUID(section_id),
        )
        .first()
    )


def add_section_to_report(
    db: Session, report_id: str, section: ReportSectionSchema, use_old_id: bool = False
):
    db_report = get_report(db, report_id)
    if db_report:
        # TODO: get max number of section order and use that to create a new order number
        # last_section = get_section(db, report_id, section.id, last_section=True)
        # if not included an order number
        # order_number = last_section.order_number + 1
        # else has a section number and now we need to reorder all other sections
        # for i in range(section.order_number, last_section.order_number + 1):
        #
        # section.order_number
        section_id = uuid.uuid4()
        if use_old_id:
            section_id = section.id
        if not section.section_settings:
            section.section_settings = {}

        new_section = ReportSection(
            id=section_id,  # uuid.uuid4(),
            section_name=section.section_name,
            locked=section.locked,
            selected_instance_id=section.selected_instance_id,
            report_id=db_report.id,
            # report=db_report,
            section_settings=section.section_settings,
            content_array=[],  # section.content_array,
        )
        db.add(new_section)
        db.commit()
        db.refresh(new_section)
        for instance in section.content_array:
            add_instance_to_section(
                db, report_id, str(new_section.id), instance, use_old_id=True
            )
        db.commit()
        db.refresh(new_section)
        return db_report, section_id
    return None


def update_section(
    db: Session, report_id: str, section_id: str, section_update: ReportSectionSchema
):
    # db_report = get_report(db, report_id)
    # if db_report:
    db_section = get_section(db=db, report_id=report_id, section_id=section_id)
    if db_section:
        db_section.section_name = section_update.section_name or db_section.section_name
        db_section.locked = section_update.locked  # or db_section.locked
        # db_section.order_number = section_update.order_number or db_section.order_number
        db_section.selected_instance_id = (
            section_update.selected_instance_id or db_section.selected_instance_id
        )
        db_section.content_array = (
            section_update.content_array or db_section.content_array
        )
        db.commit()
        return db_section
    return None


def delete_section(db: Session, report_id: str, section_id: str):
    db_section = get_section(db=db, report_id=report_id, section_id=section_id)
    if db_section:
        db.delete(db_section)
        db.commit()
        return True
    return False


# Get
def get_instance(db: Session, report_id: str, section_id: str, instance_id: str):
    return (
        db.query(SectionResponseInstance)
        .filter(
            SectionResponseInstance.id == uuid.UUID(instance_id),
            SectionResponseInstance.report_section_id == uuid.UUID(section_id),
        )
        .first()
    )


import pandas as pd
from routers.rag import create_citations, doc_section_gen, search_help


def gen_section(
    instance: SectionResponseInstanceSchema, db_section: ReportSectionSchema
):
    # db_section = get_section
    if instance.sources:
        # create filter for ai search for sources here.
        # source_filter = "filenamed filter"
        # results = []
        # source_filter = ""
        source_filter = ", ".join(
            [f"{source_file['file_name']}" for source_file in instance.sources]
        )
        source_filter = f"search.in(file_name, '{source_filter}', ',')"
        # for file in instance.sources:
        #     filter_string = f"search.in(file_name, '{filename}', ',')"
        #     results_for_file = search_assist(filter_string)
        #     results.extend(results_for_file)
    else:
        source_filter = None
    if not instance.content:
        # AI Search search
        search_results = search_help(
            top=6,
            query_text=instance.instance_description,
            use_text_search=False,
            use_vector_search=True,
            minimum_search_score=0.0,
            filter_string_test=source_filter,
        )

        context = "\n\n".join(
            [
                "Snippet from page "
                + str(file["pages"])
                + " of file:"
                + file["file_name"]
                + "also called:"
                + file["title"]
                + "\n"
                + file["content"]
                for file in search_results
            ]
        )
        citations = create_citations(search_results=search_results)
        instance.citations = citations
        sources = [
            {
                "title": citation["title"],
                "file_name": citation["file_name"],
                "source_path": citation["source_path"],
            }
            for citation in citations
        ]
        df = pd.DataFrame(sources).drop_duplicates().to_dict(orient="records")
        instance.sources = df  # set(sources)
        # TODO: get section_id from instance, and then pass section details to doc_section_gen
        # generate content with LLMs
        instance.content = "Generated section by LLM"
        instance.content = doc_section_gen(
            instance.instance_description,
            context=context,
            section_settings=db_section.section_settings,
        )
        return instance
    return instance


# Create, Retrieve, Update, Delete for section instances
def add_instance_to_section(
    db: Session,
    report_id: str,
    section_id: str,
    instance: SectionResponseInstanceSchema,
    use_old_id: bool = False,
):
    db_section = get_section(db, report_id, section_id)
    if not instance.content:
        instance = gen_section(instance, db_section)
    if db_section:
        print(instance.citations)
        print(instance.content)
        instance_id = uuid.uuid4()
        if use_old_id:
            instance_id = instance.id
        new_instance = SectionResponseInstance(
            id=instance_id,  # uuid.uuid4(),
            instance_description=instance.instance_description,
            content=instance.content,
            favorite=instance.favorite,
            sources=instance.sources,
            citations=instance.citations,
            # report_id=db_section.report_id,
            report_section_id=db_section.id,
        )
        db.add(new_instance)
        db.commit()
        db.refresh(new_instance)
        # TODO: section needs to be updated with the selected_instance_id
        db_section = get_section(db=db, report_id=report_id, section_id=section_id)
        db_section.selected_instance_id = str(instance_id)
        db.commit()
        db.refresh(db_section)
        return db_section, instance_id, new_instance
    return None


def update_section_instance(
    db: Session,
    report_id: str,
    section_id: str,
    instance_id: str,
    updated_instance: SectionResponseInstanceSchema,
):
    db_instance = get_instance(db, report_id, section_id, instance_id)
    # db_section = get_section
    db_section = get_section(db=db, report_id=report_id, section_id=section_id)
    updated_instance = gen_section(updated_instance, db_section)
    if db_instance:
        db_instance.instance_description = (
            updated_instance.instance_description or db_instance.instance_description
        )
        db_instance.content = updated_instance.content or db_instance.content
        db_instance.favorite = updated_instance.favorite or db_instance.favorite
        db_instance.sources = updated_instance.sources or db_instance.sources
        db_instance.citations = updated_instance.citations or db_instance.citations
        db.commit()
        return db_instance
    return None


def delete_instance(db: Session, report_id: str, section_id: str, instance_id: str):
    db_instance = get_instance(db, report_id, section_id, instance_id)
    if db_instance:
        db.delete(db_instance)
        db.commit()
        return True
    return False


# from docx import Document
# from docx.shared import Pt, Inches, RGBColor
# import re

# def create_doc(db, report_id):
#     document = Document()

#     # Set margins to 1 inch
#     for section in document.sections:
#         section.top_margin = Inches(1)
#         section.bottom_margin = Inches(1)
#         section.left_margin = Inches(1)
#         section.right_margin = Inches(1)

#     db_report = get_report(db, report_id)
#     works_cited = []
#     if db_report:

#         if hasattr(db_report, 'report_name'):
#             title = document.add_heading(level=0)  # level 0 is for the main title
#             title_run = title.add_run(db_report.report_name)
#             title_run.font.name = 'Times New Roman'
#             title_run.font.size = Pt(14)  # slightly larger for title
#             title_run.bold = True
#             title_run.font.color.rgb = RGBColor(0, 0, 0)

#             document.add_paragraph()

#         sections = get_sections(db, report_id, limit=10000)
#         if sections:
#             section_number = 1

#             # Regex pattern for section number (e.g., a., b., etc.)
#             subsection_pattern = re.compile(r'^[a-z]\.')

#             # Regex pattern for point number (e.g., (1), (2), etc.)
#             point_pattern = re.compile(r'^\(\d+\)')

#             for section in sections:
#                 if section.selected_instance_id:
#                     instance = get_instance(
#                         db,
#                         report_id,
#                         str(section.id.hex),
#                         str(section.selected_instance_id.hex),
#                     )
#                     if instance:
#                         if section.section_name:
#                             # Add the section header with specific styling
#                             heading = document.add_heading(level=1)

#                             # Add the section number without bold
#                             run_number = heading.add_run(f"{section_number}. ")
#                             run_number.font.name = 'Times New Roman'
#                             run_number.font.size = Pt(12)
#                             run_number.bold = False  # Section number not bold
#                             run_number.font.color.rgb = RGBColor(0, 0, 0)  # Black color

#                             # Add the section name with colon
#                             run_name = heading.add_run(f"{section.section_name}: ")
#                             run_name.font.name = 'Times New Roman'
#                             run_name.font.size = Pt(12)
#                             run_name.bold = True  # Section name bold
#                             run_name.font.color.rgb = RGBColor(0, 0, 0)  # Black color

#                             if instance.content:
#                                 # Split the content into lines
#                                 lines = instance.content.split('\n')

#                                 # Add first line directly after the colon, in the same paragraph
#                                 if lines:
#                                     first_line = lines[0]
#                                     run_content_first_line = heading.add_run(first_line)
#                                     run_content_first_line.font.name = 'Times New Roman'
#                                     run_content_first_line.font.size = Pt(12)
#                                     run_content_first_line.bold = False
#                                     run_content_first_line.font.color.rgb = RGBColor(0, 0, 0)


#                                 for line in lines[1:]:

#                                     para = document.add_paragraph()
#                                     para.style = document.styles['Normal']
#                                     para_format = para.paragraph_format
#                                     para_format.space_before = Pt(0)
#                                     para_format.space_after = Pt(4)

#                                     # Set indentation based on line type
#                                     if subsection_pattern.match(line):
#                                         # Subsections (a., b., etc.) aligned with section title
#                                         para.left_indent = Inches(0.4)
#                                         run = para.add_run(line)
#                                         run.bold = True
#                                     elif point_pattern.match(line):
#                                         # Points ((1), (2), etc.) indented further
#                                         para.left_indent = Inches(0.8)
#                                         run = para.add_run(line)
#                                         run.bold = False
#                                     else:
#                                         # Regular text aligned with points
#                                         para.left_indent = Inches(0.8)
#                                         run = para.add_run(line)
#                                         run.bold = False

#                                     # Apply common formatting
#                                     run.font.name = 'Times New Roman'
#                                     run.font.size = Pt(12)
#                                     run.font.color.rgb = RGBColor(0, 0, 0)

#                             section_number += 1

#                         works_cited.extend(instance.sources)  # Assuming sources is a list of citations

#         document.save(f"{report_id}.docx")
#         return True

#     return False

from docx import Document
from models.DocumentTemplate import DocumentTemplate


def create_doc(db, report_id: str, template_name: str = "default") -> bool:
    try:
        document = Document()
        template = DocumentTemplate(template_name)

        # Apply margins
        template.apply_margins(document)

        db_report = get_report(db, report_id)
        works_cited = []

        if not db_report:
            return False

        # Add title if exists
        if hasattr(db_report, "report_name"):
            template.add_title(document, db_report.report_name)

        # Process sections
        sections = get_sections(db, report_id, limit=10000)
        if sections:
            section_number = 1

            for section in sections:
                if section.selected_instance_id:
                    instance = get_instance(
                        db,
                        report_id,
                        str(section.id.hex),
                        str(section.selected_instance_id.hex),
                    )

                    if instance and section.section_name:
                        # Get first line of content if it exists
                        first_line = ""
                        remaining_lines = []

                        if instance.content:
                            lines = instance.content.split("\n")
                            if lines:
                                first_line = lines[0]
                                remaining_lines = lines[1:]

                        # Add section header with first line
                        template.add_section_header(
                            document, section_number, section.section_name, first_line
                        )

                        # Add remaining content
                        for line in remaining_lines:
                            template.add_content_line(document, line)

                        section_number += 1

                        if instance.sources:
                            works_cited.extend(instance.sources)

        document.save(f"{report_id}.docx")
        return True

    except Exception as e:
        print(f"Error creating document: {str(e)}")
        return False


import pypandoc

# from docx2pdf import convert
# from spire.doc import Document
# from spire.doc.common import FileFormat


def convert_to_pdf(report_id):
    try:
        # pypandoc.convert_file(
        #     f"{report_id}.docx", format="docx", to="pdf", outputfile=f"{report_id}.pdf"
        # )
        # convert(f"{report_id}.docx", f"{report_id}_docx2pdf.pdf")

        # Create word document
        # document = Document()

        # # Load a doc or docx file
        # document.LoadFromFile(f"{report_id}.docx")

        # # Save the document to PDF
        # document.SaveToFile(f"{report_id}_spire.pdf", FileFormat.PDF)
        # document.Close()
        pypandoc.convert_file(
            f"{report_id}.docx",
            "latex",
            outputfile=f"{report_id}_pandoc.pdf",  # format="docx", to="pdf", outputfile=f"{report_id}.pdf"
        )
        
        # pypandoc.convert_file(
        #     f"{report_id}.docx",
        #     to="pdf",
        #     outputfile=f"{report_id}_pandoc.pdf",  # format="docx", to="pdf", outputfile=f"{report_id}.pdf"
        # )
        return True
    except Exception as e:
        raise


def get_templates(
    db: Session, skip: int = 0, limit: int = 10, user_id: Optional[str] = None
):
    if user_id:
        pass
        # search document_ownership table for rows where user_id = user_id and
        # all_reports_in_table = db.query(ReportUsersKeyTable.report_id).all()
        # print(all_reports_in_table)
        # report_ids = db.query(ReportUsersKeyTable.report_id).filter(ReportUsersKeyTable.user_id == user_id).all()
        # report_ids = [r[0] for r in report_ids]
        # print(report_ids)
        # # Step 3: Get the list of report details for all reports belonging to the user
        # return db.query(ReportDetails).filter(ReportDetails.id.in_(report_ids)).all()
        # pass
    # return db.query((ReportDetails.offset(skip).limit(limit).all(), )
    return db.query(TemplateDetails).offset(skip).limit(limit).all()


def get_template(db: Session, template_id: str):
    return (
        db.query(TemplateDetails)
        .filter(TemplateDetails.id == uuid.UUID(template_id))
        .first()
    )


def create_template(
    db: Session, template: TemplateDetailsSchema, user_id: Optional[str] = None
):  # TODO: create template details schema
    db_template_id = uuid.uuid4()
    db_template = TemplateDetails(
        id=db_template_id,
        # id: Optional[UUID] = Field(default_factory=uuid.uuid4)
        template_name=template.template_name,
        description=template.description,
        # sections=template.sections
    )
    db.add(db_template)
    for section in template.sections:
        db_template_section = TemplateSection(
            id=uuid.uuid4(),
            section_name=section.section_name,
            template_id=db_template_id,
            section_description=section.section_description,
            tone=section.tone,
            structure=section.structure,
            section_type=section.section_type,
            content="content",
        )
        db.add(db_template_section)

    db.commit()

    # if user_id:
    #     insert_user_template_access(db=db, user_id=user_id, template_id=db_template.id)
    #     # pass
    db.refresh(db_template)
    return db_template


def create_report_from_template(
    db: Session,
    report: ReportDetailsSchema,
    template: TemplateDetailsSchema,
    user_id: Optional[str] = None,
):
    # create report
    report_settings = {
        "report_description": template.description,
        "created_from_template": template.template_name,
        ###add export_template
        "export_template": template.export_template
    }
    report.report_details = report_settings
    report = create_report(
        db=db, report=report, user_id=user_id
    )  # TODO: add optional template details at the report level
    # for section in template, add section to report
    for section in template.sections:
        section_settings = {
            "section_name": section.section_name,
            "section_type": section.section_type,
            "section_description": section.section_description,
            "tone": section.tone,
            "structure": section.structure,
        }
        report_section = ReportSectionSchema(
            section_name=section.section_name, section_settings=section_settings
        )  # TODO: need to add optional settings json to this object
        add_section_to_report(db, str(report.id), report_section)
    # return report
    return report"  DocumentTemplate.py "from docx import Document
from docx.shared import Pt, Inches, RGBColor
from docx.text.run import Run
from docx.text.paragraph import Paragraph
import re
from typing import Dict, Any, Optional
from pathlib import Path
import json
import os

class DocumentTemplate:
    def __init__(self, template_name: str):
        self.template_name = template_name
        self.config = self._load_template_config()  # JSON loading happens here
        if not self._validate_config():
            print("Invalid template configuration, using defaults")
            self.config = self._get_default_config()
        self.subsection_pattern = re.compile(r'^[a-z]\.')
        self.point_pattern = re.compile(r'^\(\d+\)')
        
    def _load_template_config(self) -> Dict[str, Any]:
        backend_dir = Path(__file__).parent.parent
        templates_dir = backend_dir / 'templates'
        
        template_path = templates_dir / f"{self.template_name}.json"
        default_path = templates_dir / "default.json"
        
        try:
            if template_path.exists():
                with open(template_path, 'r') as f:
                    return json.load(f)
            elif default_path.exists():
                with open(default_path, 'r') as f:
                    return json.load(f)
            else:
                return self._get_default_config()
        except Exception as e:
            return self._get_default_config()

    def _get_default_config(self) -> Dict[str, Any]:
        return {
            'margins': {
                'top': 1,
                'bottom': 1,
                'left': 1,
                'right': 1
            },
            'title': {
                'font': 'Times New Roman',
                'size': 11,
                'bold': True,
                'alignment': 1,
                'color': {'r': 0, 'g': 0, 'b': 0}
            },
            'section': {
                'font': 'Times New Roman',
                'size': 12,
                'bold': False,
                'heading_bold': True,  
                'color': {'r': 0, 'g': 0, 'b': 0},
                'space_before': 0,
                'space_after': 4
            },
            'subsection': {
                'indent': 0.4,
                'bold': True,
                'font': 'Times New Roman',
                'size': 12,
                'color': {'r': 0, 'g': 0, 'b': 0}
            },
            'point': {
                'indent': 0.8,
                'bold': False,
                'font': 'Times New Roman',
                'size': 12,
                'color': {'r': 0, 'g': 0, 'b': 0}
            }
        }

    def _validate_config(self) -> bool:
        required_settings = {
            'margins': {'top', 'bottom', 'left', 'right'},
            'title': {'font', 'size', 'bold', 'alignment', 'color'},
            'section': {'font', 'size', 'bold', 'heading_bold', 'color', 'space_before', 'space_after'},
            'subsection': {'indent', 'bold', 'font', 'size', 'color'},
            'point': {'indent', 'bold', 'font', 'size', 'color'}
        }

        try:
            for section, fields in required_settings.items():
                if section not in self.config:
                    return False
                for field in fields:
                    if field not in self.config[section]:
                        return False
            return True
        except Exception as e:
            return False

    def apply_margins(self, document: Document) -> None:
        margins = self.config['margins']
        for section in document.sections:
            section.top_margin = Inches(margins['top'])
            section.bottom_margin = Inches(margins['bottom'])
            section.left_margin = Inches(margins['left'])
            section.right_margin = Inches(margins['right'])

    def add_title(self, document: Document, title_text: str) -> None:
        title_config = self.config['title']
        title = document.add_paragraph()
        title.alignment = title_config['alignment']
        title_run = title.add_run(title_text)
        title_run.font.name = title_config['font']
        title_run.font.size = Pt(title_config['size'])
        title_run.bold = title_config['bold']
        color = title_config['color']
        title_run.font.color.rgb = RGBColor(color['r'], color['g'], color['b'])
        #document.add_paragraph()

    def add_section_header(self, document: Document, section_number: int, 
                         section_name: str, first_line: str) -> None:
        section_config = self.config['section']
        #heading = document.add_heading(level=section_config['level'])
        heading = document.add_heading(level=1)

        # Add section number
        run_number = heading.add_run(f"{section_number}. ")
        self._apply_run_format(run_number, section_config, bold = False)

        # Add section name
        run_name = heading.add_run(f"{section_name}: ")
        if self.template_name != "DLAIssuance":
            self._apply_run_format(run_name, section_config, bold = True)
        else:
            self._apply_run_format(run_name, section_config, bold = False)

        # Add first line
        if first_line:
            run_content = heading.add_run(first_line)
            self._apply_run_format(run_content, section_config, bold=False)

    def add_content_line(self, document: Document, line: str) -> None:
        section_config = self.config['section']
        para = document.add_paragraph()
        para.style = document.styles['Normal']
        para.paragraph_format.space_before = Pt(section_config['space_before'])
        para.paragraph_format.space_after = Pt(section_config['space_after'])

        if self.subsection_pattern.match(line):
            config = self.config['subsection']
            para.left_indent = Inches(config['indent'])
            run = para.add_run(line)
            self._apply_run_format(run, config)
        elif self.point_pattern.match(line):
            config = self.config['point']
            para.left_indent = Inches(config['indent'])
            run = para.add_run(line)
            self._apply_run_format(run, config)
        else:
            config = self.config['point']  # Use point indentation for regular text
            para.left_indent = Inches(config['indent'])
            run = para.add_run(line)
            self._apply_run_format(run, config)

    def _apply_run_format(self, run: Run, config: Dict[str, Any], bold: Optional[bool] = None) -> None:
        run.font.name = config['font']
        run.font.size = Pt(config['size'])
        run.bold = config['bold'] if bold is None else bold
        
        # if self.template_name == "DLAIssuance":
        #     run.underline = True
        
        
            
        color = config['color']
        run.font.color.rgb = RGBColor(color['r'], color['g'], color['b'])" doc_gen.py from typing import Annotated, List, Optional

import internal.report_gen.report_db as crud
from fastapi import APIRouter, Depends, HTTPException

# from internal.report_gen.report_db import engine # database, engine
from internal.report_gen.report_db_setup import Base
from internal.report_gen.report_models import (
    ReportDetailsSchema,
    ReportSectionSchema,
    SectionResponseInstanceSchema,
)
from internal.report_gen.template_models import (
    TemplateDetailsSchema,
    TemplateSectionSchema,
)
from models.models import Feedback, SearchRequest
from pydantic import BaseModel
from routers.auth import (
    get_current_user,
    get_user_id_from_username,
    langfuse,
    oauth2_scheme,
)
from routers.rag import create_citations, search_help
from settings import POSTGRES_PASSWORD
from sqlalchemy import MetaData, create_engine
from sqlalchemy.orm import Session

DATABASE_URL = f"postgresql://langfusedb:{POSTGRES_PASSWORD}@dlaauditconciergedb.postgres.database.usgovcloudapi.net:5432/postgres"

# SQLAlchemy
engine = create_engine(DATABASE_URL)
metadata = MetaData()
router = APIRouter()

Base.metadata.create_all(bind=engine)

# @router.post("/score-response")
# async def score_response(
#     token: Annotated[str, Depends(oauth2_scheme)], feedback: Feedback
# ):


def get_db():
    db = Session(bind=engine)
    try:
        yield db
    finally:
        db.close()


# TODO: make TemplateDetailsSchema
# get template
@router.get("/templates/{template_id}", response_model=TemplateDetailsSchema)
async def get_template(
    token: Annotated[str, Depends(oauth2_scheme)],
    template_id,
    db: Session = Depends(get_db),
):
    db_template = crud.get_template(db, template_id=template_id)
    if not db_template:
        raise HTTPException(status_code=404, detail="Report not found")
    return db_template


# get templates
@router.get("/templates", response_model=List[TemplateDetailsSchema])
async def get_templates_list(
    token: Annotated[str, Depends(oauth2_scheme)],
    skip: int = 0,
    limit: int = 10,
    db: Session = Depends(get_db),
):
    user = await get_current_user(token=token)
    username = user.username
    reports = crud.get_templates(
        db, skip=skip, limit=limit
    )  # , user_id=get_user_id_from_username(username))
    return reports


# create template (and create template from doc)
@router.post("/templates", response_model=TemplateDetailsSchema)
async def create_template(
    token: Annotated[str, Depends(oauth2_scheme)],
    template: TemplateDetailsSchema,
    db: Session = Depends(get_db),
):
    user = await get_current_user(token=token)
    username = user.username
    return crud.create_template(
        db=db, template=template
    )  # , user_id=get_user_id_from_username(username))


# create tone from text
# share document
# user access list
# create_report_from_template


@router.get("/reports/{report_id}", response_model=ReportDetailsSchema)
async def get_report(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id,
    db: Session = Depends(get_db),
):
    db_report = crud.get_report(db, report_id=report_id)
    if not db_report:
        raise HTTPException(status_code=404, detail="Report not found")
    return db_report


@router.get("/reports", response_model=List[ReportDetailsSchema])
async def get_reports_list(
    token: Annotated[str, Depends(oauth2_scheme)],
    skip: int = 0,
    limit: int = 10,
    db: Session = Depends(get_db),
):
    user = await get_current_user(token=token)
    username = user.username
    reports = crud.get_reports(
        db, skip=skip, limit=limit, user_id=get_user_id_from_username(username)
    )
    return reports


@router.post("/reports", response_model=ReportDetailsSchema)
async def create_report(
    token: Annotated[str, Depends(oauth2_scheme)],
    report: ReportDetailsSchema,
    template: Optional[TemplateDetailsSchema] = None,
    db: Session = Depends(get_db),
):
    user = await get_current_user(token=token)
    username = user.username
    if template:
        return crud.create_report_from_template(
            db=db,
            report=report,
            template=template,
            user_id=get_user_id_from_username(username),
        )
    return crud.create_report(
        db=db, report=report, user_id=get_user_id_from_username(username)
    )


@router.put("/reports/{report_id}", response_model=ReportDetailsSchema)
async def update_report(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    report: ReportDetailsSchema,
    db: Session = Depends(get_db),
):
    db_report = crud.update_report(db, report_id, report)
    if db_report is None:
        raise HTTPException(status_code=404, detail="Report not found")
    return db_report


@router.delete("/reports/{report_id}", response_model=dict)
async def delete_report(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    db: Session = Depends(get_db),
):
    if not crud.delete_report(db, report_id):
        raise HTTPException(status_code=404, detail="Report not found")
    return {"status": "success"}


# add get sections here
@router.get(
    "/reports/{report_id}/sections/{section_id}", response_model=ReportSectionSchema
)
async def get_section(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section_id: str,
    db: Session = Depends(get_db),
):
    db_section = crud.get_section(db, report_id=report_id, section_id=section_id)
    if not db_section:
        raise HTTPException(status_code=404, detail="Section not found")
    return db_section


@router.get("/reports/{report_id}/sections", response_model=List[ReportSectionSchema])
async def get_sections_list(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    skip: int = 0,
    limit: int = 10,
    db: Session = Depends(get_db),
):
    reports = crud.get_sections(db, report_id, skip=skip, limit=limit)
    return reports


class ReportDetailsSchemaWithSection(BaseModel):
    db_report: ReportDetailsSchema
    section_id: str


@router.post(
    "/reports/{report_id}/sections"
)  # , response_model=dict)# ReportDetailsSchema)
async def create_section(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section: ReportSectionSchema,
    db: Session = Depends(get_db),
):
    db_report, section_id = crud.add_section_to_report(db, report_id, section)
    if db_report is None:
        raise HTTPException(status_code=404, detail="Report not found")
    return ReportDetailsSchemaWithSection(
        db_report=ReportDetailsSchema.from_orm(db_report), section_id=str(section_id)
    )  # {"section_id": section_id, "report": ReportDetailsSchema.from_orm(db_report).dict()}# ReportDetailsSchema(**db_report.__dict__)}# .from_orm(db_report).dict()}
    # return db_report


@router.put(
    "/reports/{report_id}/sections/{section_id}", response_model=ReportSectionSchema
)
async def update_section(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section_id: str,
    section: ReportSectionSchema,
    db: Session = Depends(get_db),
):
    db_section = crud.update_section(db, report_id, section_id, section)
    if db_section is None:
        raise HTTPException(status_code=404, detail="Section not found")
    return db_section


@router.delete("/reports/{report_id}/sections/{section_id}", response_model=dict)
async def delete_section(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section_id: str,
    db: Session = Depends(get_db),
):
    if not crud.delete_section(db, report_id, section_id):
        raise HTTPException(status_code=404, detail="Section not found")
    return {"status": "success"}


# Create, Retrieve, Update, Delete for section instances
@router.get(
    "/reports/{report_id}/sections/{section_id}/instances",
    response_model=List[SectionResponseInstanceSchema],
)
def get_instances(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section_id: str,
    db: Session = Depends(get_db),
):
    db_section = crud.get_section(db, report_id, section_id)
    if not db_section:
        raise HTTPException(status_code=404, detail="Section not found")
    return db_section.content_array


@router.get(
    "/reports/{report_id}/sections/{section_id}/instances/{instance_id}",
    response_model=SectionResponseInstanceSchema,
)
def get_instance(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id,
    section_id,
    instance_id: str,
    db: Session = Depends(get_db),
):
    db_instance = crud.get_instance(db, report_id, section_id, instance_id)
    if not db_instance:
        raise HTTPException(status_code=404, detail="Instance not found")
    return db_instance


@router.post(
    "/reports/{report_id}/sections/{section_id}/instances",
    response_model=SectionResponseInstanceSchema,
)
def create_instance(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section_id: str,
    instance: SectionResponseInstanceSchema,
    db: Session = Depends(get_db),
):
    db_section, _, new_instance = crud.add_instance_to_section(
        db, report_id, section_id, instance
    )
    if not db_section:
        raise HTTPException(status_code=404, detail="Section not found")
    return new_instance


@router.put(
    "/reports/{report_id}/sections/{section_id}/instances/{instance_id}",
    response_model=str,
)
def update_instance(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section_id: str,
    instance_id: str,
    instance_update: SectionResponseInstanceSchema,
    db: Session = Depends(get_db),
):
    updated_instance = crud.update_instance(
        db, report_id, section_id, instance_id, instance_update
    )
    if not updated_instance:
        raise HTTPException(status_code=404, detail="Instance not found")
    return "success"


@router.delete(
    "/reports/{report_id}/sections/{section_id}/instances/{instance_id}",
    response_model=str,
)
def delete_instance(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    section_id: str,
    instance_id: str,
    db: Session = Depends(get_db),
):
    if not crud.delete_instance(db, report_id, section_id, instance_id):
        raise HTTPException(status_code=404, detail="Instance not found")
    return "success"


# @router.post(
#     "/reports/{report_id}/sections/{section_id}/instances", response_model=dict
# )
# async def create_instance(
#     token: Annotated[str, Depends(oauth2_scheme)],
#     report_id: str,
#     section_id: str,
#     instance: SectionResponseInstanceSchema,
#     db: Session = Depends(get_db),
# ):
#     return crud.add_instance_to_section(db, report_id, section_id, instance)


@router.post("/reports/{report_id}/sections/{section_id}/query", response_model=dict)
async def submit_query(
    token: Annotated[str, Depends(oauth2_scheme)],
    query: str,
    report_id: str,
    section_id: str,
    db: Session = Depends(get_db),
):
    # Placeholder for your query logic implementation
    # Return success for now
    return {"status": "query submitted", "query": query}


import base64
import os


@router.get("/reports/{report_id}/download")
async def download_as_pdf(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    db: Session = Depends(get_db),
):
    # get report and sections and selected instances
    # loop through sections and add them to document
    if crud.create_doc(db=db, report_id=report_id):
        crud.convert_to_pdf(report_id=report_id)
    with open(f"{report_id}_pandoc.pdf", "rb") as pdf_file:
        encoded_string_pandoc = base64.b64encode(pdf_file.read())
    with open(f"{report_id}.docx", "rb") as pdf_file:
        encoded_string_docx = base64.b64encode(pdf_file.read())
    # with open(f"{report_id}_docx2pdf.pdf", "rb") as pdf_file:
    #     encoded_string_docx2pdf = base64.b64encode(pdf_file.read())
    os.remove(f"{report_id}_pandoc.pdf")
    os.remove(f"{report_id}.docx")
    return {
        # "pandoc_file": encoded_string_pandoc,
        "docx_file": encoded_string_docx,
        # "docx2pdf_file": encoded_string_docx2pdf,
    }

@router.get("/reports/{report_id}/download/{export_template}")
async def download_as_pdf(
    token: Annotated[str, Depends(oauth2_scheme)],
    report_id: str,
    export_template: str,
    db: Session = Depends(get_db),
):
    # get report and sections and selected instances
    # loop through sections and add them to document
    if crud.create_doc(db=db, report_id=report_id, template_name=export_template):
        crud.convert_to_pdf(report_id=report_id)
    with open(f"{report_id}_pandoc.pdf", "rb") as pdf_file:
        encoded_string_pandoc = base64.b64encode(pdf_file.read())
    with open(f"{report_id}.docx", "rb") as pdf_file:
        encoded_string_docx = base64.b64encode(pdf_file.read())
    # with open(f"{report_id}_docx2pdf.pdf", "rb") as pdf_file:
    #     encoded_string_docx2pdf = base64.b64encode(pdf_file.read())
    os.remove(f"{report_id}_pandoc.pdf")
    os.remove(f"{report_id}.docx")
    return {
        # "pandoc_file": encoded_string_pandoc,
        "docx_file": encoded_string_docx,
        # "docx2pdf_file": encoded_string_docx2pdf,
    }

from internal.report_gen.doc_gen_agent import tools


@router.post("/reports/testing_gen")
def test_create_instance(
    token: Annotated[str, Depends(oauth2_scheme)],
    # report_id: str,
    # section_id: str,
    instance: SectionResponseInstanceSchema,
    db: Session = Depends(get_db),
):
    # section: ReportSectionSchema = crud.get_section(
    #     db=db, report_id=report_id, section_id=section_id
    # )
    results = tools.doc_gen_section_generator(
        instance_description=instance.instance_description,
        selected_files=instance.sources,
        tone="professional, analytical, Finance and audit minded",  # section.section_settings.get("tone", None),
        structure="Multiple small, 4-6 sentence paragraphs.",  # section.section_settings.get("structure", None),
        section_name="Audit Controls",  # section.section_name,
    )
    # db_section, _ = crud.add_instance_to_section(db, report_id, section_id, instance)
    # if not db_section:
    #     raise HTTPException(status_code=404, detail="Section not found")
    return {"answer": results["answer"], "sources": results["sources"]}"    template_models.py "from pydantic import BaseModel, Field
from typing import List, Optional, Dict
from uuid import UUID
import uuid


class TemplateSectionSchema(BaseModel):
    id: Optional[UUID] = Field(default_factory=uuid.uuid4)
    section_name: str
    template_id: Optional[UUID] = Field(default_factory=uuid.uuid4)
    section_description: str
    tone: str
    structure: str
    section_type: Optional[str] = "text_block"
    content: Optional[str]

    class Config:
        orm_mode = True
        from_attributes=True

class TemplateDetailsSchema(BaseModel):
    id: Optional[UUID] = Field(default_factory=uuid.uuid4)
    template_name: str
    description: str
    export_template: str
    sections: Optional[List[TemplateSectionSchema]]

    class Config:
        orm_mode = True
        from_attributes=True" Issuance.json "{
    "margins": {
        "top": 1,
        "bottom": 1,
        "left": 1,
        "right": 1
    },
    "title": {
        "font": "Times New Roman",
        "size": 11,
        "bold": true,
        "alignment": 1,
        "color": {"r": 0, "g": 0, "b": 0}
    },
    "section": {
        "font": "Times New Roman",
        "size": 12,
        "heading_bold": false,
        "bold": false,
        "color": {"r": 0, "g": 0, "b": 0},
        "space_before": 0,
        "space_after": 4,
        "underline": true
    },
    "subsection": {
        "indent": 0.4,
        "bold": false,
        "font": "Times New Roman",
        "size": 12,
        "color": {"r": 0, "g": 0, "b": 0}
    },
    "point": {
        "indent": 0.8,
        "bold": false,
        "font": "Times New Roman",
        "size": 12,
        "color": {"r": 0, "g": 0, "b": 0}
    }
}"
