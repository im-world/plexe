- Non-supervised learning cases?
- Validation overfitting - you become what you measure
- Agent brittleness and few guardrails
- Context overflows?


## Technical Q&A

### Q: In the tree search, how are models tested in parallel without missing each other's insights?
A: They don't share insights during execution; this is a Batch-Based Optimization approach.

The HypothesiserAgent generates a single hypothesis (e.g., "Let's tune the learning rate on XGBoost").
The PlannerAgent creates a batch of multiple variations (e.g., lr=0.01, lr=0.05, lr=0.1).
These variants run in completely isolated parallel threads at the same time.
The Synchronization Point: Only after all parallel runs finish does the InsightExtractorAgent wake up. It looks at the batch results as a whole, distills what worked best, and saves a unified insight to the InsightStore. The next iteration then uses this updated knowledge.

### Q: Doesn't passing the massive BuildContext to every agent cause context window exhaustion and hallucinations?
A: No, because the LLM never actually sees the BuildContext object. In Plexe, BuildContext is a purely Python-level state container managed by the orchestrator (workflow.py). The agents you see (like DatasetSplitterAgent) are just Python Wrapper Classes.

The Python wrapper reads the massive BuildContext.
It extracts only the specific narrow variables it needs for its immediate task (e.g., just the dataset_uri and task_type).
It initializes the actual smolagents.CodeAgent (the LLM) and passes only those narrow variables into its isolated execution environment via additional_args. This "air-traffic controller" pattern keeps the massive global state safely in Python memory, preventing the LLM from being overwhelmed by irrelevant data and hallucinating.

### Q: I see that Statistical Analyser Agent has some things about simple data cleaning in its prompt, but is more complex data cleaning (json flattening, removing illegible values like -10 from percentages, removing PII, using regex to extract meaningful features from longtext columns, etc.) handled anywhere?
A: **This is a major architectural gap.** The system heavily relies on the assumption that the provided Parquet/CSV dataset is already clean and relatively flat. While the `FeatureProcessorAgent` can do basic transformations (like `StandardScaler` and `OneHotEncoder`), it operates at the feature pipeline level during model search. There is no dedicated Data Engineering/Cleaning phase prior to sampling. If a column contains messy JSON strings or illegible values like `-10` for percentages, the system expects the LLM to somehow write a custom `FunctionTransformer` to handle it on the fly, which is highly unreliable. Plexe practically expects "Analytics-Ready" data.

### Q: How does the Feature Processor Agent avoid defaulting to familiar techniques (like One-Hot Encoding) and actually explore diverse feature engineering?
A: **It currently struggles with this due to LLM bias.** Because the `FeatureProcessorAgent` is driven by standard foundational models (like GPT-4 or Claude), it suffers from "mean behavior" bias. Unless explicitly prompted with a highly creative hypothesis, it will default to the most statistically common techniques it saw in its training data (e.g., `StandardScaler` for numerics, `OneHotEncoder` for strings). The framework attempts to combat this by injecting an `InsightStore` to guide it, but it fundamentally lacks an automated, brute-force feature generation engine (like Deep Feature Synthesis) that would mathematically guarantee the exploration of complex polynomial or aggregate features.

### Q: The Feature Processor Agent's prompt contains `Do NOT reference target or group columns in your pipeline code`. Do we have any other guardrails to prevent data leakage, or we are relying on feature engineering agent not hallucinating for our output to be meaningful?
A: **It primarily relies on prompt adherence, which is highly dangerous.** If you look at `plexe/agents/feature_processor.py`, the protection against target leakage is literally just a string in the prompt: `Do NOT reference target or group columns in your pipeline code`.
While Phase 4's execution environment eventually drops target columns from the dataframe before passing it to the pipeline's `.transform()` method (which prevents silent data leakage into the model), there is no programmatic Abstract Syntax Tree (AST) parser that analyzes the generated Python code to guarantee the target column name isn't hardcoded somewhere. If the agent hallucinates and references the target column, the pipeline will simply throw a `KeyError` during `fit` or `transform`, failing the run and wasting a search iteration.

### Q: Does Plexe provide any MLOps, observability, or deployment services once the model is in production?
A: **No. Plexe is purely an artifact generator, not a serving platform.** Phase 6 (Packaging) outputs a decoupled ZIP/TAR file containing the pickled model, standard inference templates, and JSON schemas. Once that package is generated, Plexe's job is done. 
It has zero built-in support for model registry tracking (like MLflow), endpoint deployment (like Sagemaker Endpoints), data drift monitoring, or real-time observability. The user is entirely responsible for taking the `work_dir/model/` package and deploying it to their own MLOps infrastructure.