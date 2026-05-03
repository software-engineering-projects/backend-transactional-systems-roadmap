
### 0. Core Design Principle

This system is not a call graph tool. It is a state-transition observability system. Every component must answer:

*   What is the state?
*   When does it change?
*   What caused the change?

---

### 1. High-Level Architecture

The system consists of four primary modules:

1.  **Analyzer:** Static structure (AST)
2.  **Tracker:** Runtime state and transitions
3.  **Recorder:** Logs transitions
4.  **Visualizer:** Graph output

---

### 2. Module 1 — Analyzer (AST Parsing)

**Purpose:** Extract structure including functions, modules, and call relationships.

**Responsibilities:**
*   Parse .py files.
*   Identify functions, classes, imports, and function calls.

**Tools:** `ast` (built-in Python module)

**Skeleton:**
```python
import ast

class Analyzer(ast.NodeVisitor):
    def __init__(self):
        self.functions = {}
        self.calls = []

    def visit_FunctionDef(self, node):
        self.functions[node.name] = {
            "lineno": node.lineno,
            "calls": []
        }
        self.current_function = node.name
        self.generic_visit(node)

    def visit_Call(self, node):
        if isinstance(node.func, ast.Name):
            self.calls.append((self.current_function, node.func.id))
        self.generic_visit(node)
```

**Output:**
```json
{
    "functions": {...},
    "calls": [("A", "B"), ("B", "C")]
}
```

---

### 3. Module 2 — Tracker (Runtime Instrumentation)

**Purpose:** Capture actual execution and state transitions.

**Key Idea:** Wrap functions with a decorator.

**Defining State:**
*   **Option A (Simple):** Function inputs and outputs.
*   **Option B (Advanced):** Tracked variables (dictionary snapshot).

**Skeleton:**
```python
import functools

class Tracker:
    def __init__(self, recorder):
        self.recorder = recorder

    def track(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            before_state = {
                "args": args,
                "kwargs": kwargs
            }

            result = func(*args, **kwargs)

            after_state = {
                "result": result
            }

            self.recorder.record(
                func.__name__,
                before_state,
                after_state
            )

            return result
        return wrapper
```

---

### 4. Module 3 — Recorder (Event Log)

**Purpose:** Store transitions as events.

**Structure:**
```python
class Recorder:
    def __init__(self):
        self.events = []

    def record(self, func_name, before, after):
        self.events.append({
            "function": func_name,
            "before": before,
            "after": after
        })
```

**Future Extensions:**
*   Timestamps
*   Call stack
*   Parent-child relationships

---

### 5. Module 4 — Visualizer (Graph)

**Purpose:** Convert logs into a graph.

**Tools:** Graphviz

**Skeleton:**
```python
from graphviz import Digraph

class Visualizer:
    def __init__(self, recorder):
        self.recorder = recorder

    def build_graph(self):
        dot = Digraph()

        for i, event in enumerate(self.recorder.events):
            node_name = f"{event['function']}_{i}"

            label = f"""
            {event['function']}
            before: {event['before']}
            after: {event['after']}
            """

            dot.node(node_name, label)

            if i > 0:
                prev = f"{self.recorder.events[i-1]['function']}_{i-1}"
                dot.edge(prev, node_name)

        return dot
```

---

### 6. Connecting the Components

```python
recorder = Recorder()
tracker = Tracker(recorder)

@tracker.track
def A(x):
    return B(x + 1)

@tracker.track
def B(y):
    return y * 2

A(5)

viz = Visualizer(recorder)
viz.build_graph().render("output", view=True)
```

---

### 7. Version 1 Capabilities

The initial version provides a visualization of:
*   A → B transition
*   Input state
*   Output state
*   Execution order

---

### 8. Strategic Extensions

#### A. Call Stack Tracking
- Implement `self.call_stack.append(func.__name__)` to derive true parent-child relationships.

#### B. Variable-Level State Tracking
- Instead of just tracking arguments and results, track `locals()` using the `inspect` module or frame access.

#### C. Invariant Detection
- Add checks (e.g., `assert balance >= 0`) and log violations for visualization.

#### D. Static and Dynamic Merging
- Combine Analyzer output (possible calls) with Tracker output (actual calls) to visualize expected vs. actual system behavior.

---

### 9. Implementation Challenges

*   **State Explosion:** Excessive data can result in unreadable graphs.
*   **Introspection Limits:** Tracking local variables in Python is complex.
*   **Performance Overhead:** Decorators introduce execution delays.

---

### Final Mental Model

You are building a **State Graph Engine**.

*   **Nodes:** Function execution instances.
*   **Edges:** Transitions.
*   **Data:** Before/after state.
