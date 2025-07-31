def generate_all_code(json_string: str, flow_name: str) -> Dict[str, str]:
    """Generate all Python code files from graph JSON with dynamic folder splitting."""
    try:
        canvas_data = json.loads(json_string)
        graph = create_graph_from_json(canvas_data)
        paths_dict = get_paths(canvas_data, graph)
        
        generated_content = {}
        boards = []
        disclaimer = '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'

        # First pass: generate combined board file
        combined_board_content = disclaimer
        for elem_id, elem_info in paths_dict.items():
            if elem_info["type"] == "Board":
                board_name = elem_info["label"] or "UntitledBoard"
                pascal_name = to_pascal_case(board_name)
                snake_name = to_snake_case(board_name)
                boards.append(board_name)
                
                # Add board class to combined file
                combined_board_content += (
                    f"class {pascal_name}(Board):\n"
                    f"    functions = {{\n"
                )
                
                # Add functions dictionary
                func_entries = []
                for func_name in elem_info["functions"]:
                    clean_func = to_snake_case(func_name)
                    func_entries.append(f'        "{clean_string(func_name)}": {clean_func}')
                
                combined_board_content += ",\n".join(func_entries) + "\n    }\n\n"
                combined_board_content += f"    paths = {json.dumps(elem_info['paths'], indent=8)}\n\n"

        generated_content["combined_boards.py"] = combined_board_content

        # Second pass: split into separate folders when we see duplicate disclaimers
        if "combined_boards.py" in generated_content:
            content = generated_content["combined_boards.py"]
            board_sections = content.split(disclaimer)[1:]  # Split by disclaimer
            
            for i, section in enumerate(board_sections):
                if not section.strip():
                    continue
                    
                board_name = boards[i] if i < len(boards) else f"Board_{i+1}"
                snake_name = to_snake_case(board_name)
                pascal_name = to_pascal_case(board_name)
                
                # Create board folder files
                generated_content[f"Board_{snake_name}/board.py"] = disclaimer + section
                
                # Create matching functions.py
                functions_content = disclaimer
                for func_name, doc_str in paths_dict[list(paths_dict.keys())[i]]["functions"].items():
                    functions_content += generate_function_code(func_name, doc_str)
                
                generated_content[f"Board_{snake_name}/functions.py"] = functions_content
                
                # Add __init__.py
                generated_content[f"Board_{snake_name}/__init__.py"] = (
                    f'"""\nPackage initialization.\n"""\n\n'
                    f"from .board import {pascal_name}\n"
                    f"__all__ = ['{pascal_name}']\n"
                )

            # Remove the temporary combined file
            del generated_content["combined_boards.py"]

        # Generate workflow.py if we have boards
        if boards:
            generated_content["workflow.py"] = generate_workflow_code(flow_name, boards)
            generated_content["__init__.py"] = '"""\nPackage initialization.\n"""\n'

        return generated_content

    except Exception as e:
        return {"error": f"Code generation failed: {str(e)}"}
