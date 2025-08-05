def split_question_label(label: str):
    """
    Splits a label on '?'. Returns (before, after).
    If '?' not found, returns ("", label).
    """
    if '?' in label:
        before, after = label.split('?', 1)
        return before.strip(), after.strip()
    return "", label.strip()


def generate_function_code(func_label: str) -> str:
    """
    Generate a single Python function definition.
    If the label contains a '?', before is the comment, after is the function name.
    """
    comment, name_part = split_question_label(func_label)
    clean_func_name = to_snake_case(name_part) or "unnamed_function"
    result = f"def {clean_func_name}(mr) -> (bool, dict):\n"
    if comment:
        result += f"    # {comment}\n"
    result += f'    """{name_part}"""\n'
    result += f'    print("{clean_func_name} called.")\n'
    result += f"    return True, {{}}\n\n"
    return result


for func_label in functions:
    functions_content += generate_function_code(func_label)


for func_label in functions:
    _, name_part = split_question_label(func_label)
    clean_func = to_snake_case(name_part)
    func_entries.append(f'        "{name_part}": {clean_func}')
