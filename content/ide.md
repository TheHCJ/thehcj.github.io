---
title: "Harley's Python IDE"
date: 2025-02-24
layout: "single"
markup: "html"
---
<!-- BEGIN: Python IDE -->

<!-- CodeMirror CSS -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.65.13/codemirror.min.css">
<!-- CodeMirror Hint CSS -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.65.13/addon/hint/show-hint.min.css">

<!-- CodeMirror JavaScript Libraries -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.65.13/codemirror.min.js" defer></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.65.13/mode/python/python.min.js" defer></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.65.13/addon/hint/show-hint.min.js" defer></script>

<!-- Pyodide -->
<script src="https://cdn.jsdelivr.net/pyodide/v0.23.2/full/pyodide.js" defer></script>

<!-- Supabase JS -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2.25.0/dist/umd/supabase.js"></script>

<style>
  /* Update to make the code editor fit into a Hugo blog layout */
  .ide-container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
  }

  #editor {
    background-color: #f5f5f5;
    font-family: 'Fira Mono', Courier New', Courier, monospace;
    color: #333;
  }

  #output {
    border: 1px solid #ddd;
    height: 10em;
    margin-top: 10px;
    background-color: #fafafa;
    padding: 10px;
    overflow-y: auto;
    white-space: pre-wrap;
    font-family: 'Fira Mono', 'Courier New', Courier, monospace;
    color: #333;
  }

  button {
    margin-top: 10px;
    padding: 10px 20px;
    border-radius: 5px;
    border: none;
    background-color: #0077cc;
    color: white;
    cursor: pointer;
  }

  button:hover {
    background-color: #005fa3;
  }

  input[type="text"] {
    padding: 10px;
    margin-top: 10px;
    width: 100%;
    max-width: 300px;
    border: 1px solid #ddd;
    border-radius: 4px;
  }

  .editor-footer {
    display: flex;
    justify-content: space-between;
    margin-top: 20px;
  }

  .editor-footer button {
    width: 48%;
  }
</style>

<div class="ide-container">
  <h1>Python IDE</h1>
  <div id="editor"></div>
  
  <div class="editor-footer">
    <button id="run-btn">Run Code</button>
    <button id="save-btn">Save Code</button>
  </div>
  
  <h2>Output:</h2>
  <!-- Allow user input in the output window -->
  <div id="output" contenteditable="true"></div>
</div>

<script>
document.addEventListener("DOMContentLoaded", () => {
  // --- Global variable for Python keywords ---
  let pythonKeywords = [];

  // --- Utility Functions ---
  const outputEl = document.getElementById('output');
  function output(text) {
    outputEl.textContent += text + "\n";
    outputEl.scrollTop = outputEl.scrollHeight; // Auto-scroll
  }
  function clearOutput() {
    outputEl.textContent = "";
  }

  // Expose output globally for Pyodide's Python code.
  window.output = output;

  // --- Initialize CodeMirror ---
  const editor = CodeMirror(document.getElementById('editor'), {
    value: '# Write your Python code here\nprint("Hello, world!")',
    mode: 'python',
    lineNumbers: true,
    extraKeys: {
      "Ctrl-Space": "autocomplete"
    },
    theme: 'default'
  });

  // --- Autocomplete using dynamic Python keywords ---
  CodeMirror.registerHelper("hint", "python", function(editor, options) {
    const cur = editor.getCursor();
    const token = editor.getTokenAt(cur);
    const start = token.start;
    const end = cur.ch;
    const word = token.string.slice(0, end - start);
    // Use the dynamically loaded list if available; else fallback to a minimal list.
    const keywords = pythonKeywords.length ? pythonKeywords : [
      "False", "None", "True", "and", "as", "assert", "async", "await",
      "break", "class", "continue", "def", "del", "elif", "else", "except",
      "finally", "for", "from", "global", "if", "import", "in", "is", "lambda",
      "nonlocal", "not", "or", "pass", "raise", "return", "try", "while", "with", "yield"
    ];
    const completions = keywords.filter(kw => kw.startsWith(word));
    return {
      list: completions,
      from: CodeMirror.Pos(cur.line, start),
      to: CodeMirror.Pos(cur.line, end)
    };
  });

  editor.on("inputRead", (cm, change) => {
    if (change.text[0] && /[\w\.]/.test(change.text[0])) {
      cm.showHint({ hint: CodeMirror.hint.python });
    }
  });

  // --- Load Pyodide and Retrieve Python Keywords ---
  let pyodide = null;
  async function initPyodide() {
    output("Loading Python interpreter...");
    try {
      pyodide = await loadPyodide({ indexURL: "https://cdn.jsdelivr.net/pyodide/v0.23.2/full/" });
      output("Python interpreter loaded.");
      // Redirect Python's stdout/stderr to our output window.
      await pyodide.runPythonAsync(`
import sys
import js

class StdoutCatcher:
    def write(self, s):
        if s.strip():
            js.window.output(s)
    def flush(self):
        pass

sys.stdout = sys.stderr = StdoutCatcher()
      `);
      // Get the full list of Python keywords from the built-in module.
      pythonKeywords = await pyodide.runPythonAsync('import keyword; keyword.kwlist');
    } catch (err) {
      output("Error loading Pyodide: " + err);
    }
  }
  initPyodide();

  // --- Button Handlers ---

  // Run Python code.
  document.getElementById('run-btn').addEventListener('click', async () => {
    if (!pyodide) {
      output("Python interpreter not loaded yet.");
      return;
    }
    clearOutput();
    const code = editor.getValue();
    try {
      const result = await pyodide.runPythonAsync(code);
      if (result !== undefined) {
        output(result.toString());
      }
    } catch (err) {
      output("Error: " + err);
    }
  });
  
  // Initialize Supabase client outside the DOMContentLoaded event listener
const supabaseClient = supabase.createClient(
  'https://gjjevzqihdzldkxlucjw.supabase.co',
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImdqamV2enFpaGR6bGRreGx1Y2p3Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NDA0MzExMjYsImV4cCI6MjA1NjAwNzEyNn0.6ssXGyJch26dIq0zAcDgJfrnIo6JYi7Uah10A5iEcSM'
);

// Add error handling for clipboard operations
async function saveCode(code) {
  try {
    const { data, error } = await supabaseClient
      .from('code_snippets')
      .insert([{ 
        code: code,
        created_at: new Date().toISOString()
      }])
      .select();

    if (error) throw error;

    if (data && data[0]) {
      const codeId = data[0].id;
      const shareUrl = `${window.location.origin}/ide?id=${codeId}`;
      output(`Code saved successfully! Share using this URL:\n${shareUrl}`);
      
      try {
        await navigator.clipboard.writeText(shareUrl);
        output('Share URL copied to clipboard!');
      } catch (clipboardError) {
        output('Share URL (please copy manually):\n' + shareUrl);
      }
    }
  } catch (error) {
    output(`Error saving code: ${error.message}`);
    throw error; // Re-throw to be handled by the caller
  }}

});
</script>
