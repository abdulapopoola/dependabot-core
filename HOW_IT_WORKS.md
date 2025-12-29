# How Dependabot Core Works

This document provides a comprehensive explanation of how the Dependabot Core codebase works, its architecture, and how all the components fit together.

## Table of Contents

- [Overview](#overview)
- [High-Level Architecture](#high-level-architecture)
- [Core Concepts](#core-concepts)
- [The Update Workflow](#the-update-workflow)
- [Package Structure](#package-structure)
- [Key Components](#key-components)
- [Docker Architecture](#docker-architecture)
- [Native Helpers](#native-helpers)
- [Data Flow](#data-flow)
- [Adding a New Ecosystem (Educational)](#adding-a-new-ecosystem-educational)
- [Security Model](#security-model)

---

## Overview

**Dependabot Core** is a Ruby-based library that provides automated dependency update functionality for multiple programming languages and package managers. It's the engine that powers GitHub's Dependabot service, but it's also designed to be used as a standalone library.

### What It Does

Dependabot Core performs three main functions:

1. **Detects outdated dependencies** in a project by analyzing manifest and lockfiles
2. **Generates updated dependency files** with the latest compatible versions
3. **Creates detailed pull request descriptions** including changelogs, release notes, and commit histories

### Supported Ecosystems

Dependabot Core supports 16+ package ecosystems including:
- Ruby (Bundler)
- JavaScript (npm/yarn)
- Python (pip/pipenv/poetry)
- Java (Maven/Gradle)
- .NET (NuGet)
- Go (Go Modules)
- Rust (Cargo)
- PHP (Composer)
- Elixir (Hex/Mix)
- Swift
- Docker
- Terraform
- GitHub Actions
- Git Submodules
- And more...

---

## High-Level Architecture

Dependabot Core follows a **plugin-based architecture** where each package manager is implemented as a separate Ruby gem with a common interface.

```
┌─────────────────────────────────────────────────────────────┐
│                      Dependabot Core                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Bundler    │  │  npm_and_yarn│  │  Python      │  ...  │
│  │   Gem       │  │     Gem      │  │    Gem       │       │
│  └─────────────┘  └──────────────┘  └──────────────┘       │
│         │                 │                  │               │
│         └─────────────────┴──────────────────┘               │
│                           │                                   │
│                  ┌────────▼─────────┐                        │
│                  │  Common Library  │                        │
│                  │  (Base Classes)  │                        │
│                  └──────────────────┘                        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Principles

1. **Separation of Concerns**: Each ecosystem is isolated in its own gem
2. **Common Interface**: All ecosystems implement the same set of base classes
3. **Extensibility**: New ecosystems can be added by implementing the required interfaces
4. **Language-Specific Logic**: Complex logic is delegated to native helpers written in the target language

---

## Core Concepts

### 1. Dependency

A `Dependabot::Dependency` represents a single dependency in a project. It contains:

- **name**: The dependency name (e.g., "rails", "lodash")
- **version**: Current version (e.g., "6.0.0")
- **requirements**: Version constraints from manifest files (e.g., ">= 6.0.0")
- **package_manager**: Which ecosystem this belongs to (e.g., "bundler", "npm_and_yarn")
- **previous_version**: Version before update (set during updates)
- **previous_requirements**: Requirements before update (set during updates)

### 2. DependencyFile

A `Dependabot::DependencyFile` represents a file in the repository. It contains:

- **name**: File name with relative path (e.g., "Gemfile", "package.json", "src/requirements.txt")
- **content**: The actual file content
- **directory**: The base directory for the dependency job (not the file's directory)
- **type**: Usually "file", but can be other types
- **support_file**: Whether this is a supporting file (e.g., a vendored dependency)
- **content_encoding**: How the content is encoded (UTF-8 or Base64)
- **operation**: What operation to perform (update, create, delete)

### 3. Source

A `Dependabot::Source` represents where the code is hosted:

- **provider**: e.g., "github", "gitlab", "azure", "bitbucket"
- **repo**: Repository name (e.g., "dependabot/dependabot-core")
- **directory**: Subdirectory within the repo (optional)
- **branch**: Target branch (optional)

---

## The Update Workflow

The typical Dependabot update workflow follows these steps:

```
┌──────────────┐
│ File Fetcher │  Step 1: Fetch dependency files from repository
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ File Parser  │  Step 2: Parse files and extract dependencies
└──────┬───────┘
       │
       ▼
┌──────────────┐
│Update Checker│  Step 3: Check each dependency for updates
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ File Updater │  Step 4: Generate updated dependency files
└──────┬───────┘
       │
       ▼
┌──────────────┐
│Metadata Finder│ Step 5: Gather changelog, release notes, etc.
└──────┬───────┘
       │
       ▼
┌──────────────┐
│PR Creator    │  Step 6: Create pull request with changes
└──────────────┘
```

### Detailed Workflow Steps

#### Step 1: FileFetcher

**Purpose**: Fetch all relevant dependency files from the repository.

**What it does**:
- Connects to the repository (GitHub, GitLab, etc.)
- Identifies which files are needed (manifest files, lockfiles, supporting files)
- Downloads the files
- Validates that required files are present

**Example**: For a Ruby project, it fetches `Gemfile`, `Gemfile.lock`, and any `.gemspec` files.

**Key methods**:
- `.required_files_in?(filenames)` - Checks if required files are present
- `#files` - Returns array of `DependencyFile` objects

#### Step 2: FileParser

**Purpose**: Parse dependency files to extract a list of dependencies.

**What it does**:
- Reads manifest and lockfiles
- Extracts dependency names, versions, and requirements
- Identifies direct vs. transitive dependencies
- Handles ecosystem-specific parsing logic

**Example**: For `package.json` + `package-lock.json`, it extracts all packages with their versions and requirement strings.

**Key methods**:
- `#parse` - Returns array of `Dependency` objects

#### Step 3: UpdateChecker

**Purpose**: Determine if a dependency needs updating and what version to update to.

**What it does**:
- Queries package registries for available versions
- Checks if current version is outdated
- Determines the latest **resolvable** version (important!)
- Ensures updates won't break other dependencies
- Respects version constraints and ignore conditions

**Example**: For a dependency at version "1.0.0" with requirement "~> 1.0", it might find "1.5.0" is available but suggest "1.4.0" because "1.5.0" conflicts with another dependency.

**Key methods**:
- `#up_to_date?` - Is dependency at the latest version?
- `#can_update?` - Can dependency be updated while keeping project resolvable?
- `#latest_version` - Latest version available (ignoring resolvability)
- `#latest_resolvable_version` - Latest version that won't break other deps
- `#updated_dependencies` - Returns updated `Dependency` objects

**The Resolvability Challenge**:
This is often the most complex part. The UpdateChecker needs to ensure that updating a dependency won't break the project. This often requires:
- Running the language's native dependency resolver
- Testing different version combinations
- Understanding ecosystem-specific resolution rules

#### Step 4: FileUpdater

**Purpose**: Update dependency files with new versions.

**What it does**:
- Takes updated `Dependency` objects from UpdateChecker
- Modifies manifest files with new version requirements
- Regenerates lockfiles (by calling native package managers)
- Handles ecosystem-specific file formats

**Example**: Updates `package.json` with new version and regenerates `package-lock.json` by running npm.

**Key methods**:
- `#updated_dependency_files` - Returns array of updated `DependencyFile` objects

**Implementation approaches**:
- **String manipulation**: For simple manifest files (e.g., `requirements.txt`)
- **Native package manager**: For complex lockfiles (e.g., running `bundle update`)
- **Hybrid**: Parse manually, update, then regenerate lockfile with native tools

#### Step 5: MetadataFinder

**Purpose**: Gather information about the dependency for PR descriptions.

**What it does**:
- Finds the dependency's source code repository (often GitHub)
- Fetches changelog/release notes
- Generates commit comparisons between versions
- Finds upgrade guides if available
- Extracts relevant metadata from package registries

**Example**: For updating Rails from 6.0.0 to 6.1.0, it finds the Rails GitHub repo, links to release notes, and shows commits between versions.

**Key methods**:
- `#source_url` - Link to source code
- `#changelog_url` - Link to changelog
- `#changelog_text` - Relevant changelog text
- `#release_url` - Link to release notes
- `#commits_url` - Link to commit comparison
- `#upgrade_guide_url` - Link to upgrade guide (if exists)

#### Step 6: PullRequestCreator

**Purpose**: Create a pull request with the updates.

**What it does**:
- Formats PR title and description
- Includes changelog, release notes, and commit links
- Creates the PR via the platform's API (GitHub, GitLab, etc.)
- Handles PR labeling and assignment

**Note**: This step is often handled by the integration layer, not directly by Dependabot Core.

---

## Package Structure

Dependabot Core is organized as a **monorepo** containing multiple Ruby gems:

```
dependabot-core/
├── common/                    # Base classes and shared functionality
│   └── lib/dependabot/
│       ├── file_fetchers/
│       ├── file_parsers/
│       ├── update_checkers/
│       ├── file_updaters/
│       ├── metadata_finders/
│       ├── pull_request_creator/
│       └── ...
├── bundler/                   # Ruby/Bundler support
│   ├── lib/dependabot/bundler/
│   │   ├── file_fetcher.rb
│   │   ├── file_parser.rb
│   │   ├── update_checker.rb
│   │   ├── file_updater.rb
│   │   └── metadata_finder.rb
│   └── helpers/               # Native Ruby helpers
├── npm_and_yarn/              # JavaScript support
├── python/                    # Python support
├── go_modules/                # Go support
├── cargo/                     # Rust support
├── ... (other ecosystems)
├── omnibus/                   # Meta-gem that includes all ecosystems
├── updater/                   # GitHub's internal updater service
└── bin/
    └── dry-run.rb            # Script for local testing
```

### Common Package (`dependabot-common`)

The `common` package contains:

1. **Base Classes**: Abstract classes that each ecosystem must implement
   - `FileFetchers::Base`
   - `FileParsers::Base`
   - `UpdateCheckers::Base`
   - `FileUpdaters::Base`
   - `MetadataFinders::Base`

2. **Shared Utilities**:
   - Git operations (`GitCommitChecker`, `GitMetadataFetcher`)
   - Platform clients (GitHub, GitLab, Azure DevOps, etc.)
   - Registry client (for fetching from package registries)
   - Shared helpers for common operations

3. **Core Models**:
   - `Dependency`
   - `DependencyFile`
   - `Source`
   - `Version` (base class)
   - `Requirement` (base class)

### Ecosystem Packages

Each ecosystem package (e.g., `bundler`, `npm_and_yarn`) implements:

1. **Required Classes**:
   - `FileFetcher` - Fetches ecosystem-specific files
   - `FileParser` - Parses ecosystem-specific manifest formats
   - `UpdateChecker` - Checks for updates using ecosystem's rules
   - `FileUpdater` - Updates files in ecosystem's format
   - `MetadataFinder` - Finds metadata in ecosystem's registries

2. **Version Classes**:
   - `Version` - Handles ecosystem-specific version comparison (e.g., npm's semver vs Maven's version scheme)
   - `Requirement` - Handles ecosystem-specific requirement formats (e.g., `~> 1.0` in Ruby, `^1.0.0` in npm)

3. **Native Helpers** (optional):
   - Small executables in the target language
   - Handle complex logic like dependency resolution
   - Communicate via JSON over stdin/stdout

### Omnibus Package

The `omnibus` package is a convenience gem that depends on all ecosystem gems. If you want to support all languages, you can just require `dependabot-omnibus`.

---

## Key Components

### Base Classes

All ecosystem implementations inherit from base classes in the `common` package:

#### FileFetchers::Base

```ruby
class Dependabot::FileFetchers::MyEcosystem::FileFetcher < Dependabot::FileFetchers::Base
  def self.required_files_in?(filenames)
    # Return true if required files are present
  end

  def self.required_files_message
    # Return error message for missing files
  end

  private

  def fetch_files
    # Fetch and return array of DependencyFile objects
  end
end
```

**Responsibilities**:
- Determine which files are required
- Fetch files from the repository
- Handle subdirectories and multiple manifest files
- Detect vendored dependencies

**Available helper methods** (from `Base`):
- `fetch_file_from_host(filename, type: "file")` - Fetch a file from the repo
- `repo_contents(dir: ".")` - List contents of a directory
- `commit` - Get the current commit SHA

#### FileParsers::Base

```ruby
class Dependabot::FileParsers::MyEcosystem::FileParser < Dependabot::FileParsers::Base
  def parse
    # Parse files and return array of Dependency objects
  end

  private

  def check_required_files
    # Raise error if required files are missing
  end
end
```

**Responsibilities**:
- Parse manifest and lockfiles
- Extract dependency names, versions, and requirements
- Identify dependency types (direct, transitive, development, production)
- Handle ecosystem-specific formats

**Common patterns**:
- Parse manifest files manually (JSON, YAML, TOML, etc.)
- Use native helpers for complex formats (e.g., Gemfile with Ruby code)
- Correlate manifest and lockfile information

#### UpdateCheckers::Base

```ruby
class Dependabot::UpdateCheckers::MyEcosystem::UpdateChecker < Dependabot::UpdateCheckers::Base
  def latest_version
    # Return latest version from registry
  end

  def latest_resolvable_version
    # Return latest version that's resolvable
  end

  def latest_resolvable_version_with_no_unlock
    # Return latest version within current constraints
  end

  def updated_requirements
    # Return updated requirement strings
  end
end
```

**Responsibilities**:
- Query package registries
- Check version compatibility
- Run dependency resolution
- Respect version constraints
- Handle security vulnerabilities

**Common patterns**:
- Cache registry responses for performance
- Use native helpers for dependency resolution
- Implement retry logic for network requests
- Handle private registries with credentials

#### FileUpdaters::Base

```ruby
class Dependabot::FileUpdaters::MyEcosystem::FileUpdater < Dependabot::FileUpdaters::Base
  def self.updated_files_regex
    # Return array of regexes matching files this updater modifies
  end

  def updated_dependency_files
    # Return array of updated DependencyFile objects
  end
end
```

**Responsibilities**:
- Update manifest files with new versions
- Regenerate lockfiles
- Preserve file formatting and comments
- Handle multiple dependencies in one update

**Common patterns**:
- String manipulation for manifest files
- Shell out to native package manager to regenerate lockfiles
- Maintain original file formatting where possible
- Update multiple files atomically

#### MetadataFinders::Base

```ruby
class Dependabot::MetadataFinders::MyEcosystem::MetadataFinder < Dependabot::MetadataFinders::Base
  private

  def look_up_source
    # Return a Dependabot::Source object pointing to source code
  end
end
```

**Responsibilities**:
- Find source code repositories
- Fetch changelog and release notes
- Generate commit comparisons
- Find documentation links

**Common patterns**:
- Parse package registry metadata
- Search common hosting platforms (GitHub, GitLab)
- Handle multiple possible source locations
- Cache results to avoid repeated lookups

### Version and Requirement Classes

Each ecosystem needs to implement version comparison logic:

#### Version Classes

```ruby
class Dependabot::MyEcosystem::Version < Dependabot::Version
  def self.correct?(version)
    # Return true if this is a valid version string
  end

  def <=>(other)
    # Compare this version to another
  end
end
```

**Purpose**: Handle ecosystem-specific version formats and comparison rules.

**Examples**:
- Ruby/Node: Semantic versioning (1.2.3)
- Maven: Can have qualifiers (1.0.0-SNAPSHOT)
- Python: PEP 440 (1.0.0a1, 1.0.0rc1, 1.0.0.post1)

#### Requirement Classes

```ruby
class Dependabot::MyEcosystem::Requirement < Dependabot::Requirement
  def self.parse(requirement_string)
    # Parse a requirement string into components
  end

  def satisfied_by?(version)
    # Return true if version satisfies this requirement
  end
end
```

**Purpose**: Handle ecosystem-specific requirement formats (version constraints).

**Examples**:
- Ruby: `~> 1.0`, `>= 1.0, < 2.0`
- npm: `^1.0.0`, `~1.0.0`, `1.0.0 - 2.0.0`
- Python: `==1.0.0`, `>=1.0.0,<2.0.0`, `~=1.0.0`

### Clients

Dependabot Core includes clients for various platforms:

- **GitHub**: `Dependabot::Clients::Github` (uses Octokit)
- **GitLab**: `Dependabot::Clients::Gitlab`
- **Azure DevOps**: `Dependabot::Clients::Azure`
- **Bitbucket**: `Dependabot::Clients::Bitbucket`
- **AWS CodeCommit**: `Dependabot::Clients::CodeCommit`

These clients handle:
- Authentication
- Fetching file contents
- Creating pull requests
- Managing repository metadata

### Credentials

Credentials are passed as an array of hashes:

```ruby
credentials = [
  {
    "type" => "git_source",
    "host" => "github.com",
    "username" => "x-access-token",
    "password" => "ghp_..."  # GitHub PAT
  },
  {
    "type" => "npm_registry",
    "registry" => "registry.npmjs.org",
    "token" => "npm_..."
  }
]
```

**Types of credentials**:
- `git_source` - Access to Git repositories
- `<ecosystem>_registry` - Access to package registries
- `python_index` - Python package indexes
- `maven_repository` - Maven repositories
- etc.

---

## Docker Architecture

Dependabot uses a **multi-layered Docker architecture** to isolate ecosystems and manage dependencies.

### Image Hierarchy

```
┌─────────────────────────────────────────┐
│  Ecosystem Image (e.g., go_modules)    │  <- What you use for development
│  - Go toolchain                          │
│  - Go-specific helpers                   │
│  - Ecosystem source code                 │
├─────────────────────────────────────────┤
│  dependabot-updater-core                 │  <- Common runtime environment
│  - Ruby 3.1                              │
│  - Git, build tools                      │
│  - All ecosystem gemspecs (stub files)  │
│  - Updater service code                  │
├─────────────────────────────────────────┤
│  ubuntu:22.04                            │  <- Base OS
└─────────────────────────────────────────┘
```

### Why This Architecture?

1. **Isolation**: Each ecosystem has its own container with only necessary tools
2. **Size optimization**: Don't include Python tools in the Ruby image, etc.
3. **Version management**: Each ecosystem can use different tool versions
4. **Security**: Smaller attack surface per ecosystem

### Development Container

When you run `bin/docker-dev-shell go_modules`, you get:

1. The ecosystem image (with Go toolchain)
2. Your local source code mounted as a volume
3. A development shell with all tools available
4. The ability to run tests, dry-runs, and debuggers

**Key features**:
- **Volume mounts**: Local changes immediately reflected in container
- **Persistent shell**: Stay in the container while making changes
- **Full toolchain**: Everything needed to build and test

### Native Helpers in Docker

Native helpers are pre-compiled and installed in the Docker image:

```
/opt/
├── bundler/
│   ├── v1/        # Bundler 1.x helper
│   └── v2/        # Bundler 2.x helper
├── npm_and_yarn/
│   └── helpers/   # Node.js helpers
├── python/
│   └── helpers/   # Python helpers
└── go_modules/
    └── helpers/   # Go helpers
```

**Building helpers**:
```bash
$ bin/docker-dev-shell bundler
=> running docker development shell
[dependabot-core-dev] ~ $ bundler/helpers/v2/build
```

---

## Native Helpers

Many ecosystems require **native helpers**: small executables written in the target language that perform complex operations.

### Why Native Helpers?

Some operations are too complex or risky to implement in Ruby:

1. **Dependency resolution**: Often requires the native package manager
2. **Version parsing**: Native tools understand quirks better
3. **Lockfile generation**: Must match the exact native tool's output
4. **Security**: Avoid executing arbitrary code in the Ruby process

### How They Work

```
┌─────────────┐                    ┌──────────────┐
│   Ruby      │   JSON over stdin  │   Native     │
│   Process   │ ─────────────────> │   Helper     │
│  (Updater)  │ <───────────────── │  (Python,    │
│             │   JSON over stdout │   Node, etc) │
└─────────────┘                    └──────────────┘
```

**Communication protocol**:
1. Ruby sends JSON request to helper's stdin
2. Helper performs operation (e.g., dependency resolution)
3. Helper responds with JSON on stdout
4. Ruby parses response and continues

### Example: Bundler Helper

Ruby's `Gemfile` can contain arbitrary Ruby code, making it unsafe to eval. The Bundler helper runs in a sandboxed process:

**Request** (from Ruby):
```json
{
  "function": "parsed_gemfile",
  "args": {
    "gemfile_name": "Gemfile",
    "lockfile_name": "Gemfile.lock",
    "dir": "/tmp/project"
  }
}
```

**Response** (from helper):
```json
{
  "result": [
    {
      "name": "rails",
      "requirement": "~> 6.0",
      "groups": ["default"],
      "source": null,
      "type": "runtime"
    }
  ]
}
```

### Helper Structure

Helpers typically have:

```
bundler/helpers/v2/
├── lib/
│   ├── functions/        # Individual functions
│   │   ├── file_parser.rb
│   │   ├── dependency_resolver.rb
│   │   └── ...
│   └── functions.rb      # Function dispatcher
├── run.rb               # Entry point
├── Gemfile              # Helper's dependencies
└── build                # Build script
```

**The `run.rb` entry point**:
```ruby
# Read JSON request from stdin
request = JSON.parse($stdin.read)

# Dispatch to appropriate function
function_name = request["function"]
args = request["args"]
result = Functions.send(function_name, **args)

# Write JSON response to stdout
puts JSON.dump(result: result)
```

### Debugging Helpers

Use `DEBUG_FUNCTION` to pause execution:

```bash
DEBUG_FUNCTION=parsed_gemfile bin/dry-run.rb bundler dependabot/demo
```

This shows the exact command to run the helper manually:

```bash
cd /tmp/dependabot_TEMP/ruby && \
  echo '{"function":"parsed_gemfile","args":{...}}' | \
  bundle exec ruby /opt/bundler/v2/run.rb
```

You can then modify the helper code and re-run to test changes.

---

## Data Flow

Let's trace a complete update through the system:

### Example: Updating lodash in a Node.js project

**Initial State**:
- `package.json`: `"lodash": "^4.17.15"`
- `package-lock.json`: lodash locked to `4.17.15`
- Latest lodash version: `4.17.21`

#### Step 1: FileFetcher

```ruby
fetcher = Dependabot::FileFetchers::NpmAndYarn.new(
  source: source,
  credentials: credentials
)

files = fetcher.files
# => [
#   DependencyFile(name: "package.json", content: "..."),
#   DependencyFile(name: "package-lock.json", content: "...")
# ]
```

#### Step 2: FileParser

```ruby
parser = Dependabot::FileParsers::NpmAndYarn.new(
  dependency_files: files,
  source: source
)

dependencies = parser.parse
# => [
#   Dependency(
#     name: "lodash",
#     version: "4.17.15",
#     requirements: [{
#       requirement: "^4.17.15",
#       file: "package.json",
#       groups: ["dependencies"]
#     }],
#     package_manager: "npm_and_yarn"
#   ),
#   ... other dependencies ...
# ]
```

#### Step 3: UpdateChecker

```ruby
dependency = dependencies.find { |d| d.name == "lodash" }

checker = Dependabot::UpdateCheckers::NpmAndYarn.new(
  dependency: dependency,
  dependency_files: files,
  credentials: credentials
)

checker.up_to_date?
# => false

checker.can_update?(requirements_to_update: :own)
# => true

checker.latest_version
# => "4.17.21"

checker.latest_resolvable_version
# => "4.17.21" (compatible with other deps)

updated_deps = checker.updated_dependencies(requirements_to_update: :own)
# => [
#   Dependency(
#     name: "lodash",
#     version: "4.17.21",             # New version
#     previous_version: "4.17.15",    # Old version
#     requirements: [{
#       requirement: "^4.17.21",      # Updated requirement
#       file: "package.json",
#       groups: ["dependencies"]
#     }],
#     previous_requirements: [{       # Old requirement
#       requirement: "^4.17.15",
#       file: "package.json",
#       groups: ["dependencies"]
#     }]
#   )
# ]
```

#### Step 4: FileUpdater

```ruby
updater = Dependabot::FileUpdaters::NpmAndYarn.new(
  dependencies: updated_deps,
  dependency_files: files,
  credentials: credentials
)

updated_files = updater.updated_dependency_files
# => [
#   DependencyFile(
#     name: "package.json",
#     content: "{ \"dependencies\": { \"lodash\": \"^4.17.21\" } }"
#   ),
#   DependencyFile(
#     name: "package-lock.json",
#     content: "{ ... lodash: 4.17.21 ... }"  # Regenerated lockfile
#   )
# ]
```

**What happened internally**:
1. Parsed `package.json` and updated lodash requirement
2. Wrote updated `package.json` to temp directory
3. Ran `npm install` to regenerate `package-lock.json`
4. Read the regenerated lockfile
5. Returned both updated files

#### Step 5: MetadataFinder

```ruby
metadata_finder = Dependabot::MetadataFinders::NpmAndYarn.new(
  dependency: updated_deps.first,
  credentials: credentials
)

metadata_finder.source_url
# => "https://github.com/lodash/lodash"

metadata_finder.changelog_url
# => "https://github.com/lodash/lodash/blob/master/CHANGELOG.md"

metadata_finder.commits_url
# => "https://github.com/lodash/lodash/compare/4.17.15...4.17.21"

metadata_finder.release_url
# => "https://github.com/lodash/lodash/releases/tag/4.17.21"
```

#### Step 6: PR Creation

```ruby
pr_creator = Dependabot::PullRequestCreator.new(
  source: source,
  dependencies: updated_deps,
  files: updated_files,
  credentials: credentials,
  base_commit: commit_sha,
  pr_message_footer: "..."
)

pr = pr_creator.create
# => Creates PR with:
# - Title: "Bump lodash from 4.17.15 to 4.17.21"
# - Body: Includes changelog, release notes, commits
# - Files: package.json and package-lock.json changes
```

---

## Adding a New Ecosystem (Educational)

> **Note**: Dependabot Core is not currently accepting new ecosystem contributions. This section is for educational purposes and for understanding how the system works.

If you were to add support for a new package manager, here's the process:

### 1. Create the Gem Structure

```bash
mkdir my_ecosystem
cd my_ecosystem
```

Create `dependabot-my_ecosystem.gemspec`:

```ruby
Gem::Specification.new do |spec|
  spec.name         = "dependabot-my_ecosystem"
  spec.version      = "0.1.0"
  spec.summary      = "MyEcosystem support for Dependabot"
  
  spec.files        = Dir["lib/**/*"]
  spec.require_path = "lib"
  
  spec.add_dependency "dependabot-common", "~> 0.1"
end
```

### 2. Implement Required Classes

Create the basic file structure:

```
my_ecosystem/
└── lib/
    └── dependabot/
        └── my_ecosystem/
            ├── file_fetcher.rb
            ├── file_parser.rb
            ├── update_checker.rb
            ├── file_updater.rb
            ├── metadata_finder.rb
            ├── version.rb
            ├── requirement.rb
            └── my_ecosystem.rb
```

### 3. Implement FileFetcher

```ruby
module Dependabot
  module MyEcosystem
    class FileFetcher < Dependabot::FileFetchers::Base
      def self.required_files_in?(filenames)
        filenames.include?("myproject.manifest")
      end

      def self.required_files_message
        "Repo must contain a myproject.manifest file"
      end

      private

      def fetch_files
        fetched_files = []
        
        # Fetch manifest file
        fetched_files << fetch_file_from_host("myproject.manifest")
        
        # Fetch lockfile if present
        if repo_contents.map(&:name).include?("myproject.lock")
          fetched_files << fetch_file_from_host("myproject.lock")
        end
        
        fetched_files
      end
    end
  end
end
```

### 4. Implement FileParser

```ruby
module Dependabot
  module MyEcosystem
    class FileParser < Dependabot::FileParsers::Base
      require "json"

      def parse
        dependencies = []
        
        JSON.parse(manifest_file.content).each do |name, version|
          dependencies << Dependency.new(
            name: name,
            version: version,
            requirements: [{
              requirement: version,
              file: "myproject.manifest",
              groups: ["dependencies"]
            }],
            package_manager: "my_ecosystem"
          )
        end
        
        dependencies
      end

      private

      def manifest_file
        @manifest_file ||= get_original_file("myproject.manifest")
      end

      def check_required_files
        raise "No manifest!" unless manifest_file
      end
    end
  end
end
```

### 5. Implement Version Classes

```ruby
module Dependabot
  module MyEcosystem
    class Version < Dependabot::Version
      # If your ecosystem uses semver, you can just inherit
      # Otherwise, implement comparison logic
    end

    class Requirement < Dependabot::Requirement
      # Implement requirement parsing for your ecosystem
    end
  end
end
```

### 6. Implement UpdateChecker

This is usually the most complex part:

```ruby
module Dependabot
  module MyEcosystem
    class UpdateChecker < Dependabot::UpdateCheckers::Base
      def latest_version
        @latest_version ||= fetch_latest_version_from_registry
      end

      def latest_resolvable_version
        # This might require running native dependency resolution
        # For now, assume latest version is resolvable
        latest_version
      end

      def latest_resolvable_version_with_no_unlock
        # Check versions that satisfy current requirements
        available_versions.select { |v| 
          requirements_satisfied?(v)
        }.max
      end

      def updated_requirements
        # Update requirement strings to new version
        dependency.requirements.map do |req|
          req.merge(requirement: ">= #{latest_resolvable_version}")
        end
      end

      private

      def fetch_latest_version_from_registry
        # Query your ecosystem's registry
        # Return a Version object
      end

      def available_versions
        # Fetch all available versions from registry
      end

      def requirements_satisfied?(version)
        # Check if version satisfies current requirements
      end
    end
  end
end
```

### 7. Implement FileUpdater

```ruby
module Dependabot
  module MyEcosystem
    class FileUpdater < Dependabot::FileUpdaters::Base
      def self.updated_files_regex
        [/^myproject\.manifest$/, /^myproject\.lock$/]
      end

      def updated_dependency_files
        updated_files = []
        
        # Update manifest file
        updated_files << updated_manifest_file
        
        # Regenerate lockfile if needed
        updated_files << updated_lockfile if lockfile
        
        updated_files
      end

      private

      def updated_manifest_file
        content = manifest_file.content
        
        dependencies.each do |dep|
          old_req = dep.previous_requirements.first[:requirement]
          new_req = dep.requirements.first[:requirement]
          
          content = content.gsub(
            /"#{dep.name}": "#{old_req}"/,
            "\"#{dep.name}\": \"#{new_req}\""
          )
        end
        
        updated_file(file: manifest_file, content: content)
      end

      def updated_lockfile
        # Often requires shelling out to native package manager
        # Write manifest to temp dir, run native tool, read result
      end

      def manifest_file
        @manifest_file ||= get_original_file("myproject.manifest")
      end

      def lockfile
        @lockfile ||= get_original_file("myproject.lock")
      end
    end
  end
end
```

### 8. Implement MetadataFinder

```ruby
module Dependabot
  module MyEcosystem
    class MetadataFinder < Dependabot::MetadataFinders::Base
      private

      def look_up_source
        # Parse registry metadata to find source URL
        registry_data = fetch_registry_metadata
        
        source_url = registry_data["repository"]
        return nil unless source_url
        
        # Parse into Source object
        Source.from_url(source_url)
      end

      def fetch_registry_metadata
        # Fetch from your ecosystem's registry
        # Return hash of metadata
      end
    end
  end
end
```

### 9. Write Tests

Each component should have comprehensive tests:

```ruby
# spec/dependabot/my_ecosystem/file_fetcher_spec.rb
RSpec.describe Dependabot::MyEcosystem::FileFetcher do
  it_behaves_like "a dependency file fetcher"
  
  # Add ecosystem-specific tests
end

# Similar for other components
```

### 10. Add Docker Support

Create `my_ecosystem/Dockerfile`:

```dockerfile
FROM ghcr.io/dependabot/dependabot-updater-core

# Install your ecosystem's tools
RUN apt-get update && apt-get install -y my-ecosystem-tools

# Copy and build native helpers if needed
COPY my_ecosystem/helpers /opt/my_ecosystem/helpers
RUN bash /opt/my_ecosystem/helpers/build

# Copy source code
COPY --chown=dependabot:dependabot my_ecosystem $DEPENDABOT_HOME/my_ecosystem
COPY --chown=dependabot:dependabot common $DEPENDABOT_HOME/common
COPY --chown=dependabot:dependabot updater $DEPENDABOT_HOME/dependabot-updater
```

### 11. Integration

Add to `bin/dry-run.rb`:

```ruby
require "dependabot/my_ecosystem"
```

Add to `omnibus/dependabot-omnibus.gemspec`:

```ruby
spec.add_dependency "dependabot-my_ecosystem", Dependabot::VERSION
```

---

## Security Model

Dependabot Core has several security considerations:

### 1. Credential Isolation (for GitHub's Service)

When GitHub runs Dependabot, credentials are never exposed to Dependabot Core:

```
┌──────────────────┐
│ Dependabot Core  │  Makes unauthenticated requests
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Credential Proxy │  Injects credentials
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Package Registry │
└──────────────────┘
```

This protects against malicious manifest files that try to exfiltrate credentials.

### 2. Code Execution Isolation

Some package managers allow arbitrary code execution in manifest files:
- Python's `setup.py`
- Ruby's `Gemfile` and `.gemspec`
- npm's `postinstall` scripts

**Mitigation strategies**:
- Run in isolated Docker containers
- Use native helpers in separate processes
- Sandbox code execution where possible
- Avoid evaluating untrusted code in main process

### 3. Dependency Confusion

Dependabot checks for dependency confusion attacks:
- Ensures dependencies come from expected registries
- Validates package names and sources
- Alerts on suspicious version updates

### 4. Supply Chain Attacks

When a dependency is compromised, Dependabot:
- Can be configured to only update within version constraints
- Supports ignore conditions to block specific versions
- Provides security advisories integration for known vulnerabilities

### 5. Private Registry Security

Credentials for private registries:
- Are passed securely through the credentials array
- Should use tokens rather than passwords
- Are scoped to specific registries
- Never logged or exposed in errors

---

## Conclusion

Dependabot Core is a sophisticated system for automated dependency management. Its key strengths are:

1. **Modular architecture**: Each ecosystem is independent
2. **Common interface**: Consistent API across all package managers
3. **Native integration**: Uses native tools for accuracy
4. **Comprehensive metadata**: Rich PR descriptions with changelogs
5. **Production-tested**: Powers GitHub Dependabot for millions of repositories

### Key Takeaways

- **Five main components**: FileFetcher, FileParser, UpdateChecker, FileUpdater, MetadataFinder
- **Native helpers**: Delegate complex operations to the target language
- **Docker-based**: Isolated environments for each ecosystem
- **Resolvability matters**: Updates must not break other dependencies
- **Security-focused**: Multiple layers of isolation and protection

### Further Reading

- [README.md](README.md) - General overview and setup instructions
- [CONTRIBUTING.md](CONTRIBUTING.md) - Contribution guidelines
- Component-specific READMEs in `common/lib/dependabot/*/README.md`
- Example implementations in ecosystem directories (e.g., `go_modules/`, `npm_and_yarn/`)

### Getting Started

To start working with Dependabot Core:

1. **Clone the repository**
2. **Run the development shell**: `bin/docker-dev-shell bundler`
3. **Try a dry-run**: `bin/dry-run.rb bundler owner/repo-name`
4. **Read the ecosystem code**: Look at existing implementations
5. **Run tests**: `cd bundler && rspec spec`

Good luck exploring Dependabot Core! 🤖
