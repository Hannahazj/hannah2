def generate_function_code(func_name: str, doc_str: str) -> str:
    """Generate a single Python function definition."""
    # Split function name at "?" if present
    if "?" in func_name:
        parts = func_name.split("?")
        func_part = parts[-1].strip()  # Use part after ? as function name
        doc_part = parts[0].strip()  # Use part before ? as docstring
    else:
        func_part = func_name
        doc_part = doc_str
    
    clean_name = to_snake_case(func_part) or "unnamed_function"
    clean_doc = clean_string(doc_part) or clean_name
    
    return (
        f"def {clean_name}(mr) -> (bool, dict):\n"
        f'    """{clean_doc}"""\n'
        f'    print("{clean_name} called.")\n'
        f"    return True, {{}}\n\n"
    )



    def generate_board_files(board_name: str, functions: Dict, paths: List) -> Dict[str, str]:
    """Generate all files for a single board."""
    snake_name = to_snake_case(board_name)
    pascal_name = to_pascal_case(board_name)
    
    # Generate board.py content
    board_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
        f"from ...framework import Board\n"
        f"from .functions import {', '.join([to_snake_case(f.split('?')[-1] if '?' in f else f) for f in functions])}\n\n"
        f"class {pascal_name}(Board):\n"
        f"    functions = {{\n"
    )
    
    # Add functions dictionary - ensure each function only appears once
    func_entries = []
    seen_functions = set()
    for func_name in functions:
        # Get the clean function name (after ? if present)
        clean_func = to_snake_case(func_name.split('?')[-1] if '?' in func_name else func_name)
        if clean_func in seen_functions:
            continue
        seen_functions.add(clean_func)
        
        # Keep the original display name (before ? if present)
        display_name = func_name.split('?')[0] if '?' in func_name else func_name
        func_entries.append(f'        "{clean_string(display_name)}": {clean_func}')
    
    board_content += ",\n".join(func_entries) + "\n    }\n\n"
    board_content += f"    paths = {json.dumps(paths, indent=8)}\n"
    
    # Generate functions.py content - ensure each function only appears once
    functions_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
    )
    seen_functions = set()
    for func_name, doc_str in functions.items():
        clean_func = to_snake_case(func_name.split('?')[-1] if '?' in func_name else func_name)
        if clean_func in seen_functions:
            continue
        seen_functions.add(clean_func)
        functions_content += generate_function_code(func_name, doc_str)
    
    # Generate __init__.py content
    init_content = (
        '"""\nPackage initialization.\n"""\n\n'
        f"from .board import {pascal_name}\n"
        f"__all__ = ['{pascal_name}']\n"
    )
    
    return {
        f"Board_{snake_name}/board.py": board_content,
        f"Board_{snake_name}/functions.py": functions_content,
        f"Board_{snake_name}/__init__.py": init_content
    }
