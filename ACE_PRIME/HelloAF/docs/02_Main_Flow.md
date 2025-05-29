# Flow of `ACE_PRIME/HelloAF/src/main.py`

This document details the process flow of the `main.py` script, which serves as the primary entry point for running individual ACE resources within the HelloAF project.

## Overview

The `main.py` script is responsible for dynamically loading and starting a specific "ACE resource" based on runtime environment variable configurations. It searches for the resource code in predefined locations, instantiates it, calls its `start_resource()` method, and then keeps the main program thread alive.

## 1. Initial Setup

When the script begins, it performs the following initializations:

*   **Imports:**
    *   `os`: For accessing environment variables (e.g., `ACE_RESOURCE_NAME`, `ACE_RESOURCE_SUBDIRECTORY`).
    *   `time`: For the `time.sleep()` function, used to keep the main thread alive.
    *   `importlib`: For dynamically importing Python modules at runtime. This is crucial for loading resources based on names.
    *   `ace.util`: Contains utility functions, notably `util.snake_to_class()`, which converts resource names from `snake_case` to `ClassNameCase`.
    *   `ace.logger.Logger`: A custom logger from the ACE framework.
*   **Logger Initialization:**
    *   `logger = Logger(os.path.basename(__file__))`: An instance of the `Logger` is created, using the filename (`main.py`) to identify log message origins.
*   **Resource Loader Directories:**
    *   `RESOURCE_LOADER_DIRECTORIES = ["custom", "core"]`: This list defines the primary base paths (interpreted as Python package paths) where the script will search for resource modules. "custom" resources are prioritized over "core" resources.

## 2. `load_resource(resource_class_name, import_path)` Function

This utility function handles the dynamic loading of a Python class from a given module.

*   **Purpose:** To abstract the mechanics of importing a module and then retrieving a specific class from it.
*   **Arguments:**
    *   `resource_class_name` (str): The name of the class to load (e.g., "MyResource").
    *   `import_path` (str): The full Python import path for the module containing the class (e.g., "ace.resources.custom.hello_layers.my_resource").
*   **Process:**
    1.  It attempts to import the module using `importlib.import_module(import_path)`.
    2.  If the module is imported successfully, it tries to get the class attribute `resource_class_name` from the module using `getattr(module, resource_class_name)`.
*   **Return Value:**
    *   Returns the class object if successful.
    *   Returns `None` if the module cannot be imported or the class attribute is not found, logging appropriate errors.

## 3. `loader(resource_name)` Function

This is the core function for finding, loading, and initializing the specified ACE resource.

*   **Purpose:** To locate the Python class for the given `resource_name`, create an instance, and call its `start_resource()` method.
*   **Argument:**
    *   `resource_name` (str): The snake_case name of the resource to load (e.g., "my_layer").
*   **Process:**
    1.  **Name Conversion:** `resource_class_name = util.snake_to_class(resource_name)` converts the input to ClassNameCase.
    2.  **Subdirectory Determination:** `subdirectory = os.environ.get("ACE_RESOURCE_SUBDIRECTORY") or "hello_layers"` gets the target subdirectory, defaulting to "hello_layers".
    3.  **Prioritized Search:**
        *   Iterates through `RESOURCE_LOADER_DIRECTORIES` (`"custom"`, then `"core"`).
        *   For each, constructs an `import_path` (e.g., `f"ace.resources.{directory}.{subdirectory}.{resource_name}"`).
        *   Calls `load_resource()` with the class name and import path.
        *   If the class is found, the search stops.
    4.  **Framework Fallback:** If not found in "custom" or "core", it tries a fallback path: `f"ace.framework.resources.{resource_name}"`.
    5.  **Failure Handling:** If the class is still not found, it logs an error and returns `False`.
    6.  **Instantiation and Start:** If the class is found:
        *   An instance is created: `resource = resource_class()`.
        *   The resource is started: `resource.start_resource()`.
        *   Returns `True` on success.
    7.  **Exception Handling:** A `try...except` block catches any errors during the process, logs them, and the function returns `None` in such cases.
*   **Return Value:**
    *   `True`: If the resource is successfully loaded and started.
    *   `False`: If the resource class is not found.
    *   `None`: If an unexpected exception occurs.

## 4. `main()` Function

This function orchestrates the overall script execution.

*   **Purpose:** To get the desired resource name from the environment, trigger its loading via the `loader` function, and then keep the program alive.
*   **Process:**
    1.  **Get Resource Name:** `ace_resource_name = os.getenv("ACE_RESOURCE_NAME")` fetches the target resource name from the environment.
    2.  **Load Resource:** `result = loader(ace_resource_name)` calls the loader.
    3.  **Keep Alive:** If `loader` returns `True` (success), it enters an infinite loop: `while result: time.sleep(1000)`. This pause keeps the main thread running, allowing any background tasks or threads started by the resource to continue operating.

## 5. Script Execution Block (`if __name__ == "__main__":`)

This standard Python construct defines the entry point when the script is run directly.

*   **Behavior:** When `main.py` is executed (e.g., `python src/main.py`), the `__name__` variable is set to `"__main__"`. The condition `if __name__ == "__main__":` becomes true, and the `main()` function is called, initiating the resource loading process. If the script were imported as a module into another Python file, `__name__` would be different, and `main()` would not automatically execute.

This flow allows `main.py` to be a versatile launcher for various ACE resources, configured entirely through environment variables.
