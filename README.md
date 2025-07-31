import json
import networkx as nx
import re
from typing import Dict, List, Any


def clean_string(s: str) -> str:
    """Clean strings for Python code generation."""
    if not s:
        return ""
    # Remove leading/trailing whitespace and special characters
    s = re.sub(r'[^a-zA-Z0-9_ ]', '', s.strip())
    # Replace multiple spaces with single space
    return re.sub(r'\s+', ' ', s)


def to_camel_case(text: str) -> str:
    """Convert text to camelCase format for class names."""
    text = clean_string(text)
    if not text:
        return "Untitled"
    words = text.split()
    if not words:
        return "Untitled"
    return words[0].lower() + ''.join(word.capitalize() for word in words[1:])


def to_pascal_case(text: str) -> str:
    """Convert text to PascalCase format for class names."""
    text = clean_string(text)
    if not text:
        return "Untitled"
    words = text.split()
    if not words:
        return "Untitled"
    return ''.join(word.capitalize() for word in words)


def create_graph_from_json(json_data: Dict) -> nx.DiGraph:
    """Create a directed graph from JSON data."""
    G = nx.DiGraph()
    nodes = {node["id"]: node for node in json_data.get("nodes", [])}
    
    for edge in json_data.get("edges", []):
        from_node = edge.get("fromNode")
        to_node = edge.get("toNode")
        
        if from_node in nodes:
            label = nodes[from_node].get("label", nodes[from_node].get("text", ""))
            G.add_node(from_node, label=clean_string(label))
            
        if to_node in nodes:
            label = nodes[to_node].get("label", nodes[to_node].get("text", ""))
            G.add_node(to_node, label=clean_string(label))
        
        if from_node and to_node:
            edge_label = clean_string(edge.get("label", ""))
            G.add_edge(from_node, to_node, label=edge_label, id=edge.get("id"))

    return G


def get_elements_from_canvas(nodes: List[Dict]) -> List[Dict]:
    """Identify special elements (boards, composite functions) from nodes."""
    elems = []
    for node in nodes:
        if node.get("type") == "group":
            border_color = node.get("color", {}).get("border", "").lower()
            if border_color == "#2b7ce9":
                node["elem_type"] = "Board"
                elems.append(node)
            elif border_color == "#08b94e":
                node["elem_type"] = "CompositeFunction"
                elems.append(node)
    return elems


def find_closest_node(node: Dict, list_nodes: List[Dict]) -> Dict:
    """Find the closest containing group node."""
    if not list_nodes:
        return None
    
    closest_node = None
    min_area = float('inf')
    
    for n in list_nodes:
        node_area = n.get("width", 0) * n.get("height", 0)
        if (n.get("x", 0) < node.get("x", 0) and \
           (n.get("x", 0) + n.get("width", 0) > node.get("x", 0)) and \
           (n.get("y", 0) < node.get("y", 0)) and \
           (n.get("y", 0) + n.get("height", 0) > node.get("y", 0)) and \
           (node_area < min_area):
            closest_node = n
            min_area = node_area
    
    return closest_node


def generate_function_code(func_name: str, doc_str: str) -> str:
    """Generate a single Python function definition."""
    clean_name = clean_string(func_name).replace(' ', '_')
    if not clean_name:
        clean_name = "unnamed_function"
    
    return (
        f"def {clean_name}(mr) -> (bool, dict):\n"
        f'    """{clean_string(doc_str) or clean_name}"""\n'
        f'    print("{clean_name} called.")\n'
        f"    return True, {{}}\n\n"
    )


def generate_board_code(board_name: str, functions: Dict, paths: List) -> str:
    """Generate board.py content."""
    class_name = to_pascal_case(board_name)
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    
    # Generate imports
    imports = (
        "from radv.medical_record.validations.framework import Board\n"
        f"from .functions import {', '.join([clean_string(f).replace(' ', '_') for f in functions])}\n\n"
    )
    
    # Generate class definition
    class_def = (
        f"class {class_name}(Board):\n"
        f"    functions = {{\n"
    )
    
    # Add functions dictionary
    func_entries = []
    for func_name in functions:
        clean_func = clean_string(func_name).replace(' ', '_')
        func_entries.append(f'        "{clean_string(func_name)}": {clean_func}')
    
    class_def += ",\n".join(func_entries) + "\n    }\n\n"
    
    # Add paths
    class_def += f"    paths = {json.dumps(paths, indent=8)}\n"
    
    return disclaimer + imports + class_def


def generate_workflow_code(flow_name: str, boards: List[str]) -> str:
    """Generate workflow.py content."""
    class_name = to_pascal_case(flow_name)
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    
    # Generate imports
    imports = "from radv.medical_record.validations.framework import Workflow\n"
    for board in boards:
        clean_board = to_pascal_case(board)
        imports += f"from {flow_name}.Board_{clean_board}.board import {clean_board}\n"
    
    # Generate class definition
    class_def = (
        f"\nclass {class_name}(Workflow):\n"
        f"    boards = [{', '.join([to_pascal_case(b) for b in boards])}]\n"
    )
    
    return disclaimer + imports + class_def


def get_paths(canvas_data: Dict, graph: nx.DiGraph) -> Dict:
    """Analyze paths through the graph."""
    nodes = canvas_data.get("nodes", [])
    elements = get_elements_from_canvas(nodes)
    roots = {}
    
    # Identify root nodes
    for node in nodes:
        node_id = node.get("id")
        if graph.in_degree(node_id) == 0:
            closest_element = find_closest_node(node, elements)
            if closest_element:
                elem_type = closest_element.get("elem_type", "Unknown")
                roots[node_id] = {
                    "label": clean_string(closest_element.get("label", "")),
                    "type": elem_type,
                    "paths": [],
                    "functions": {}
                }
            else:
                roots[node_id] = {
                    "label": clean_string(node.get("label", "")),
                    "type": "Unknown",
                    "paths": [],
                    "functions": {}
                }

    # Analyze paths for each root
    for root_id, root_info in roots.items():
        leaf_nodes = [n for n in graph.nodes() if graph.out_degree(n) == 0]
        functions = {}
        paths = []

        for leaf in leaf_nodes:
            for path in nx.all_simple_paths(graph, source=root_id, target=leaf):
                path_functions = []
                triggers = []
                
                for i, node_id in enumerate(path):
                    node_label = graph.nodes[node_id].get("label", "")
                    clean_label = clean_string(node_label)
                    path_functions.append(clean_label.replace(' ', '_'))
                    
                    if i < len(path) - 1:
                        edge_data = graph.get_edge_data(node_id, path[i+1])
                        triggers.append(clean_string(edge_data.get("label", "")))

                # Record path information
                if path_functions:
                    paths.append({
                        "trigger": dict(zip(path_functions, triggers)),
                        "action": path_functions[-1]
                    })
                    
                    # Build functions dictionary
                    for func in path_functions[:-1]:
                        if func not in functions:
                            functions[func] = func.replace('_', ' ').title()

        root_info["paths"] = paths
        root_info["functions"] = functions

    return roots


def generate_all_code(json_string: str, flow_name: str) -> Dict[str, str]:
    """Generate all Python code files from graph JSON."""
    try:
        canvas_data = json.loads(json_string)
        graph = create_graph_from_json(canvas_data)
        paths_dict = get_paths(canvas_data, graph)
        
        generated_content = {}
        boards = []

        # Generate code for each element
        for elem_id, elem_info in paths_dict.items():
            elem_type = elem_info["type"]
            elem_label = elem_info["label"] or "Untitled"
            functions = elem_info["functions"]
            paths = elem_info["paths"]

            # Generate functions.py content
            func_content = (
                '"""\n'
                "DISCLAIMER:\n"
                "This python file was created automatically.\n"
                '"""\n\n'
            )
            for func_name, doc_str in functions.items():
                func_content += generate_function_code(func_name, doc_str)

            # Generate board or composite function content
            if elem_type == "Board":
                board_class = to_pascal_case(elem_label)
                boards.append(board_class)
                
                # Generate board.py
                board_content = generate_board_code(elem_label, functions, paths)
                generated_content[f"Board_{board_class}/board.py"] = board_content
                generated_content[f"Board_{board_class}/functions.py"] = func_content
                
            elif elem_type == "CompositeFunction":
                comp_class = to_pascal_case(elem_label)
                comp_content = generate_board_code(elem_label, functions, paths)
                generated_content[f"CompositeFunction_{comp_class}/compositefunction.py"] = comp_content
                generated_content[f"CompositeFunction_{comp_class}/functions.py"] = func_content

        # Generate workflow.py
        if boards:
            workflow_content = generate_workflow_code(flow_name, boards)
            generated_content["workflow.py"] = workflow_content

        return generated_content

    except Exception as e:
        return {"error": f"Code generation failed: {str(e)}"}
