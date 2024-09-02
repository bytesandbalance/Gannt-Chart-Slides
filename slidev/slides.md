---
layout: section
background: black
theme: dracula
---

# Serilized Gantt Chart

---
layout: default
---

# Introduction

- **Problem:** Need to represent features and their children in a structured way.
- **Objective:** Convert features into a JSON format.
- **Data Format:** `[start_date] [end_date] [program_id] [progress_status] [assigned_team] [parent_feature]->[feature]`
- **Example Data:**
  - `2023-01-01 2023-12-31 program1 InProgress TeamA null->ProductivitySuite`
  - `2023-01-01 2023-06-30 program1 Complete TeamB ProductivitySuite->Email`

---
layout: default
---


````md magic-move
```txt
2023-01-01 2023-12-31 program1 InProgress TeamA null->ProductivitySuite
2023-01-01 2023-06-30 program1 Complete TeamB ProductivitySuite->Email
2023-01-01 2023-04-30 program1 Complete TeamB Email->EmailSearch
2023-05-01 2023-06-30 program1 Complete TeamB Email->EmailFilters
```
```json
{
    "id": "program1",
    "root": {
        "feature": "ProductivitySuite",
        "start": "2023-01-01T00:00:00.000Z",
        "end": "2023-12-31T00:00:00.000Z",
        "subfeatures": [
            // it will come
        ],

    "progress_status": "Complete",
    "assigned_team": "TeamA"
    }
}
```
```json{all|3-6|8-13|16-21}
"subfeatures": [
    {
        "feature": "Email",
        "start": "2023-01-01T00:00:00.000Z",
        "end": "2023-06-30T00:00:00.000Z",
        "subfeatures": [
            {
                "feature": "EmailSearch",
                "start": "2023-01-01T00:00:00.000Z",
                "end": "2023-04-30T00:00:00.000Z",
                "subfeatures": [],
                "progress_status": "Complete",
                "assigned_team": "TeamB"
            },
            {
                "feature": "EmailFilters",
                "start": "2023-05-01T00:00:00.000Z",
                "end": "2023-06-301T00:00:00.000Z",
                "subfeatures": [],
                "progress_status": "Complete",
                "assigned_team": "TeamB"
            }
        ]
    }
```
````
---
layout: default
---

# Ingester

````md magic-move
```rust {all|1-3|5|8-21|6}
pub struct Ingester {
    pub features: Vec<RawFeature>,
}

impl<R: Read> TryFrom<BufReader<R>> for Ingester {
    type Error = crate::errors::ProgramIngesterError;

    fn try_from(mut reader: BufReader<R>) -> Result<Self, Self::Error> {
        let mut features = vec![];
        let mut buf = String::new();
        while reader.read_line(&mut buf)? > 0 {
            {
                // remove the trailing \n
                let line = buf.trim_end();
                let feature = RawFeature::from_str(line)?;
                features.push(feature);
            }
            buf.clear();
        }
        Ok(Ingester { features })
    }
}

```
````

---
layout: default
---

# RawFeature

````md magic-move
```rust
#[derive(Debug, PartialEq, Eq)]
pub struct RawFeature {
    pub id: String,
    pub parent_id: Option<String>,
    pub program_id: String,
    pub progress_status: String,
    pub assigned_team: String,
    pub start_date: chrono::DateTime<FixedOffset>,
    pub end_date: chrono::DateTime<FixedOffset>,
}
```
```rust
impl RawFeature {
    pub fn is_root(&self) -> bool {
        self.parent_id.is_none()
    }
}
```
```rust
impl TryFrom<String> for RawFeature {
    type Error = crate::errors::ProgramIngesterError;

    fn try_from(value: String) -> Result<Self, Self::Error> {
        RawFeature::from_str(value.as_str())
    }
}
```
```rust
impl FromStr for RawFeature {
    type Err = crate::errors::ProgramIngesterError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let parts: Vec<&str> = s.trim().split(" ").collect();
        //  it"ll come
    }
}
```
```rust
impl FromStr for RawFeature {
    type Err = crate::errors::ProgramIngesterError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        // As before
        if parts.len() != 6 {
            return Err(ProgramIngesterError::InvalidProgramInput(format!(
                "The feature '{s}' needs to have
                 6 parts, start, end, program,
                  progress_status, assigned_team,
                   feature-relation"
            )));
        }
    }
}
```
```rust
impl FromStr for RawFeature {
    type Err = crate::errors::ProgramIngesterError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        // As before
        let last_part = parts
            .last()
            .expect("we checked that there are 6 parts earlier");
        let feature_ids: Vec<&str> = last_part.split("->").collect();

        if feature_ids.len() != 2 {
            return Err(ProgramIngesterError::InvalidProgramInput(
                format!("The feature-relation '{s}' needs to have 2 parts")));
        }
    }
}
```
```rust

impl FromStr for RawFeature {
    type Err = crate::errors::ProgramIngesterError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        // As before
        Ok(RawFeature {
            id: feature_ids.last().unwrap().to_owned().into(),
            parent_id: match feature_ids.first().unwrap().to_owned() {
                "null" => None,
                id => Some(id.into()),
            },
            start_date: DateTime::parse_from_rfc3339(parts.first().unwrap().to_owned())?,
            end_date: DateTime::parse_from_rfc3339(parts.get(1).unwrap().to_owned())?,
            program_id: parts.get(2).unwrap().to_owned().into(),
            progress_status: parts.get(3).unwrap().to_owned().into(),
            assigned_team: parts.get(4).unwrap().to_owned().into(),
        })
    }
}
```
````

---
layout: default
---

# FeatureDataAndChildren

```rust{all|1-8|11}
#[derive(Debug)]
pub struct FeatureDataAndChildren<'a> {
    // This refers to existing data already ingested
    pub feature_data: Option<&'a RawFeature>,

    // These are the child feature IDs which can then be looked up in the FeatureMap (HashMap)
    pub children: Vec<FeatureID>,
}
// alias types to make usage simpler and more expressive
pub type FeatureID = String;
pub type FeatureMap<'a> = HashMap<FeatureID, FeatureDataAndChildren<'a>>;
```


---
layout: default
---

# ProgramIngesterError

```rust{all|5|8-9|11-15|17-21}
use std::io;

use thiserror::Error;

#[derive(Error, Debug)]
pub enum ProgramIngesterError {

    #[error("The program input is not valid: {0}")]
    InvalidProgramInput(String),

    #[error("The timestamp could not be parsed")]
    InvalidTimestamp {
        #[from]
        source: chrono::ParseError,
    },

    #[error("The IO operation failed: {source}")]
    IoError {
        #[from]
        source: io::Error,
    },
}
```


---
layout: default
---

# Feature and Serialization

```rust
#[derive(Debug, Serialize, Clone)]
pub struct Feature {
    #[serde(rename = "feature")]
    pub id: String,
    pub progress_status: String,
    pub assigned_team: String,
    pub start_date: chrono::DateTime<FixedOffset>,
    pub end_date: chrono::DateTime<FixedOffset>,
    #[serde(serialize_with = "odered_features")]
    pub subfeatures: Vec<Feature>,
}
```


---
layout: default
---

# Custom Serializer

```rust
fn odered_features<S>(value: &[Feature], serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    let mut value = value.to_owned();
    value.sort_by_key(|feature| feature.start_date);

    value.serialize(serializer)
}

```


---
layout: default
---

# Program

```rust

#[derive(Debug, Serialize, PartialEq, Clone)]
pub struct Program {
    pub id: String,
    pub root: Feature,
}
```


---
layout: default
---

# ProgramGraph


```rust

#[derive(Debug, Default, Serialize, Clone)]
pub struct ProgramGraph {
    #[serde(serialize_with = "odered_programs")]
    pub programs: Vec<Program>,
}

```


---
layout: default
---

# Serialization with `odered_programs`


```rust
fn odered_programs<S>(value: &[Program], serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    let mut value = value.to_owned();
    value.sort_by_key(|program| program.root.start_date);

    value.serialize(serializer)
}

```


---
layout: default
---

# Partial Equality with `PartialEq` Implementation


```rust
impl PartialEq for ProgramGraph {
    fn eq(&self, other: &Self) -> bool {
        // sort the programs by root start date before equality check
        let mut these_programs = self.programs.clone();
        these_programs.sort_by_key(|program| program.root.start_date);
        let mut those_programs = other.programs.clone();
        those_programs.sort_by_key(|program| program.root.start_date);

        these_programs == those_programs
    }
}
```


---
layout: default
---

# From `RawFeature` to `ProgramGraph`

````md magic-move
```rust{1-2|3-17}
impl From<Vec<RawFeature>> for ProgramGraph {
    fn from(value: Vec<RawFeature>) -> Self {
        let mut mappings = FeatureMap::new();
        for feature in value.iter() {
            match mappings.get_mut(&feature.id) {
                Some(mapping) => mapping.feature_data = Some(feature),
                None => {
                    mappings.insert(
                        feature.id.clone(),
                        FeatureDataAndChildren {
                            feature_data: Some(feature),
                            children: vec![],
                        },
                    );
                }
            };
        }
    }
}
```
```rust
for feature in value.iter() {
    if let Some(parent_id) = feature.parent_id.clone() {
        match mappings.get_mut(&parent_id) {
            Some(mapping) => mapping.children.push(feature.id.to_owned()),
            None => {
                mappings.insert(
                    parent_id,
                    FeatureDataAndChildren {
                        feature_data: None,
                        children: vec![feature.id.to_owned()],
                    },
                );
            }
        }
    };
}
```
```rust
for feature in value.iter() {
    let roots = mappings.values().filter(|&value| {
        if let Some(feature_data) = value.feature_data {
            feature_data.is_root()
        } else {
            false
        }
    });
}
```
```rust
for feature in value.iter() {
    let mut graph = ProgramGraph::default();
        for root in roots {
            if let Some(feature_data) = root.feature_data {
                let program = Program {
                    id: feature_data.program_id.clone(),
                    root: Feature {
                        id: feature_data.id.clone(),
                        start_date: feature_data.start_date,
                        end_date: feature_data.end_date,
                        assigned_team: feature_data.assigned_team.clone(),
                        progress_status: feature_data.progress_status.clone(),
                        subfeatures: resolve_subfeatures(root.children.clone(), &mappings),
                    },
                };
                graph.programs.push(program);
            } else {
                todo!("warn about missing feature_data");
            }
        }
    }
}
```
```rust{all|1|20}
fn resolve_subfeatures(child_ids: Vec<FeatureID>, mappings: &FeatureMap) -> Vec<Feature> {
    mappings
        .iter()
        .filter(|(feature_id, value)| {
            if let Some(feature_data) = value.feature_data {
                child_ids.contains(&feature_data.id) // todo: use feature_id, remove if/else
            } else {
                tracing::debug!(feature_id, "no feature_data found");
                false
            }
        })
        .filter_map(|(feature_id, value)| {
            tracing::debug!(feature_id, "...");
            value.feature_data.map(|feature_data| Feature {
                id: feature_data.id.clone(),
                start_date: feature_data.start_date,
                end_date: feature_data.end_date,
                assigned_team: feature_data.assigned_team.clone(),
                progress_status: feature_data.progress_status.clone(),
                subfeatures: resolve_subfeatures(value.children.clone(), mappings),
            })
        })
        .collect()
}
```
````

---
layout: default
---

# CLI

````md magic-move
```rust
use std::{
    env,
    fs::File,
    io::{self, BufReader},
};

use program_ingester::{input::Ingester, output::ProgramGraph};
use tracing_subscriber::layer::SubscriberExt;
```
```rust
fn main() -> anyhow::Result<()> {
    let logger = tracing_subscriber::fmt::layer().with_writer(io::stderr);
    let fallback = "info";
    let env_filter = tracing_subscriber::EnvFilter::try_from_default_env()
        .or_else(|_| tracing_subscriber::EnvFilter::try_new(fallback))
        .unwrap();
    let subscriber = tracing_subscriber::Registry::default()
        .with(logger) // .with(layer) Requires tracing_subscriber::layer::SubscriberExt
        .with(env_filter);
    tracing::subscriber::set_global_default(subscriber).expect("initialize tracing subscriber");
    let args: Vec<String> = env::args().collect();
    if args.len() > 2 {
        tracing::warn!("additional arguments supplied and will be ignored");
    }
    let ingester = if let Some(path) = args.get(1) {
        let reader = BufReader::new(File::open(path)?);
        Ingester::try_from(reader)?
    } else {
        let reader = BufReader::new(io::stdin());
        Ingester::try_from(reader)?
    };
    let graph = ProgramGraph::from(ingester.features);
    println!("{graph:#?}");
    Ok(())
}
```
```rust
let logger = tracing_subscriber::fmt::layer().with_writer(io::stderr);
let fallback = "info";
let env_filter = tracing_subscriber::EnvFilter::try_from_default_env()
    .or_else(|_| tracing_subscriber::EnvFilter::try_new(fallback))
    .unwrap();
let subscriber = tracing_subscriber::Registry::default()
    .with(logger) // .with(layer) Requires tracing_subscriber::layer::SubscriberExt
    .with(env_filter);
tracing::subscriber::set_global_default(subscriber).expect("initialize tracing subscriber");
```
```rust
fn main() -> anyhow::Result<()> {
    let logger = tracing_subscriber::fmt::layer().with_writer(io::stderr);
    let fallback = "info";
    let env_filter = tracing_subscriber::EnvFilter::try_from_default_env()
        .or_else(|_| tracing_subscriber::EnvFilter::try_new(fallback))
        .unwrap();
    let subscriber = tracing_subscriber::Registry::default()
        .with(logger) // .with(layer) Requires tracing_subscriber::layer::SubscriberExt
        .with(env_filter);
    tracing::subscriber::set_global_default(subscriber).expect("initialize tracing subscriber");
    let args: Vec<String> = env::args().collect();
    if args.len() > 2 {
        tracing::warn!("additional arguments supplied and will be ignored");
    }
    let ingester = if let Some(path) = args.get(1) {
        let reader = BufReader::new(File::open(path)?);
        Ingester::try_from(reader)?
    } else {
        let reader = BufReader::new(io::stdin());
        Ingester::try_from(reader)?
    };
    let graph = ProgramGraph::from(ingester.features);
    println!("{graph:#?}");
    Ok(())
}
```
```rust
let args: Vec<String> = env::args().collect();
if args.len() > 2 {
    tracing::warn!("additional arguments supplied and will be ignored");
}
```
```rust
fn main() -> anyhow::Result<()> {
    let logger = tracing_subscriber::fmt::layer().with_writer(io::stderr);
    let fallback = "info";
    let env_filter = tracing_subscriber::EnvFilter::try_from_default_env()
        .or_else(|_| tracing_subscriber::EnvFilter::try_new(fallback))
        .unwrap();
    let subscriber = tracing_subscriber::Registry::default()
        .with(logger) // .with(layer) Requires tracing_subscriber::layer::SubscriberExt
        .with(env_filter);
    tracing::subscriber::set_global_default(subscriber).expect("initialize tracing subscriber");
    let args: Vec<String> = env::args().collect();
    if args.len() > 2 {
        tracing::warn!("additional arguments supplied and will be ignored");
    }
    let ingester = if let Some(path) = args.get(1) {
        let reader = BufReader::new(File::open(path)?);
        Ingester::try_from(reader)?
    } else {
        let reader = BufReader::new(io::stdin());
        Ingester::try_from(reader)?
    };
    let graph = ProgramGraph::from(ingester.features);
    println!("{graph:#?}");
    Ok(())
}
```
```rust
let ingester = if let Some(path) = args.get(1) {
    let reader = BufReader::new(File::open(path)?);
    Ingester::try_from(reader)?
} else {
    let reader = BufReader::new(io::stdin());
    Ingester::try_from(reader)?
};
```
```rust
fn main() -> anyhow::Result<()> {
    let logger = tracing_subscriber::fmt::layer().with_writer(io::stderr);
    let fallback = "info";
    let env_filter = tracing_subscriber::EnvFilter::try_from_default_env()
        .or_else(|_| tracing_subscriber::EnvFilter::try_new(fallback))
        .unwrap();
    let subscriber = tracing_subscriber::Registry::default()
        .with(logger) // .with(layer) Requires tracing_subscriber::layer::SubscriberExt
        .with(env_filter);
    tracing::subscriber::set_global_default(subscriber).expect("initialize tracing subscriber");
    let args: Vec<String> = env::args().collect();
    if args.len() > 2 {
        tracing::warn!("additional arguments supplied and will be ignored");
    }
    let ingester = if let Some(path) = args.get(1) {
        let reader = BufReader::new(File::open(path)?);
        Ingester::try_from(reader)?
    } else {
        let reader = BufReader::new(io::stdin());
        Ingester::try_from(reader)?
    };
    let graph = ProgramGraph::from(ingester.features);
    println!("{graph:#?}");
    Ok(())
}
```
```rust{all|2|5}
// Build the graph
let graph = ProgramGraph::from(ingester.features);

// Output the graph
println!("{graph:#?}");

Ok(())
```
````
