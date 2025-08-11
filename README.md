def generate_function_code(raw_label: str, fallback_name: str = "") -> str:
    """
    Generate a single Python function.
    - raw_label: original node label, may contain "?"
      * left of "?" becomes docstring
      * right of "?" becomes function name
    - fallback_name: used if raw_label has no "?" (already cleaned)
    """
    raw = (raw_label or "").strip()
    if "?" in raw:
        doc_part, name_part = raw.split("?", 1)
        doc_part  = doc_part.strip()
        name_part = name_part.strip()
    else:
        # No "?" present → doc from raw, name from fallback/raw
        doc_part  = raw
        name_part = fallback_name or raw

    clean_name = to_snake_case(name_part) or "unnamed_function"
    clean_doc  = clean_string(doc_part) or clean_name

    return (
        f"def {clean_name}(mr) -> (bool, dict):\n"
        f'    """{clean_doc}"""\n'
        f'    print("{clean_name} called.")\n'
        f"    return True, {{}}\n\n"
    )


def generate_board_files(board_name: str, functions: Dict[str, str], paths: List) -> Dict[str, str]:
    """
    Generate all files for a single board.

    functions: dict mapping { clean_func_key: raw_label_with_possible_question_mark }
               (Values come from get_paths → graph.nodes[node_id]['raw_label'])
    """
    snake_name = to_snake_case(board_name)
    pascal_name = to_pascal_case(board_name)

    # Build consistent display→callable mapping by parsing each raw label.
    # - display name (key in Board.functions) = left of '?', cleaned
    # - callable name (import + value)        = right of '?', snake_cased
    function_map: Dict[str, str] = {}   # callable_name -> display_name
    for clean_key, raw in functions.items():
        raw = (raw or "").strip()
        if "?" in raw:
            left, right = raw.split("?", 1)
            display = clean_string(left.strip()) or clean_string(right.strip()) or clean_key
            callable_name = to_snake_case(right.strip()) or to_snake_case(clean_key) or "unnamed_function"
        else:
            # No "?" — fall back to the cleaned key for the name; doc/display from raw
            display = clean_string(raw) or clean_key
            callable_name = to_snake_case(clean_key) or "unnamed_function"

        # keep first occurrence per callable
        if callable_name not in function_map:
            function_map[callable_name] = display

    # ----- board.py -----
    imports = ", ".join(sorted(function_map.keys())) or ""
    board_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
        f"from ...framework import Board\n"
        f"from .functions import {imports}\n\n"
        f"class {pascal_name}(Board):\n"
        f"    functions = {{\n"
        + ",\n".join([f'        "{display}": {callable_name}'
                     for callable_name, display in function_map.items()])
        + "\n    }\n\n"
        f"    paths = {json.dumps(paths, indent=8)}\n"
    )

    # ----- functions.py -----
    functions_content = '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
    seen = set()
    for clean_key, raw in functions.items():
        # derive the callable name exactly the same way
        if "?" in (raw or ""):
            _, right = raw.split("?", 1)
            callable_name = to_snake_case(right.strip()) or to_snake_case(clean_key) or "unnamed_function"
        else:
            callable_name = to_snake_case(clean_key) or "unnamed_function"

        if callable_name in seen:
            continue
        # Generate using the RAW label for split + a fallback for safety
        functions_content += generate_function_code(raw, fallback_name=clean_key)
        seen.add(callable_name)

    # ----- __init__.py -----
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
