# Proposal for Documentation Automation Script

## 1. Goal

The primary goal of this automation script is to assist in keeping the Markdown documentation located in the `/documents/` directory synchronized with the evolving codebase. It aims to improve the accuracy of the documentation by verifying module paths, class names, and function/method names referenced in the documents, thereby reducing stale or incorrect information.

## 2. Proposed Features / Functionalities

### a. Link Validation

*   **Internal Link Checker:**
    *   Verify that all relative links within the documentation (e.g., `[User Authentication](./feature-user-authentication.md)` in `summary.md`) point to existing `.md` files within the `/documents/` directory and its subdirectories.
    *   Report broken or incorrect internal links.
*   **External Code Link Checker (Future Enhancement):**
    *   If documentation starts including direct links to code on a repository platform (e.g., GitHub permalinks), this feature could be added to check if those links are still valid (return a 200 OK). This is lower priority as it depends on external services and specific linking conventions.

### b. Code Reference Validation (Core Feature)

This is the core functionality, designed to ensure that technical entities mentioned in the documentation accurately reflect the codebase.

*   **AST Parsing:**
    *   Leverage Abstract Syntax Trees (AST) for Python files to reliably identify class definitions, function definitions (including methods within classes).
    *   For future expansion to other languages, similar AST parsing or language server capabilities could be integrated.
*   **Configuration File:**
    *   A central configuration file (e.g., `doc_code_mapping.yaml` or `doc_code_mapping.json`) will map documentation files to the source code directories or specific files they intend to cover.
    *   **Example Configuration Entry:**
        ```yaml
        - doc_file: "documents/feature-user-authentication.md"
          code_paths:
            - "api/services/account_service.py"
            - "api/controllers/console/auth/"
            - "api/libs/passport.py"
        - doc_file: "documents/feature-application-management.md"
          code_paths:
            - "api/services/app_service.py"
            - "api/controllers/console/app/app.py"
            - "api/models/model.py" # (Focus on App, AppModelConfig)
            - "api/core/app/apps/"
            - "api/core/app/app_config/"
        # ... other mappings
        ```
*   **Extraction & Comparison Logic:**
    1.  For each entry in the configuration file:
        *   The script will parse all Python files found in the specified `code_paths` using AST to extract a list of all public class names and public function/method names.
        *   It will then read the content of the `doc_file`.
    2.  The script will scan specific sections of the Markdown file, primarily:
        *   "Related Modules" tables (looking at the "Path" and "Major Classes/Functions" columns).
        *   "Data Flow" sections (looking for mentions of classes and functions in the descriptions).
    3.  **Validation Checks & Reporting:**
        *   **Docs Mismatch (Not Found in Code):** Identify class/function names mentioned in the documentation (in the designated sections) but not found in the extracted list from the mapped source code. This could indicate typos, renamed items, or outdated documentation.
        *   **Code Mismatch (Not Found in Docs):** Identify public classes/functions present in the mapped source code files but *not* mentioned in the relevant sections of the documentation. This could highlight new features or components that need to be documented.
        *   **Path Verification:** For paths mentioned in "Related Modules" tables, verify if they exist in the codebase (using simple file/directory existence checks).

### c. Mermaid Diagram Helper (Optional Enhancement - Complex)

*   **Dependency Suggestion:**
    *   For specified core modules/classes, parse Python `import` statements.
    *   Analyze function/method calls between these core modules.
    *   Suggest potential dependencies or connections that might be useful to include in Mermaid diagrams.
    *   *Caveat:* This is complex due to dynamic calls, aliasing, and the difficulty of accurately representing high-level architectural relationships from static analysis alone. It would likely be a heuristic-based suggestion tool.

### d. New Module Detection

*   The script can be configured with a list of "core" or "service" directories (e.g., `api/services/`, `api/core/workflow/nodes/`).
*   It can check if new Python files (modules) have been added to these directories that are not yet referenced in the `doc_code_mapping.yaml` configuration file.
*   This can prompt developers to either create new documentation for these modules or add them to the scope of existing documents.

## 3. Workflow

*   **Developer Usage (Local):**
    1.  A developer makes changes to the codebase and/or the documentation.
    2.  Before committing, they run the automation script locally: `python run_doc_automation.py` (or similar).
    3.  The script outputs a report to the console or a file, highlighting:
        *   Broken internal links.
        *   Code references in docs not found in the current codebase (potential typos or removals).
        *   Public code entities in mapped files not found in docs (potential documentation gaps).
        *   Suggestions for new modules in core directories that might need documentation.
    4.  The developer reviews the report and makes necessary corrections to code comments, documentation, or the script's configuration file.
*   **CI Integration (Automated Check):**
    1.  The script can be integrated into the Continuous Integration (CI) pipeline.
    2.  On every pull request or push to main branches, the script runs.
    3.  If validation errors (e.g., broken links, critical code mismatches) are found, the CI check can fail, prompting the developer to fix the documentation before merging. Non-critical suggestions (like new items to document) could be warnings.

## 4. Output

The script's output would primarily be a text-based report, which could include:

*   **Link Validation:**
    *   List of broken internal links and the files they are in.
*   **Code Reference Validation:**
    *   For each documented file:
        *   "Outdated References": List of class/function names found in the doc but not in the mapped code.
        *   "Missing Documentation": List of public class/function names found in the mapped code but not in the doc.
        *   "Incorrect Paths": List of module paths in "Related Modules" tables that don't exist.
*   **New Module Detection:**
    *   List of new modules in monitored directories not yet mapped in the configuration.

## 5. Limitations

*   **Not a Replacement for Human Review:** The script is a helper to catch common structural and naming issues. It cannot verify the semantic correctness, clarity, or completeness of the descriptions themselves. Human review remains essential.
*   **AST Focus on Definitions:** AST parsing is excellent for finding definitions (classes, functions). Accurately finding all *mentions* of these entities in prose or complex data flow descriptions within Markdown can be challenging and might rely on regex or heuristic string matching, which can have false positives/negatives. The focus should be on structured parts like "Related Modules" tables first.
*   **Configuration Dependent:** The accuracy of code reference validation heavily depends on the correctness and completeness of the `doc_code_mapping.yaml` file. If this mapping is wrong or incomplete, the script's output will be misleading.
*   **Dynamic Code Behavior:** The script will primarily rely on static analysis. It won't understand dynamically generated classes/functions or complex inheritance patterns easily.
*   **Scope of "Public":** Determining what constitutes a "public" class or function that *should* be documented might need clear conventions (e.g., underscore prefixes for private members). The script would need to follow these conventions.
*   **Mermaid Helper Complexity:** The Mermaid diagram helper is a more speculative feature and would be significantly harder to implement reliably.
