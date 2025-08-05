import json
import networkx as nx
import re


def create_graph_from_json(json_data):
    G = nx.DiGraph()
    nodes = {node["id"]: node for node in json_data["nodes"]}
    
    for edge in json_data["edges"]:
        from_node = edge["fromNode"]
        to_node = edge["toNode"]
        
        if from_node in nodes:
            G.add_node(from_node, label=nodes[from_node].get("label", nodes[from_node].get("text", "")))
        if to_node in nodes:
            G.add_node(to_node, label=nodes[to_node].get("label", nodes[to_node].get("text", "")))
        
        G.add_edge(from_node, to_node, label=edge["label"], id=edge["id"])
    
    return G


def get_elements_from_canvas(nodes):
    elems = []
    
    for node in nodes:
        # Example logic to detect certain node attributes
        if "color" in node.keys():
            if node["color"]["border"] == "#2b7ce9" and node["type"] == "group":
                node["elem_type"] = "Board"
                elems.append(node)
            elif node["color"]["border"] == "#2b7ce9" and node["type"] == "group":
                node["elem_type"] = "CompositeFunction"
                elems.append(node)
    
    return elems


def find_closest_node(node, list_nodes):
    if not list_nodes:
        return None
    # Your existing logic to find the "closest" group
    closest_node = list_nodes[0]
    min_area = closest_node["width"] * closest_node["height"]
    
    for n in list_nodes:
        node_area = n["width"] * n["height"]
        
        if (
            (n["x"] < node["x"]) and
            (n["x"] + n["width"] > node["x"]) and
            (n["y"] < node["y"]) and
            (n["y"] + n["height"] > node["y"]) and
            (node_area < min_area)
        ):
            closest_node = n
            min_area = node_area
    
    return closest_node


def merge_unique_key_value_pairs(dictionary_list):
    merged = {}
    
    for d in dictionary_list:
        for key, value in d.items():
            if key not in merged:
                merged[key] = value
    
    return merged


def string_to_bool_or_original(value):
    if value == "True":
        return True
    elif value == "False":
        return False
    else:
        return value


def extract_between_brackets(text):
    pattern = r"[[(.*?)]]"
    matches = re.findall(pattern, text)
    
    if len(matches) == 0:
        return [text]
    else:
        return matches


def to_camel_case(text):
    words = text.strip().split(" ")
    if len(words) == 1:
        return text.capitalize()
    return "".join(w.capitalize() for w in words)


def get_paths(canvas_data, graph):
    """
    Build a structure describing all root -> leaf paths
    and the functions in each step, triggers, actions, etc.
    """
    nodes = canvas_data["nodes"]
    elements = get_elements_from_canvas(nodes)
    roots = {}
    
    # Identify root nodes
    for x in nodes:
        if graph.in_degree(x["id"]) == 0:
            closest_element = find_closest_node(x, elements)
            if closest_element:
                closest_element_label = closest_element.get("label", closest_element.get("text", ""))
                closest_element_type = closest_element["elem_type"]
                roots[x["id"]] = {
                    "label": closest_element_label,
                    "type": closest_element_type,
                    "paths": [],
                }
            else:
                fallback_label = x.get("label", x.get("text", ""))
                roots[x["id"]] = {
                    "label": fallback_label,
                    "type": "Unknown",
                    "paths": [],
                }

    # For each root, gather possible paths
    for root in roots:
        func_map = []
        leaf_nodes = [nid for nid in graph.nodes() if graph.out_degree(nid) == 0]
        paths = []

        for leaf in leaf_nodes:
            possible_paths = list(nx.all_simple_paths(graph, source=root, target=leaf))
            
            for p in possible_paths:
                funcs = []
                doc_strs = []
                
                for i in p:
                    node_label = graph.nodes[i].get("label", graph.nodes[i].get("text", ""))
                    doc_str = node_label.split("[")[0]
                    doc_strs.append(doc_str)
                    bracket_content = extract_between_brackets(node_label)[0].split("|")
                    
                    if len(bracket_content) == 1:
                        func_name = bracket_content[0]
                    else:
                        # e.g. if bracket_content = ["something", "functionName"]
                        func_name = bracket_content[-1]
                    
                    funcs.append(func_name)

                func_map.append(dict(zip(funcs[:-1], doc_strs[:-1])))

                triggers = []
                for s, t in zip(p[:-1], p[1:]):
                    z = graph.get_edge_data(s, t)["label"]
                    triggers.append(z)
                
                # Last function is the "action"
                final_action = string_to_bool_or_original(funcs[-1])
                paths.append({"trigger": dict(zip(funcs, triggers)), "action": final_action})

        updated_map = merge_unique_key_value_pairs(func_map)
        roots[root]["paths"] = paths
        roots[root]["functions"] = updated_map

    return roots


# IN-MEMORY CODE GENERATORS
def generate_function_code(flow_name, elem_label, elem_type, functions_dict, markdown_lookup=None):
    """
    Generates the content that would go into functions.py (as a string).
    Optionally reads from an external “markdown_lookup” dict if needed.
    Returns a single string.
    """
    # If you have no Markdown or don’t need it, simply omit that part
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    code_lines = [disclaimer]
    
    # If you had a dictionary of markdown_content keyed by function name:
    # markdown_lookup = { "functionA": "...md content..." }

    composite_funcs = []  # You might gather them separately if needed

    for func_name, doc_str in functions_dict.items():
        # Optionally skip if it's a composite function; depends on your logic
        if func_name in composite_funcs:
            continue

        # If you have actual Python code from markdown_lookup
        # just sample logic below:
        if markdown_lookup and func_name in markdown_lookup:
            py_code = markdown_lookup[func_name]
            # Minimal check or parse
            if len(py_code) > 5:
                code_lines.append(py_code.strip() + "\n\n")
            else:
                # fallback
                code_lines.append(
                    f"def {func_name}(mr) -> (bool, dict):\n"
                    f'    """{doc_str}"""\n'
                    f'    print("Function {func_name} called.")\n'
                    f"    return (True, {{}})\n\n"
                )
        else:
            # fallback
            code_lines.append(
                f"def {func_name}(mr) -> (bool, dict):\n"
                f'    """{doc_str}"""\n'
                f'    print("Function {func_name} called.")\n'
                f"    return (True, {{}})\n\n"
            )

    return "".join(code_lines)


def generate_board_code(flow_name, elem_label, elem_type, functions_dict, paths):
    """
    Generates the content for board.py or compositefunction.py (as a string).
    """
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    code_lines = [disclaimer]
    # Possibly an import to your validation framework
    code_lines.append(f"from TO_BE_REPLACED.framework import {elem_type}\n")

    # Then, if referencing local functions or composite ones
    # This is where you might import from .functions
    # or from another node’s compositefunction if needed.
    func_names = list(functions_dict.keys())
    imported_func_list = ", ".join(func_names)
    code_lines.append(f"from .functions import {imported_func_list}\n\n")

    # Define the class for the board or composite function
    class_name = to_camel_case(elem_label)
    code_lines.append(f"class {class_name}({elem_type}):\n")

    # Indent assignment lines
    # Convert functions_dict keys to a dictionary { "funcName": funcName, ... }
    dict_items = ", ".join([f"'{k}': {k}" for k in functions_dict])
    code_lines.append(f"    functions = {{{dict_items}}}\n")
    code_lines.append(f"    paths = {paths}\n")

    return "".join(code_lines)


def generate_workflow_code(flow_name, boards):
    """
    Generates the content for workflow.py (as a string).
    """
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    code_lines = [disclaimer]
    code_lines.append("from TO_BE_REPLACED import Workflow\n\n")
    
    # Import any boards you discovered
    for board_class_name in boards:
        code_lines.append(
            f"from TO_BE_REPLACED.{flow_name}.Board_{board_class_name}.board import {board_class_name}\n"
        )

    code_lines.append("\n")
    code_lines.append(f"class {flow_name}(Workflow):\n")
    # boards = ["BoardOne", "BoardTwo", ...]
    joined_boards = ", ".join(boards)
    code_lines.append(f"    boards = [{joined_boards}]\n")

    return "".join(code_lines)


def generate_all_code(json_string, flow_name):
    """
    Example “master” function that:
      1) Reads the JSON (already in memory as a string)
      2) Builds the graph
      3) Extracts paths, boards, composite functions
      4) Generates in-memory code for each board
      5) Generates in-memory code for each composite function
      6) Generates in-memory “workflow” code
    Returns a data structure with all code as strings
    """
    # 1) Load data
    canvas_data = json.loads(json_string)
    graph = create_graph_from_json(canvas_data)
    paths_dict = get_paths(canvas_data, graph)
    
    # 2) Identify boards, composites, etc.
    boards = []
    generated_content = {}

    for key, info in paths_dict.items():
        elem_label = info["label"]
        elem_type = info["type"]
        func_dict = info["functions"]
        path_data = info["paths"]

        # Convert label for directory/class usage
        u_elem_label = to_camel_case(elem_label)

        # (A) functions.py content
        func_py_content = generate_function_code(
            flow_name, elem_label, elem_type, func_dict
        )

        # (B) board.py or compositefunction.py content
        if elem_type == "Board":
            board_py_content = generate_board_code(
                flow_name, elem_label, elem_type, func_dict, path_data
            )
            boards.append(u_elem_label)

            # Store in memory
            generated_content[f"{elem_type}_{u_elem_label}/board.py"] = board_py_content
            generated_content[f"{elem_type}_{u_elem_label}/functions.py"] = func_py_content

        elif elem_type == "CompositeFunction":
            comp_py_content = generate_board_code(
                flow_name, elem_label, elem_type, func_dict, path_data
            )
            # Store in memory
            generated_content[f"{elem_type}_{u_elem_label}/compositefunction.py"] = comp_py_content
            generated_content[f"{elem_type}_{u_elem_label}/functions.py"] = func_py_content

    # 3) Finally, workflow.py
    workflow_content = generate_workflow_code(flow_name, boards)
    generated_content["workflow.py"] = workflow_content

    return generated_content


# Testing with the provided JSON string
json_str = """
{
  "nodes": [
    {
      "id": "3e7b8acd417adc40",
      "type": "group",
      "x": -595.5,
      "y": -269,
      "width": 667,
      "height": 483,
      "label": "Flow",
      "color": {
        "border": "#2b7ce9",
        "background": "rgba(43,124,233,0.122)",
        "highlight": {
          "border": "#2b7ce9",
          "background": "rgba(43,124,233,0.122)"
        },
        "hover": {
          "border": "#2b7ce9",
          "background": "rgba(43,124,233,0.122)"
        },
        "selected": {
          "border": "#2b7ce9",
          "background": "rgba(43,124,233,0.122)"
        }
      }
    },
    {
      "id": "04f402c4e2af7f56",
      "type": "group",
      "x": -575.5,
      "y": -109,
      "width": 622,
      "height": 303,
      "label": "Replace",
      "color": {
        "border": "#08b94e",
        "background": "rgba(8,185,78,0.147)",
        "highlight": {
          "border": "#08b94e",
          "background": "rgba(8,185,78,0.147)"
        },
        "hover": {
          "border": "#08b94e",
          "background": "rgba(8,185,78,0.147)"
        },
        "selected": {
          "border": "#08b94e",
          "background": "rgba(8,185,78,0.147)"
        }
      }
    },
    {
      "id": "6d1ef69553b877d9",
      "type": "text",
      "text": "Replace with text",
      "x": -553,
      "y": 92,
      "width": 272,
      "height": 82,
      "color": {
        "border": "#f6f4f4",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#f6f4f4",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#f6f4f4",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#f6f4f4",
          "background": "rgba(0,0,0,0)"
        }
      }
    },
    {
      "id": "214991ab8cafdfbc",
      "type": "text",
      "text": "Replace with text",
      "x": -248,
      "y": 92,
      "width": 272,
      "height": 82,
      "color": {
        "border": "#f4f0f0",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#f4f0f0",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#f4f0f0",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#f4f0f0",
          "background": "rgba(0,0,0,0)"
        }
      }
    },
    {
      "id": "01c0c38aeb5e5e13",
      "type": "text",
      "text": "Replace with Text",
      "x": -273,
      "y": -249,
      "width": 322,
      "height": 122,
      "color": {
        "border": "#e0ac00",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#e0ac00",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#e0ac00",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#e0ac00",
          "background": "rgba(0,0,0,0)"
        }
      }
    },
    {
      "id": "6babf3007067a1e7",
      "type": "text",
      "text": "Replace with Text",
      "x": -493,
      "y": -89,
      "width": 402,
      "height": 102,
      "color": {
        "border": "#086ddd",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#086ddd",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#086ddd",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#086ddd",
          "background": "rgba(0,0,0,0)"
        }
      }
    }
  ],
  "edges": [
    {
      "id": "37f808b16516c77e",
      "fromNode": "6babf3007067a1e7",
      "toNode": "6d1ef69553b877d9",
      "fromSide": "bottom",
      "toSide": "top",
      "label": "True",
      "color": "#e0ac00"
    },
    {
      "id": "b5e7f2a787970ec8",
      "fromNode": "6babf3007067a1e7",
      "toNode": "214991ab8cafdfbc",
      "fromSide": "bottom",
      "toSide": "top",
      "label": "False",
      "color": "#e93147"
    }
  ]
}
"""


#res = generate_all_code(json_str, "example")
#print(json.dumps(res, indent=2))
