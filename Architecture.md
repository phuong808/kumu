# Kumu (R Package) Repository - Complete File Structure and Architecture

## Repository Root: `/kumu/`

### 📦 R Package Core: `R/` Directory
**Purpose:** Core R functions for causal discovery via Tetrad-CMD

#### Main Interface Module
- **`tetrad.R`** (52 lines)
  - Main function: `tetrad()`
  - Purpose: Execute Tetrad-CMD via external Java process
  - Implementation:
    ```r
    tetrad <- function(tetrad_cmd_path, data_flags, knowledge_flags,
                      algorithm_flags, score_flags, bootstrapping_flags) {
      # Uses system2() to spawn Java process
      # Command: java -jar -Xms4096M -Xmx6144M tetrad_cmd_path [flags...]
      # Returns: stdout as character vector
    }
    ```
  - Parameters:
    - `tetrad_cmd_path`: Path to tetrad-cmd.jar file
    - `data_flags`: From score.R functions
    - `knowledge_flags`: From knowledge.R (optional)
    - `algorithm_flags`: From algorithm.R functions
    - `score_flags`: From score.R functions
    - `bootstrapping_flags`: From bootstrapping.R functions
  - Memory allocation: -Xms4096M -Xmx6144M (4-6GB heap)
  - Returns: Raw stdout output from Java process

#### Algorithm Configuration Module
- **`algorithm.R`** (186 lines)
  - Functions for algorithm selection and configuration
  - Each returns character vector of command-line flags
  
  **Available Algorithms:**
  
  1. **`algorithm_boss()`**
     - BOSS (Best Order Score Search)
     - Parameters:
       - `num_starts` (default=1): Number of random restarts
       - `num_threads` (default=1): Parallel threads
       - `time_lag` (default=0): Time-series lag
       - `use_bes` (logical): Use backward elimination
       - `use_data_order` (logical): Use data variable order
       - `verbose` (logical): Print detailed output
     - Returns: `c("--algorithm", "boss", "--numStarts", num_starts, ...)`
  
  2. **`algorithm_fges()`**
     - FGES (Fast Greedy Equivalence Search)
     - Parameters:
       - `max_degree` (default=1000): Maximum node degree
       - `faithfulness_assumed` (logical): Assume faithfulness
       - `symmetric_first_step` (logical): Symmetric forward search
       - `time_lag` (default=0): Time-series lag
       - `verbose` (logical): Verbose output
       - `parallelized` (logical): Use parallelization
       - `meek_verbose` (logical): Meek rule verbosity
     - Returns: Flag vector for FGES configuration
  
  3. **`algorithm_pc()`**
     - PC (Peter-Clark) Algorithm
     - Parameters:
       - `stable` (logical): Use stable PC
       - `concurrent_fci` (logical): Concurrent FAS
       - `collision_rule` (0-3): Collision resolution
       - `conflict_rule` (1-6): Conflict resolution
       - `depth` (default=-1): Max conditioning set size
       - `fas_rule` (1-2): FAS adjacency rule
       - `verbose` (logical)
     - Returns: PC configuration flags
  
  4. **`algorithm_cpc()`**
     - CPC (Conservative PC)
     - Similar parameters to PC
     - Conservative orientation rules
  
  5. **`algorithm_fci()`**
     - FCI (Fast Causal Inference)
     - For latent confounders and selection bias
     - Parameters include depth, PAG completion
  
  6. **`algorithm_grasp()`**
     - GRaSP (Greedy Relaxations of Sparsest Permutation)
     - Parameters:
       - `ordered_alg`: Sub-algorithm (BOSS, FGES)
       - `score`: Scoring function
       - `num_starts`: Random restarts
       - `depth`: Search depth
       - `use_bes`: Backward elimination
       - `verbose`
     - Returns: GRaSP configuration

#### Scoring Function Module
- **`score.R`** (53 lines)
  - Functions for score configuration
  
  **Available Scores:**
  
  1. **`score_sem_bic()`**
     - SEM-BIC Score (Linear Gaussian)
     - Formula: BIC = 2L - ck log N
       - L = likelihood
       - c = penalty_discount
       - k = parameters
       - N = sample size
     - Parameters:
       - `penalty_discount` (default=0.0): Penalty multiplier
       - `sem_bic_rule`: Lambda type
         - 1 = Chickering rule
         - 2 = Nandy rule
       - `sem_bic_structure_prior` (default=0): Structure prior
       - `precompute_covariances` (logical): Cache covariances
     - Returns: `c("--score", "sem-bic-score", "--penaltyDiscount", ...)`
     - Use case: Continuous Gaussian data
  
  2. **`score_bdeu()`**
     - BDeu Score (Bayesian Dirichlet)
     - For discrete data
     - Parameters:
       - `sample_prior`: Prior equivalent sample size
       - `structure_prior`: Structure prior weight
     - Returns: BDeu configuration flags
  
  3. **`score_bic()`**
     - Standard BIC Score
     - Parameters: penalty_discount
     - Simpler than SEM-BIC
  
  4. **`score_conditional_gaussian()`**
     - For mixed continuous/discrete data
     - Parameters:
       - `penalty_discount`
       - `discretize`: Force discretization
       - `structure_prior`
  
  **Data I/O Function:**
  - **`data_io()`**
    - Specifies input data file
    - Parameters:
      - `data_type`: "continuous", "discrete", "mixed"
      - `data_path`: Path to CSV file
      - `delimiter` (default=","): Column separator
    - Returns: `c("--data-type", data_type, "--dataset", data_path, ...)`

#### Bootstrapping Module
- **`bootstrapping.R`** (60+ lines)
  - Configure bootstrap resampling
  
  **Main Function:**
  - **`bootstrapping()`**
    - Parameters:
      - `number_resampling`: Number of bootstrap iterations
      - `percent_resample_size`: Percentage of data to sample (e.g., 90)
      - `resampling_with_replacement` (logical): With/without replacement
      - `add_original_dataset` (logical): Include original in ensemble
      - `resampling_ensemble`:
        - 0 = Preserved (edges present in all bootstraps)
        - 1 = Highest (majority voting)
        - 2 = Majority (> 50% threshold)
      - `save_bootstrap_graphs` (logical): Save individual graphs
      - `seed`: Random seed for reproducibility
      - `verbose` (logical)
    - Returns: `c("--bootstrapping", "--numberResampling", number, ...)`
    - Note: Algorithm runs (1 + number_resampling) times total
  
  **Restarts Function:**
  - **`restarts()`**
    - For algorithms with random initialization
    - Parameters: `number_of_starts`
    - Returns: `c("--restarts", number_of_starts)`

#### Knowledge (Prior Information) Module
- **`knowledge.R`**
  - Specify prior causal knowledge
  
  **Main Function:**
  - **`knowledge_file_path()`**
    - Parameters: `knowledge_file_path` - path to knowledge file
    - Returns: `c("--knowledge", knowledge_file_path)`
  
  **Knowledge File Format:**
  ```
  /knowledge
  forbiddirect
  X1 X2
  X3 X4
  
  requiredirect
  X5 X6
  
  addtemporal
  0 X1 X2 X3
  1 X4 X5
  ```
  - Constraints:
    - `forbiddirect`: Forbidden edges
    - `requiredirect`: Required edges
    - `addtemporal`: Time-ordering tiers

#### Graph Parsing Module
- **`graph.R`**
  - Parse Tetrad output into R structures
  
  **Main Function:**
  - **`parse_graph()`**
    - Parses stdout from tetrad() function
    - Extracts graph string representation
    - Converts to R-friendly format
    - Returns: Graph structure (edge list or adjacency matrix)

#### Utility Modules
- **`data.R`**
  - Data manipulation helpers
  - Preprocessing functions
  - Data validation

- **`noise.R`**
  - Noise model utilities
  - Statistical functions
  - Variance estimation

---

### 📚 Documentation: `man/` Directory
**R Documentation Files (.Rd format)**

Each function has detailed documentation:

1. **`algorithm_boss.Rd`**
   - Usage, parameters, examples
   - References to academic papers
   - Links to Tetrad Javadocs

2. **`algorithm_fges.Rd`**
   - FGES documentation
   - Parameter descriptions
   - Performance characteristics

3. **`bootstrapping.Rd`**
   - Bootstrap methodology
   - Ensemble methods explained
   - Statistical theory

4. **`data_io.Rd`**
   - Data input/output
   - File format requirements
   - Supported data types

5. **`knowledge_file_path.Rd`**
   - Prior knowledge format
   - Constraint syntax
   - Examples

6. **`parse_graph.Rd`**
   - Graph output parsing
   - Return value structure
   - Conversion utilities

7. **`restarts.Rd`**
   - Random restart documentation
   - When to use restarts
   - Performance impact

8. **`score_sem_bic.Rd`**
   - SEM-BIC theory
   - Parameter tuning guide
   - Mathematical formulation

9. **`tetrad.Rd`**
   - Main interface documentation
   - Complete workflow examples
   - Troubleshooting

**Figures Subdirectory:**
- `man/figures/` - Contains plots, diagrams, example outputs

---

### 📖 Tutorials: `vignettes/` Directory
**R Markdown Tutorial Documents**

1. **`boss_analysis.Rmd`**
   - Complete BOSS algorithm walkthrough
   - Real data analysis example
   - Interpretation of results
   - Visualization techniques
   - Compiled to: HTML, PDF documentation

2. **`issue_causal_analysis.Rmd`**
   - Software engineering issue analysis
   - Causal modeling of bug reports
   - Real-world application
   - Integration with data sources

3. **`null_variables.Rmd`**
   - Handling null/missing variables
   - Statistical considerations
   - Data imputation strategies
   - Impact on causal discovery

4. **`random_causality_threshold.Rmd`**
   - Threshold selection for causality
   - Statistical significance testing
   - Power analysis
   - Sensitivity analysis

**Vignette Features:**
- Executable R code blocks
- Narrative explanations
- Plots and visualizations
- Step-by-step instructions
- Best practices

---

### ⚙️ Package Configuration

#### Core Package Files

1. **`DESCRIPTION`** (33 lines)
   - Package metadata:
     ```
     Package: kumu
     Version: 0.0.0.9000
     Title: Kumu - Causal Analysis in R
     Description: R package for causal discovery via Tetrad
     Authors: Carlos Paradis (Maintainer), Michael Konrad, Rick Kazman
     License: MPL-2.0
     ```
   - Dependencies:
     - R >= 3.6.0
     - data.table >= 1.12.8
     - stringi >= 1.4.6
   - Suggests:
     - testthat (testing)
     - knitr, rmarkdown (vignettes)
   - URL: https://github.com/sailuh/kumu
   - ORCID identifiers for authors

2. **`NAMESPACE`**
   - Exported functions:
     ```r
     export(tetrad)
     export(algorithm_boss)
     export(algorithm_fges)
     export(algorithm_pc)
     export(algorithm_cpc)
     export(algorithm_fci)
     export(algorithm_grasp)
     export(score_sem_bic)
     export(score_bdeu)
     export(bootstrapping)
     export(restarts)
     export(knowledge_file_path)
     export(parse_graph)
     export(data_io)
     ```
   - Package imports from data.table, stringi

3. **`NEWS.md`**
   - Version history
   - Change log
   - Bug fixes
   - New features
   - Breaking changes

4. **`README.md`**
   - Quick start guide
   - Installation instructions
   - Example usage
   - Links to documentation
   - Citation information

5. **`LICENSE`**
   - Mozilla Public License 2.0
   - Full license text

6. **`_pkgdown.yml`**
   - Website configuration
   - Documentation site structure
   - Navigation menu
   - Theme settings
   - Used by pkgdown to build documentation website

7. **`kumu.Rproj`**
   - RStudio project file
   - IDE settings
   - Build configuration

8. **`.Rbuildignore`**
   - Files to ignore during R CMD build
   - Development files
   - Hidden files

9. **`.gitignore`**
   - Git ignore rules
   - Temporary files
   - Build artifacts

---

## 🏗️ System Architecture

### Complete Workflow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│ LAYER 1: R USER CODE                                         │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ R Script or RStudio Interactive Session                  │ │
│ │                                                           │ │
│ │ library(kumu)                                            │ │
│ │                                                           │ │
│ │ # Configure data input                                   │ │
│ │ data_flags <- data_io(                                   │ │
│ │     data_type = "continuous",                           │ │
│ │     data_path = "null_variable_dt.csv",                 │ │
│ │     delimiter = ","                                      │ │
│ │ )                                                        │ │
│ │ # Returns: c("--data-type", "continuous", "--dataset", ...)│ │
│ │                                                           │ │
│ │ # Configure scoring function                             │ │
│ │ score_flags <- score_sem_bic(                           │ │
│ │     penalty_discount = 2,                               │ │
│ │     sem_bic_rule = 1,                                   │ │
│ │     sem_bic_structure_prior = 0,                        │ │
│ │     precompute_covariances = TRUE                       │ │
│ │ )                                                        │ │
│ │ # Returns: c("--score", "sem-bic-score", ...)          │ │
│ │                                                           │ │
│ │ # Configure algorithm                                    │ │
│ │ algo_flags <- algorithm_fges(                           │ │
│ │     max_degree = 1000,                                  │ │
│ │     faithfulness_assumed = TRUE,                        │ │
│ │     symmetric_first_step = TRUE,                        │ │
│ │     parallelized = FALSE,                               │ │
│ │     verbose = TRUE                                       │ │
│ │ )                                                        │ │
│ │ # Returns: c("--algorithm", "fges", "--max-degree", ...)│ │
│ │                                                           │ │
│ │ # Configure bootstrapping                                │ │
│ │ boot_flags <- bootstrapping(                            │ │
│ │     number_resampling = 500,                            │ │
│ │     percent_resample_size = 90,                         │ │
│ │     seed = 32,                                          │ │
│ │     resampling_with_replacement = TRUE,                 │ │
│ │     add_original_dataset = TRUE,                        │ │
│ │     resampling_ensemble = 1                             │ │
│ │ )                                                        │ │
│ │ # Returns: c("--bootstrapping", "--numberResampling", ...)│ │
│ │                                                           │ │
│ │ # Optional: Prior knowledge                              │ │
│ │ knowledge_flags <- knowledge_file_path("knowledge.txt") │ │
│ │ # Returns: c("--knowledge", "knowledge.txt")            │ │
│ │                                                           │ │
│ │ # Execute Tetrad                                         │ │
│ │ result <- tetrad(                                        │ │
│ │     tetrad_cmd_path = "~/tetrad-cmd-1.6.0.jar",        │ │
│ │     data_flags = data_flags,                            │ │
│ │     knowledge_flags = knowledge_flags,                  │ │
│ │     algorithm_flags = algo_flags,                       │ │
│ │     score_flags = score_flags,                          │ │
│ │     bootstrapping_flags = boot_flags                    │ │
│ │ )                                                        │ │
│ │ # result is character vector of stdout                   │ │
│ │                                                           │ │
│ │ # Parse results                                          │ │
│ │ graph <- parse_graph(result)                            │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────┐
│ LAYER 2: R SYSTEM CALL (system2 function)                   │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ tetrad() Function Implementation:                        │ │
│ │                                                           │ │
│ │ system2(                                                 │ │
│ │     command = "java",                                    │ │
│ │     args = c(                                            │ │
│ │         "-jar",                                          │ │
│ │         "-Xms4096M",    # Min heap: 4GB                 │ │
│ │         "-Xmx6144M",    # Max heap: 6GB                 │ │
│ │         tetrad_cmd_path, # Path to JAR                  │ │
│ │         data_flags,      # e.g., "--data-type continuous"│ │
│ │         knowledge_flags, # e.g., "--knowledge file.txt" │ │
│ │         algorithm_flags, # e.g., "--algorithm fges"     │ │
│ │         score_flags,     # e.g., "--score sem-bic-score"│ │
│ │         bootstrapping_flags # e.g., "--bootstrapping"   │ │
│ │     ),                                                   │ │
│ │     stdout = TRUE  # Capture stdout                      │ │
│ │ )                                                        │ │
│ │                                                           │ │
│ │ Complete Command Example:                                │ │
│ │ java -jar -Xms4096M -Xmx6144M tetrad-cmd-1.6.0.jar \   │ │
│ │      --data-type continuous \                           │ │
│ │      --dataset null_variable_dt.csv \                   │ │
│ │      --delimiter , \                                     │ │
│ │      --algorithm fges \                                  │ │
│ │      --max-degree 1000 \                                │ │
│ │      --faithfulnessAssumed \                            │ │
│ │      --symmetricFirstStep \                             │ │
│ │      --score sem-bic-score \                            │ │
│ │      --penaltyDiscount 2 \                              │ │
│ │      --semBicRule 1 \                                   │ │
│ │      --bootstrapping \                                   │ │
│ │      --numberResampling 500 \                           │ │
│ │      --percentResampleSize 90 \                         │ │
│ │      --seed 32 \                                        │ │
│ │      --resamplingWithReplacement \                      │ │
│ │      --addOriginalDataset \                             │ │
│ │      --resamplingEnsemble 1 \                           │ │
│ │      --verbose                                           │ │
│ │                                                           │ │
│ │ Process Behavior:                                        │ │
│ │ • Spawns separate Java process                           │ │
│ │ • R waits for completion (blocking call)                 │ │
│ │ • Java prints to stdout                                  │ │
│ │ • R captures stdout and returns as character vector      │ │
│ │ • Process exit code checked (non-zero = error)           │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────┐
│ LAYER 3: SEPARATE JAVA PROCESS (tetrad-cmd.jar)             │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ Tetrad Command-Line Interface (Standalone Application)   │ │
│ │                                                           │ │
│ │ Main Execution Flow:                                     │ │
│ │ 1. Parse command-line arguments                          │ │
│ │    • Extract flags and values                            │ │
│ │    • Validate parameter combinations                     │ │
│ │                                                           │ │
│ │ 2. Load data file                                        │ │
│ │    • Read CSV file from disk                             │ │
│ │    • Convert to Tetrad DataSet object                    │ │
│ │    • Validate data types                                 │ │
│ │                                                           │ │
│ │ 3. Initialize scoring function                           │ │
│ │    • Create Score object (e.g., SemBicScore)            │ │
│ │    • Set parameters (penalty, structure prior, etc.)     │ │
│ │                                                           │ │
│ │ 4. Initialize algorithm                                  │ │
│ │    • Create Search object (e.g., Fges)                  │ │
│ │    • Set algorithm parameters                            │ │
│ │    • Apply knowledge constraints if provided             │ │
│ │                                                           │ │
│ │ 5. Run algorithm                                         │ │
│ │    • If bootstrapping:                                   │ │
│ │      - Create GeneralBootstrapTest                       │ │
│ │      - Generate 500 bootstrap samples                    │ │
│ │      - Run FGES on each sample                          │ │
│ │      - Apply ensemble method (Preserved/Highest/Majority)│ │
│ │    • If no bootstrapping:                                │ │
│ │      - Run FGES once on original data                   │ │
│ │    • Print progress to stdout (if verbose)               │ │
│ │                                                           │ │
│ │ 6. Format output                                         │ │
│ │    • Convert Graph to string representation              │ │
│ │    • Print graph structure to stdout                     │ │
│ │    • Print statistics (edges, runtime, etc.)             │ │
│ │    • Print edge list                                     │ │
│ │                                                           │ │
│ │ 7. Exit                                                  │ │
│ │    • Return exit code (0 = success)                      │ │
│ │                                                           │ │
│ │ Output Format (printed to stdout):                       │ │
│ │ Graph Edges:                                             │ │
│ │ 1. X1 --> X2                                            │ │
│ │ 2. X3 --- X4                                            │ │
│ │ 3. X5 o-o X6                                            │ │
│ │ ...                                                      │ │
│ │                                                           │ │
│ │ Graph Statistics:                                        │ │
│ │ Nodes: 138                                              │ │
│ │ Edges: 101                                              │ │
│ │ Runtime: 82.5 seconds                                   │ │
│ │ ...                                                      │ │
│ │                                                           │ │
│ │ Memory Management:                                       │ │
│ │ • Runs in separate JVM (not shared with R)               │ │
│ │ • Heap: 4GB - 6GB                                       │ │
│ │ • Garbage collected independently                        │ │
│ │ • Process terminates after completion                    │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                            ↓ stdout
┌──────────────────────────────────────────────────────────────┐
│ LAYER 4: RETURN TO R                                         │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ system2() returns stdout as character vector             │ │
│ │                                                           │ │
│ │ result <- c(                                             │ │
│ │     "Graph Edges:",                                      │ │
│ │     "1. var1 --> var2",                                 │ │
│ │     "2. var3 --- var4",                                 │ │
│ │     ...,                                                 │ │
│ │     "Graph Statistics:",                                 │ │
│ │     "Nodes: 138",                                       │ │
│ │     "Edges: 101"                                        │ │
│ │ )                                                        │ │
│ │                                                           │ │
│ │ # User can parse this output                             │ │
│ │ graph <- parse_graph(result)                            │ │
│ │ # Returns structured R object (data.frame or list)       │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔄 Example: Complete FGES Workflow

```r
# Install and load package
install.packages("kumu", repos = "https://github.com/sailuh/kumu")
library(kumu)

# Step 1: Prepare data flags
data_flags <- data_io(
    data_type = "continuous",              # Continuous data
    data_path = "null_variable_dt.csv",    # Input file
    delimiter = ","                         # CSV format
)
# Returns: c("--data-type", "continuous", "--dataset", 
#           "null_variable_dt.csv", "--delimiter", ",")

# Step 2: Configure SEM-BIC scoring
score_flags <- score_sem_bic(
    penalty_discount = 2,                   # Penalty multiplier
    sem_bic_rule = 1,                      # Chickering rule
    sem_bic_structure_prior = 0,           # No structure prior
    precompute_covariances = TRUE          # Cache covariances
)
# Returns: c("--score", "sem-bic-score", "--penaltyDiscount", "2",
#           "--semBicRule", "1", "--semBicStructurePrior", "0",
#           "--precomputeCovariances")

# Step 3: Configure FGES algorithm
algo_flags <- algorithm_fges(
    max_degree = 1000,                     # Max edges per node
    faithfulness_assumed = TRUE,           # Assume faithfulness
    symmetric_first_step = TRUE,           # Symmetric search
    time_lag = 0,                          # No time lag
    verbose = TRUE,                        # Print progress
    parallelized = FALSE,                  # Single-threaded
    meek_verbose = FALSE                   # Quiet Meek rules
)
# Returns: c("--algorithm", "fges", "--max-degree", "1000",
#           "--faithfulnessAssumed", "--symmetricFirstStep", ...)

# Step 4: Configure bootstrapping
boot_flags <- bootstrapping(
    number_resampling = 500,               # 500 bootstrap samples
    percent_resample_size = 90,            # 90% of data each sample
    seed = 32,                             # Random seed
    resampling_with_replacement = TRUE,    # Bootstrap sampling
    add_original_dataset = TRUE,           # Include original
    resampling_ensemble = 1,               # Preserved ensemble
    save_bootstrap_graphs = FALSE,         # Don't save graphs
    verbose = TRUE                         # Print progress
)
# Returns: c("--bootstrapping", "--numberResampling", "500", ...)

# Step 5: (Optional) Add prior knowledge
knowledge_flags <- knowledge_file_path("prior_knowledge.txt")
# Returns: c("--knowledge", "prior_knowledge.txt")
# Or use: knowledge_flags <- c() for no knowledge

# Step 6: Execute Tetrad
result <- tetrad(
    tetrad_cmd_path = "~/tetrad-cmd-1.6.0.jar",  # Path to JAR
    data_flags = data_flags,
    knowledge_flags = c(),                        # No knowledge
    algorithm_flags = algo_flags,
    score_flags = score_flags,
    bootstrapping_flags = boot_flags
)
# This runs:
# java -jar -Xms4096M -Xmx6144M ~/tetrad-cmd-1.6.0.jar \
#      [all flags concatenated]
# Takes ~90 seconds to complete
# Returns stdout as character vector

# Step 7: Parse results
graph <- parse_graph(result)
# Converts text output to structured R object

# Step 8: Analyze results
print(graph)
summary(graph)
plot(graph)  # If plotting functions available
```

---

## 🔑 Key Characteristics

### ✓ **External Process Architecture**
- R spawns separate Java process using `system2()`
- No shared memory between R and Java
- Communication only via command-line flags and stdout
- Java process terminates after completion

### ✓ **Command-Line Interface**
- All configuration via flags (not direct API calls)
- Flag-based approach limits flexibility
- No access to intermediate Java objects
- Cannot compose custom algorithm chains

### ✓ **File-Based Data Transfer**
- Input: Data file read from disk by Java
- Output: Text printed to stdout, captured by R
- No in-memory data transfer
- Potential bottleneck for large datasets

### ✓ **Modular Function Design**
- Each R function returns flag vector
- Composable: Combine flags from different modules
- Easy to understand: One function per concept
- Type-safe within R

### ✓ **Standard R Package Structure**
- Follows R package conventions (DESCRIPTION, NAMESPACE, man/)
- Can be installed via `install.packages()`
- Integrates with RStudio
- Uses roxygen2 for documentation

---

## 📊 File Count Summary

- **R source files**: 8 core modules (tetrad.R, algorithm.R, score.R, etc.)
- **Documentation files**: 9 .Rd files + figures/
- **Vignettes**: 4 RMarkdown tutorials
- **Configuration files**: DESCRIPTION, NAMESPACE, NEWS.md, _pkgdown.yml
- **Project files**: kumu.Rproj, .Rbuildignore, .gitignore
- **README/LICENSE**: README.md, LICENSE

---

## 🔄 Comparison: Kumu vs PyKumu

| Aspect | Kumu (R) | PyKumu (Python) |
|--------|----------|-----------------|
| **Interface** | Command-line flags | Direct Python API |
| **Execution** | External Java process | In-process via JPype |
| **Data Transfer** | File I/O | In-memory conversion |
| **Performance** | Slower (process overhead) | Faster (no I/O) |
| **Flexibility** | Limited by flags | Full API access |
| **Memory** | Separate JVM | Shared JVM |
| **Output** | Text parsing | Python objects |
| **Language** | R package | Python package |
| **Learning Curve** | Easier (functions→flags) | Steeper (Java concepts) |

Both provide access to the same underlying Tetrad algorithms, but with different architectural approaches optimized for their respective language ecosystems.