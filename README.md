def generate_function_code(func_name: str, doc_str: str) -> str:
    """
    Generate a single Python function definition.
    If the function name contains a question mark,
    the part before '?' becomes a comment, and the part after becomes the function name.
    """
    comment = ""
    name_part = func_name
    # Split on '?', only at the first occurrence
    if '?' in func_name:
        parts = func_name.split('?', 1)
        comment = parts[0].strip()
        name_part = parts[1].strip()
    
    clean_func_name = to_snake_case(name_part) or "unnamed_function"
    result = f"def {clean_func_name}(mr) -> (bool, dict):\n"
    if comment:
        result += f"    # {comment}\n"
    # Prefer the passed doc_str unless empty
    doc = clean_string(doc_str) if doc_str and doc_str.strip() else clean_func_name
    result += f'    """{doc}"""\n'
    result += f'    print("{clean_func_name} called.")\n'
    result += f"    return True, {{}}\n\n"
    return result


for i, node_id in enumerate(path):
    node_label = graph.nodes[node_id].get("label", "")
    # Split on '?'
    if '?' in node_label:
        _, name_part = node_label.split('?', 1)
        clean_label = clean_string(name_part)
    else:
        clean_label = clean_string(node_label)
    path_functions.append(to_snake_case(clean_label))
