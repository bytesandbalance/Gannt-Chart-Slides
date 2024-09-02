# [Serilized Gantt Chart](https://github.com/FirstHourExpress/learnings/tree/main/gantt-chart)

A Gantt chart is a type of bar chart that illustrates a project schedule. It is used to represent the timeline of a project or multiple projects, and displays the tasks or activities involved, their duration, and the dependencies between them. In software development, it is often used to track the progress of building features for a product, with each task or activity representing a step in the development process. The horizontal axis of a Gantt chart represents time, and the vertical axis represents tasks or activities. The chart helps project managers and teams to visualize the work that needs to be done, estimate the time it will take, and track progress towards completion.

Our program aims to convert the components of a Gannt diagram into a JSON representation. This allows for the data to be more easily accessible and processed for further analysis and use.

# Intorduction

I am tackling the problem of representing a set of features and their relationships in a structured JSON format. The goal is to take data that details the start and end dates, program ID, progress status, assigned team, and parent-child feature relationships, and convert this into a JSON structure. This is especially useful for creating Gantt charts that visualize the timeline and hierarchy of features within a program.

For instance, a sample input might show a feature called 'ProductivitySuite' as the parent, with 'Email' as its child feature, and 'EmailSearch' and 'EmailFilters' as sub-features. The output JSON should reflect the hierarchical structure of these features, making it easier to understand and visualize the project’s progress and dependencies."

# Ingester

The input crate is responsible for defining the input structs and implementing the required traits for reading and parsing the input data. The crate contains two main structs, `Ingester` and `RawFeature`.
click
The `Ingester` struct has a single public field features, which is a vector of `RawFeatures`. `Ingester` implements the `TryFrom` trait, which allows converting an instance of `BufReader<R>` (where `R` implements the Read trait) into an instance of `Ingester`.
click
The implementation of the `TryFrom` trait for `Ingester` reads lines from the `BufReader<R>` using the `read_line` method, removes the trailing newline character, converts the line to a `RawFeature` using the `from_str` method, and pushes the `RawFeature` to the features vector.

**Note** The reason we use `while reader.read_line()` instead of iterating over the lines with `map` is because the latter allocates a string for each iteration, while with a dumb loop we can instead use string buffer that is allocated once, and cleared after each loop.

click
If any errors occur during the reading or conversion process, the implementation returns an `Err` variant of the Result, with the error being of type `crate::errors::ProgramIngesterError`.


# RawFeature

The `RawFeature` struct contains 7 fields, including:
- ID of the node
- parent_id (if set to `None`, then it is a root feature)
- program_id
- progress_status (Complete or In Progress)
- assigned_team generating the feature
- start_date
- end_date

click

There are implementations for the `TryFrom` and `FromStr` traits for the `RawFeature` struct.
- The `TryFrom` implementation allows creating a `RawFeature` instance from a `String`

click

- The `FromStr` implementation allows creating a `RawFeature` instance from a string slice (`&str`) and tries to turn a program log line into a feature by parsing the string slice into its component parts. If the string slice doesn't have the expected format or can't be parsed into a `RawFeature`, then it returns an error.

click

The `FromStr` implementation has a `from_str` method which takes a string `s` as input and returns a `Result` containing either an instance of the `RawFeature` type or an error.

click

The input string `s` is first trimmed and split into a vector of string slices `parts` using the `split` method. If the resulting vector `parts` doesn't have 6 elements, the code returns an error indicating an invalid input string.

click
Next, the code takes the last element of the `parts` vector and splits it into two parts using the `->` separator, resulting in a `feature_ids` vector. If the `feature_ids` vector doesn't have 2 elements, the code returns an error indicating an invalid input string.

If all the checks have passed, the code creates an instance of the `RawFeature` type and returns it as the result of the function.

# `FeatureDataAndChildren`

The `FeatureDataAndChildren` struct holds two pieces of data:

- `feature_data`: an `Option` of a reference to a `RawFeature` struct. It's an `Option` because it is only updated to `Some` when that data is found (which might be later than when the `RawFeature` is created). This means that `feature_data` could either contain a reference to a `RawFeature` instance, or it could be `None`, indicating that there is no `RawFeature` instance associated with this struct.

- `children`: a Vec (vector) of `FeatureID` values. This is a list of child feature IDs that can be used to look up related information in the `FeatureMap`.

click

`FeatureMap` is a type alias for a HashMap (hash map) data structure. A HashMap is a collection of key-value pairs, where the keys are of type `FeatureID` (which is defined as a type alias for String) and the values are of type `FeatureDataAndChildren`. This type alias makes it more expressive and easier to use, as the type `FeatureMap` is more meaningful and understandable than the underlying type HashMap.

'a is a lifetime annotation. In Rust, lifetimes are a way of expressing the relationship between references. They ensure that references are used in a safe and correct way by preventing references from pointing to data that has been dropped.


# ProgramIngesterError

This code defines an error enum for the ProgramIngester module, named `ProgramIngesterError`.

click

The `Error` trait and the `#[derive(Error, Debug)]` attribute are from the `thiserror` crate. `thiserror` allows to handle custom errors as well as wrapping errors produced by other crates (to make using `?` possible).

The enum has three variants:

click
- InvalidProgramInput
This variant is used when the input to the program is not valid, and it carries a string message describing the error.

click
- InvalidTimestamp
This variant is used when the timestamp in the input cannot be parsed, and it carries an underlying error of type `chrono::ParseError`.

click
- IoError
This variant is used when an I/O operation fails, and it carries an underlying error of type `io::Error`.


# Feature and Serialization

The `Feature` struct represents a feature in a software development project. It has the following fields:
- `id`: a string identifier for the feature
- `progress_status`: a string indicating the progress status of the feature
- `assigned_team`: a string indicating the team responsible for the feature
- `start_date`: a `chrono::DateTime<FixedOffset>` indicating the start date of the feature
- `end_date`: a `chrono::DateTime<FixedOffset>` indicating the end date of the feature
- `subfeatures`: a vector of `Feature` objects, representing subfeatures of the current feature

The struct is decorated with `#[derive(Debug, Serialize, Clone)]`, which uses Rust's "derive" macro to automatically generate implementations for the `Debug`, `Serialize`, and `Clone` traits. This means that instances of `Feature` can be debugged, serialized (converted to a format like JSON or BSON), and cloned (duplicated).

The line `#[serde(rename = "feature")]` uses Serde's "serde" attribute to specify that the `id` field should be renamed to `"feature"` when serializing or deserializing instances of `Feature`. The line `#[serde(serialize_with = "odered_features")]` uses Serde's "serde" attribute to specify that the `subfeatures` field should be serialized using a custom serialization function named `odered_features`. This allows for custom logic to be used when serializing the `subfeatures` field.

# Custom Serializer

`odered_features` takes in a slice of `Feature` objects, `value`, and a Serde serializer, `serializer`. The function sorts the input slice of `Feature` objects by the `start_date` field and then serializes the sorted slice using the provided `serializer`. The function returns a `Result` type that represents the outcome of the serialization process. If the serialization is successful, the result will contain the serialized value of type `S::Ok`. If an error occurs during serialization, the result will contain an error value of type `S::Error`. The function uses the `where` clause to specify that the type of the `serializer` must implement the `Serializer` trait. This allows the function to be used with any serializer that implements this trait, making it more flexible and reusable. The method `to_owned()` is a method that creates a new owned value (a deep copy) from a borrowed value, such as a reference. In this case, the method is used to convert the input slice of `Feature` objects, `value`, into an owned vector of `Feature` objects. This is necessary because the sorting operation needs to modify the contents of the vector, and a reference to the original slice cannot be modified. By creating an owned copy, the original data remains unchanged, and the sort operation can be performed on the copy.

# Program
This struct has two fields: `id` of type String and `root` of type `Feature`.

# ProgramGraph

This struct has one field: programs of type `Vec<Program>`.


# Serialization with `odered_programs`

The `#[serde(serialize_with = "odered_programs")]` line is a Serde attribute that specifies the custom serialization function for the programs field. The odered_programs function sorts the programs by their root.start_date value before serializing them.


# Partial Equality with `PartialEq` Implementation

The `PartialEq` implementation for `ProgramGraph` sorts the programs field by their root.start_date value before checking for equality.


# Transformation from `RawFeature` to `ProgramGraph`

The impl `From` block is an implementation of the `From` trait, which allows a type conversion from `Vec<RawFeature>` to `ProgramGraph`. The implementation makes use of the helper function `resolve_subfeatures` to resolve the subfeatures of each feature in the vector of `RawFeature` objects and construct the `ProgramGraph` object. The code is transforming a vector of `RawFeature` objects into a `ProgramGraph` object.

click
The code initializes a FeatureMap with let mut mappings = FeatureMap::new();, creating a container to hold feature data and their relationships. It then iterates through each RawFeature in the input vector. For each feature, it checks if it already exists in mappings using match mappings.get_mut(&feature.id). If the feature is present, it updates its feature_data; if not, it inserts a new entry with the feature data and an empty list of children. This process ensures that all features are properly added or updated in the map with their respective parent-child relationships.

click
This snippet handles the establishment of parent-child relationships within the `mappings` by checking if the current feature has a `parent_id`. If it does, it attempts to update the parent’s entry in the `mappings`. If the parent already exists in the map, the snippet adds the current feature's ID to the parent's `children` list. If the parent does not exist, it inserts a new entry for the parent with no `feature_data` but initializes its `children` list with the current feature's ID. This ensures that each feature is correctly associated with its parent, or that a new parent entry is created if needed.

click

This snippet filters the values in the mappings to identify root features. It iterates over each value in the mappings and checks if the feature_data is present. If feature_data is available, it uses feature_data.is_root() to determine if the feature is a root feature (i.e., a feature with no parent). If feature_data is absent, the feature is not considered a root. The result is a collection of root features, which are used to construct the final ProgramGraph.

click
the code initializes a ProgramGraph with let mut graph = ProgramGraph::default(); and then processes each root feature to build Program objects. For each root feature in the roots collection, it checks if feature_data is present. If it is, it creates a Program with the root feature's data and recursively resolves its subfeatures using resolve_subfeatures(root.children.clone(), &mappings). The constructed Program is then added to the graph.programs list. If feature_data is missing for a root, the code contains a placeholder to handle this situation with a todo! macro. This results in a complete ProgramGraph with all the programs built from root features and their respective subfeatures.

click

The code uses a helper function, `resolve_subfeatures`, to recursively resolve and build the hierarchy of subfeatures. The function takes a list of child feature IDs (child_ids) and a reference to FeatureMap (mappings). It filters the mappings to find features that match the given child IDs and have valid feature data. For each matching feature, it maps the feature data into a Feature struct, including resolving its subfeatures recursively. The result is a vector of Feature objects representing the hierarchy, which is collected and used to construct the ProgramGraph. The function also includes tracing for debugging, which logs information about the features being processed.


# CLI

The command line interface (CLI) tool ingests data, builds a graph and outputs it to the terminal.

The following are the high-level steps:


This code imports essential modules for the program: standard library components for environment variable access, file handling, and buffered I/O operations; types from the `program_ingester` crate for managing input and output, specifically `Ingester` and `ProgramGraph`; and extensions from the `tracing_subscriber` crate to enhance logging and tracing capabilities. These imports facilitate environment configuration, data processing, and detailed logging throughout the application.

click

1. Initialize tracing: A tracing-subscriber is set up to log messages to standard error (stderr). The log level is either set by the RUST_LOG environment variable, or falls back to info if the environment variable is not set.

click

2. Read CLI arguments: The command line arguments passed to the program are collected into a vector of strings. If there are more than 2 arguments, a warning is logged that the additional arguments will be ignored.

click

3. Build the ingester: The ingester reads input from either a file specified in the command line arguments, or from standard input (stdin) if no file is specified. The ingester is built using the Ingester struct, which provides a method to parse features from the input source.

click
4. Build the graph: A ProgramGraph is built from the ingester's features. The ProgramGraph is a collection of Programs, and each Program has an id and a root Feature.

click
5. Output the graph: The ProgramGraph is output to the terminal using the println! macro and the Debug format.

# Cargo commands (in terminal)
- Test

```sh
cargo test
```

- Run examples

> **Note**: I've used examples to test small bits of code for building the library. Usually I use examples for showing how to use the library in various ways.

```sh
cargo run --example simple_tree
```

- Generate docs:

> **Note**: Rust docs are awesome. Cargo can compile example code in comments to make sure they are correct.

```sh
cargo doc --no-deps --lib --document-private-items ## add --open to open in a new browser tab
```
