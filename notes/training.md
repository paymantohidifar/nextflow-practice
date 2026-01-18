# NextFlow Basic Training

## Part 1: Run Basic operations

The training uses a basic "Hello World" example to demonstrate essential Nextflow operations. Direct terminal commands like `echo 'Hello World!'` and `echo 'Hello World!' > output.txt` are used as a warm-up to show simple text output and writing to a file.

A Nextflow workflow is launched using the command `nextflow run 1-hello.nf --greeting 'Hello World!'`. The console output confirms successful execution, showing a line like `[a3/7be2fa] sayHello | 1 of 1 ✔`, indicating the `sayHello` process ran successfully.

Workflows are often configured to publish final outputs to a designated directory, such as `results/`, which contains the final output files (e.g., `output.txt`).

Nextflow creates a unique **task directory** for every process execution under the `work/` directory.
* The task directory path is visible in the console output (e.g., `[a3/7be2fa]`).
* This directory contains the original output file, as well as crucial log and helper files:
    * **.command.sh**: The main command executed by the process.
    * **.command.out** / **.command.err**: Standard output and error messages.
    * **.exitcode**: The exit code of the command.

### Examine a Workflow Starter Script

A Nextflow script is composed of one or more `process` blocks and a `workflow` block.
* **Process:** A `process` block (e.g., `sayHello`) describes a single step in the pipeline. It defines `input` variables (e.g., using `val`), declares expected `output` files (e.g., using `path`), and contains the `script` to execute.
* **Workflow:** The `workflow` block defines the dataflow logic, connecting various processes. It calls processes (e.g., `sayHello(params.greeting)`), passing inputs.
* **Command-Line Parameters:** The `params` system allows passing values from the command line using double-dash arguments (e.g., `--greeting` corresponds to `params.greeting` in the script).

### Manage Workflow Executions

* **Resuming Runs:** Adding the `-resume` flag to the `nextflow run` command skips processes that have already completed successfully with the same code, settings, and inputs.
    * This is useful for rapid iteration during development and recovering from production failures.
    * The console output indicates a skipped process with `cached:`.
* **Inspecting Logs:** The `nextflow log` command displays a history of all workflow executions launched from the current directory, including the session ID and full command.
* **Cleaning Work Directories:** The `nextflow clean` subcommand is used to delete older work subdirectories.
    * Use the `-n` (dry run) flag first to check what will be deleted, and then the `-f` (force) flag to execute the deletion.
    * Deleting work directories breaks Nextflow's ability to resume those runs, so published outputs should be saved elsewhere.

---

## Part 2: Running Nextflow Pipelines

### Processing Multiple Inputs Efficiently

* **Channels for Parallelism:** Nextflow uses channels (queues) to handle multiple input data points efficiently and enable processing them in parallel. Data from an input file (e.g., `greetings.csv`) is loaded into a channel using functions like `channel.fromPath()`, followed by **operators** (e.g., `.splitCsv()` and `.map{...}`) to parse and extract the relevant data elements. The workflow then calls the process (e.g., `sayHello(greeting_ch)`) once for each element in the channel.
* **Isolation and Naming:** Nextflow creates a separate task directory under `work/` for each individual process call, ensuring isolation. Output files must be uniquely named (e.g., using the input value in the name) to prevent collisions when published to the `results/` directory.


Here is a brief description of these operators.

#### `.splitCsv()`

| Feature | Description |
| :--- | :--- |
| **Purpose** | Parses an input file (usually provided via `channel.fromPath()`) into a structure of rows and columns, typically an array of arrays. |
| **Context** | Applied directly after reading a path channel, instructing Nextflow to treat the file's contents as Comma Separated Values (CSV). |
| **Example Use** | Used on a CSV file to turn its contents into structured data for downstream processing. |
| **Input $\rightarrow$ Output** | A file content string $\rightarrow$ An array, where each element is a row (e.g., `[['Hello', 'English', 123], ['Bonjour', 'French', 456]]`). |

#### `.map { ... }`

| Feature | Description |
| :--- | :--- |
| **Purpose** | Applies a custom transformation or function to each individual item (element) emitted by the channel. |
| **Context** | Used to extract a specific piece of information from a structured element. It takes a closure (the code inside the curly braces `{}`) written in the Groovy language. |
| **Example Use** | After using `.splitCsv()`, you use `.map { line -> line[0] }` to take only the first element (`[0]`) from each row, extracting only the greeting string. |
| **Input $\rightarrow$ Output** | Multiple array elements $\rightarrow$ Multiple transformed scalar values (e.g., `['Hello', 'English', 123]` $\rightarrow$ `'Hello'`). |

### Multi-Step Workflows and Data Flow

* **Chaining Processes:** Multi-step workflows connect multiple processes. Data flows from one process to the next by referencing the output channel of the preceding process (e.g., `convertToUpper(sayHello.out)`). 
* **Operators for Logic:** Operators are used to customize the flow logic between steps.
    * The **`.collect()`** operator is crucial for gathering outputs from multiple parallel process calls into a single channel element, allowing a downstream process (e.g., `collectGreetings`) to run only once on the combined data.

#### `.collect()`

| Feature | Description |
| :--- | :--- |
| **Purpose** | Gathers **all items** emitted by a channel over time and packages them into a **single new item** (usually a list or array). |
| **Context** | Essential for changing the execution mode from parallel to sequential aggregation. It ensures a downstream process only runs **once** after all upstream parallel processes have completed. |
| **Example Use** | Used to collect all the separate uppercased greeting files generated by parallel processes into a single list of file paths, which is then fed into a final process (`collectGreetings`) that merges them all. |
| **Input $\rightarrow$ Output** | Multiple individual elements (e.g., three separate file paths) $\rightarrow$ A single array containing all those elements. |

* **Visualization:** The **DAG preview** tool in the VSCode Nextflow extension can help visualize the connections and inputs of processes in the pipeline.

### Modularized Pipelines

* **Modules for Reusability:** Nextflow pipelines benefit from **modularization**, where single process definitions are encapsulated in standalone files called **modules**.
* **Code Structure:** A module (e.g., `sayHello.nf`) contains only the process definition. The main workflow file uses an **`include`** statement (e.g., `include { sayHello } from './modules/sayHello.nf'`) to import the process, making the code more maintainable and reusable.
* **Resuming with Modules:** Nextflow's **`-resume`** feature still works even when code is split into modules, as it caches based on the final generated job script, not the source file structure.

### Using Containerized Software

* **Containers for Reproducibility:** **Containers** (like Docker) include all necessary code, libraries, and settings to run a tool, solving dependency management and ensuring **reproducibility** across different systems.
* **Direct Container Use:** You can manually interact with containers using `docker pull` (to download the image) and `docker run` (to execute commands inside the isolated container).
* **Nextflow Integration:** Nextflow automatically handles container management when you specify the container URI using the **`container`** directive within a process definition and enable the technology (e.g., `docker.enabled = true` in `nextflow.config`). 
* **Automatic Launch:** Nextflow generates the appropriate `docker run` command (including volume mounting) to execute the task inside the container, saving the user from manual setup.

### Channel Operators Overview

The channel operators you mentioned are essential Nextflow tools for **transforming and managing the flow of data** between processes. They allow you to shift from handling individual inputs to managing large collections or modifying how outputs are grouped. 

