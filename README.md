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
        self._apply_run_format(run_name, section_config, bold = True)

        # Add first line
        if first_line:
            run_content = heading.add_run(first_line)
            self._apply_run_format(run_content, section_config, bold=False)

    def add_content_line(self, document: Document, line: str) -> None:
        section_config = self.config['section']
        para = document.add_paragraph()
        para.style = document.styles['Normal']
        para.paragraph_format.space_before = Pt(section_config['space_before'])
        if self.template_name != "speaking_notes" or self.template_name != "information_paper":
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
        color = config['color']
        run.font.color.rgb = RGBColor(color['r'], color['g'], color['b'])
