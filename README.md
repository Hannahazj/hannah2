def generate_function_code(func_name: str, doc_str: str) -> str:
    """Generate a single Python function definition."""
    # Split function name at "?" if present
    if "?" in func_name:
        func_part = func_name.split("?")[-1].strip()  # Use part after ? as function name
        doc_part = func_name.split("?")[0].strip()    # Use part before ? as docstring
    else:
        func_part = func_name
        doc_part = doc_str or func_name  # Fallback to func_name if no doc_str
    
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
    
    # First pass: create mapping of clean function names to display names
    function_map = {}
    for func_name in functions:
        if "?" in func_name:
            display_name = func_name.split("?")[0].strip()
            clean_func = to_snake_case(func_name.split("?")[-1].strip())
        else:
            display_name = func_name
            clean_func = to_snake_case(func_name)
        
        # Only keep the first occurrence of each clean function name
        if clean_func not in function_map:
            function_map[clean_func] = display_name
    
    # Generate board.py content
    board_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
        f"from ...framework import Board\n"
        f"from .functions import {', '.join(sorted(function_map.keys()))}\n\n"
        f"class {pascal_name}(Board):\n"
        f"    functions = {{\n"
    )
    
    # Add functions dictionary using display names as keys
    func_entries = []
    for clean_func, display_name in function_map.items():
        func_entries.append(f'        "{clean_string(display_name)}": {clean_func}')
    
    board_content += ",\n".join(func_entries) + "\n    }\n\n"
    board_content += f"    paths = {json.dumps(paths, indent=8)}\n"
    
    # Generate functions.py content
    functions_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
    )
    
    # Second pass: generate function code using original mapping
    seen_functions = set()
    for func_name, doc_str in functions.items():
        if "?" in func_name:
            clean_func = to_snake_case(func_name.split("?")[-1].strip())
        else:
            clean_func = to_snake_case(func_name)
            
        if clean_func not in seen_functions:
            functions_content += generate_function_code(func_name, doc_str)
            seen_functions.add(clean_func)
    
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
