
# Eivolet Definition 

An Eivolet Definition is a specification defined using the **Artifact Definition Language (ADL)** that orchestrates the content generation process. It acts as a configuration manifest that references models, prompts, and contexts.

## Structure 

The basic structure of an Eivolet Definition:

```yaml
model: eivoletdef  
metadata:  
 name: [identifier]  
 namespace: [namespace]  
def:  
 models:  
   object: [model configuration]  
   image: [image model configuration]  
 prompts:  
   eivolet: [main prompt reference]  
   trees: [modeler prompt references]  
   objects: [prompt references]  
   supportObjects: [helper prompt references]  
   images: [image prompt references]  
   system:  
     units: [prompt references]  
     partials: [system prompt partials]  
 contexts: [context references]

```

## Generation Algorithm 

When the **Foundry engine** is invoked to generate an **Eivolet** using an **Eivolet Definition ID**, the following process executes:

1. **Load Configuration**: Retrieve the Eivolet Definition and resolve references to models, prompts, and contexts

2. **Execute Generation**: Call Crafter for each prompt type:

   * *trees*: Generate hierarchical structures using modeler prompts

   * *objects*: Generate individual content objects

   * *images*: Generate image assets

   * *eivolet*: Generate the Eivolet specification with UI labels

3. **System Context**: All generations use system **prompts** defined in **system.units** and **system.partials** to provide consistent context and constraints

4. **Return Assets**: Content Objects

5. **Storage Decision**: The caller determines what to do with the returned content. Depending of the execution parameters, it can persist or just return them

The generation process is orchestrated by Foundry, which coordinates Crafter's interactions with LLM services based on the model configuration and prompt specifications. The **Editorial API** is the main user of Foundry.

## Specification 

### Models 

The models configuration specifies which LLM models to use for different types of content generation.

*For Object Generation:*

```yaml
models:  
  object:  
    model:  
      id: [model-identifier]  
    overrides:  
      - type: some  
        kinds: [kind-1, kind-2, ..., kind-n]  
        model:  
          id: [model-identifier]  
          provider: [openrouter | scaleway]

```

**model**: Default LLM model for generating structured content

* **id**: Model identifier (see Supported Models)

* **provider** (optional): Multi-provider service [`openrouter` | `scaleway`] - Scaleway only supports OpenAI-compatible Llama models

**overrides**: Rules to use different models based on prompt classification

* **type**: Override scope (`some` or `all`)

* **kinds**: Prompt kinds (from prompt metadata) that trigger this override (e.g., exercise, challenge)

* **model**: Alternative model configuration

*For Image Generation:*

```yaml
models:  
image:  
  model:  
    id: [model-identifier]

```

**model**: LLM model for generating images

* **id**: Image generation model identifier (currently only Gemini models supported)

### Prompts 

The **prompts** configuration references various prompt types used during the Eivolet generation or, in case of the ones listed in **supportObjects** to be used as part of some specific use case like the evaluation of a challenge.

```yaml
prompts:  
  trees: [modeler-prompt-id, ...]  
  images: [image-prompt-id, ...]  
  objects: [prompt-id, ...]  
  supportObjects: [support-prompt-id, ...]  
  eivolet: [eivolet-prompt-id]  
  system:  
    units: [prompt-id, ...]  
    partials: [partial-prompt-id, ...]
```

#### Prompt Types 

* **trees**: Modeler prompts that generate hierarchical structures (aggregates and capabilities)

* **images**: Prompts for image generation

* **objects**: Prompts that generate single objects (not hierarchical)

* **supportObjects**: Helper prompts for runtime content generation outside the Eivolet generation process. These prompts are referenced but not executed during Eivolet creation. They are used later for on-demand artifact generation, such as challenge evaluation prompts used by the Assistance engine.

* **eivolet**: Prompt that generates the Eivolet specification, including UI labels for navigation rendering

* **system**:

  * **units**: System-level prompt units

  * **partials**: Partial system prompts that compose into complete system contexts

Each prompt reference corresponds to a prompt defined elsewhere in the system, documented in the Prompts, Modeler Prompts, and Partial Prompts sections.

### Contexts 

contexts: [context-id-1, context-id-2, ...]

Contexts reference context specifications that define variables and data available during prompt generation. Context values can be referenced within prompts using Mustache template syntax:

* `{{id}}` - Escaped value

* `{{{id}}}` - Raw value

Context specifications provide shared data across all prompts in the generation process, enabling consistent variables, configuration, and content throughout the Eivolet generation.

## Example 

```yaml
model: eivoletdef  
metadata:  
  name: trainning_french  
  namespace: lingv  
def:  
  models:  
    object:  
      model:  
        id: gemini-2.5-flash-lite  
      overrides:  
        - type: some  
          kinds:  
            - exercise  
          model:  
            id: mistralai/devstral-2512:free  
            provider: openrouter  
    image:  
      model:  
        id: crafter@gemini-2.0-flash-preview-image-generation  
  prompts:  
    supportObjects:  
      - frech-challenge-feedback  
    trees:  
      - syllabusFrenchTest  
    images:  
      - image_french_course  
    system:  
      partials:  
        - system_base_partial  
        - markdown_lang_advanced  
    eivolet: eivolet_french  
  contexts:  
    - context-for-eivolet
```

# Generative Artifacts  

Generative artifacts are specifications that define how content is created through LLM-based generation. Unlike content artifacts that contain the actual learning materials, generative artifacts provide the instructions, templates, and configurations that guide the generation process. These artifacts work together to orchestrate content creation: Eivolet Definitions configure the overall generation workflow, Modeler Prompts define hierarchical content structures, Prompts specify individual content generation, Partial Prompts enable reusable prompt components, Image Prompts direct visual asset creation, Schemas enforce output structure validation, and Contexts provide shared variables across generations.

## Artifact Types Overview 

| Artifact Type | Used For | Description |
| ----- | ----- | ----- |
| Eivolet Definition | Content Generation Configuration | Specifies models, prompts, contexts, and generation workflows for creating complete content libraries |
| Modeler  | Hierarchical Content Generation | Defines recursive generation of nested content structures (syllabi → units → lessons → exercises) |
| Prompt | Single Object Generation | Provides instructions for generating individual content objects without hierarchical nesting |
| Partial Prompt | Reusable Prompt Components | Modular sections that compose into complete system prompts, enabling prompt reuse and extension |
| Image Prompt | Visual Asset Generation | Defines instructions for generating images and visual content using image generation models |
| Schema | Output Structure Validation | Zod-compatible specifications that define and validate the structure of generated content |
| Context | Shared Generation Variables | Key-value pairs that provide reusable data available across all prompts during generation |

## Modeler  

Modeler prompts are specialized prompts that generate hierarchical tree structures. They define not only what content to generate but also specify child prompts that recursively generate nested content. They are usually used to generate structures based on **aggregates**.

### Structure 

```yaml
model: modeler  
metadata:  
  name: [modeler-name]  
def:  
  prompt:  
    content: [generation instructions]  
    schema: [schema-id]  
  processors: [processor-ids]  
  children:  
    - model: modeler  
      metadata:  
        name: [child-modeler-name]  
      def:  
        prompt:  
          content: [child generation instructions]  
          schema: [child-schema-id]  
        children: [...]
```

**Elements**

* **prompt**: Generation instructions and output schema

  * **content**: Detailed instructions for the LLM, can include Mustache template variables (e.g., `{{currentUnitLabel.title}}`)

  * **schema**: Schema identifier referencing a schema specification that defines the structure of generated content

* **processors**: Optional post-generation processors (e.g., `langLabelExtractor`). They are commonly used for extracting information from the generated object and setting it in the current context.

* **children**: Array of child modeler prompts that generate nested content based on parent output

### Hierarchical Generation 

Modeler prompts enable recursive content generation where each level can reference data from parent levels through template variables. The parent's generated content provides context for child generations, enabling coherent hierarchical structures like:

*Syllabus → Units → Lessons → Exercises*

*Course → Modules → Topics → Activities*

These structures are usually constructed around aggregates and simple content objects like templates.   

### Example 

A modeler that generates a French course syllabus with nested units and lessons:

```yaml
model: modeler  
metadata:  
  name: syllabusFrenchTest  
def:  
  prompt:  
    content: |  
      Using the aggregate model from the Eivo specification, create a syllabus for an introductory French   
      course. The course should be titled 'French A1 - Beginner Level' and divided into two units:  
      …  
    schema: syllabusFrench  
  processors:  
    - langLabelExtractor  
  children:  
    - model: modeler  
      metadata:  
        name: lessonLevelInitiatesEivoFrench  
      def:  
        prompt:  
          content: |  
            For the unit titled '{{currentUnitLabel.title}}' from our 'French A1 - Beginner Level'   
            syllabus, generate a complete lesson. Include at least 3 labels, for "en-US", "fr", and "es".   
            The lesson should be   
           …         
          schema: lessonFrench
```

## Prompt 

Prompts are specifications that define single-object generation instructions. Unlike modeler prompts, they generate individual content objects without hierarchical nesting.

### Structure 

```yaml
model: prompt  
metadata:  
  name: [prompt-name]  
def:  
  content: [generation instructions]  
  schema: [schema-id]
```

**Elements**

* **content**: Detailed generation instructions for the LLM

  * Can include Mustache template variables for dynamic content

  * Should specify the expected output format and structure

  * Often includes the target schema definition inline for LLM reference

* **schema**: Schema identifier referencing a schema specification that validates the generated output

### Example 

```yaml
model: prompt  
metadata:  
  name: eivolet_french  
def:  
  content: |  
   Generate an Eivolet object. An Eivolet represents a library of curriculums organized under a common   
   theme — in this case, French language instruction. This library may include multiple course types   
   (e.g., beginner A1, business French, conversational practice), and should be named and labeled     
   accordingly.

   …  
  schema: eivolet_schema
```

## Partial Prompts 

Partial prompts are modular components that compose into complete system prompts. They enable reusable prompt sections that can be extended, customized, and combined at generation time.

### Structure 

```yaml
model: promptpartial  
metadata:  
  name: [partial-name]  
def:  
  content:  
    [section-id-1]: |  
     [section content]  
    [section-id-2]: |  
     [section content]
```

**Elements**

* **content**: Named sections that compose the partial prompt

  * Each key represents a section identifier (e.g., `intro`, `content_scope`, `eivo_spec_intro`)

  * Sections can be overridden or extended by other partial prompts

  * Multiple partials are assembled into a complete system prompt during execution

### Composition 

Partial prompts enable modular system prompt construction:

1. Base partial defines core sections

2. Extension partials add new sections or override existing ones

3. At execution time, all referenced partials are merged into a single system prompt

### Example 

```yaml
model: promptpartial  
metadata:  
name: system_base_partial  
def:  
  content:  
    intro: |  
      # System Prompt  
      ## Role  
      You are **Eivot**, an assistant capable of generating structured  
      educational content for various domains and proficiency levels.  
    content_scope: |  
      ## Content Scope  
      You are capable of generating content for any domain and level.  
      The specific level(s), domain, and topics will be defined by the user prompt.  
…

```

Extension partial adding new sections:

```yaml
model: promptpartial  
metadata:  
  name: system_base_partial_ext  
def:  
  content:  
    intro_extension: |  
     [Additional role context]  
    eivo_spec_intro_ext: |  
     [Extended specification details]
```

Partial prompts are referenced in **Eivolet Definitions** under `prompts.system.partials` and assembled during generation.

## Image Prompt 

Image prompts define instructions for generating visual assets. They specify the visual concept, style, and requirements for image generation models.

### Structure 

```yaml
model: promptimage  
metadata:  
  name: [image-prompt-name]  
def:  
  content: [image generation instructions]
```

**Elements**

* **content**: Detailed instructions describing the image to generate

  * Visual concept and subject matter

  * Style specifications (e.g., icon, illustration, photograph)

  * Technical requirements (e.g., color scheme, composition)

  * Can include Mustache template variables for dynamic descriptions

### Example 

```yaml
model: promptimage  
metadata:  
  name: image_french_course  
def:  
  content: |  
   Generate a shade of gray icon for a course that teaches French.
```

Image prompts are referenced in **Eivolet Definitions** under `prompts.images` and used with image generation models configured in the `models.image` section.

## Schema 

Schemas define the structure and validation rules for generated content using Zod-compatible specifications. They ensure generated output conforms to expected formats and includes required fields.

### Structure 

```yaml
model: schema  
metadata:  
  name: [schema-name]  
  namespace: [namespace]  
  extra:  
    [metadata-fields]  
def:  
  context:  
    [variable-name-1]: [value]  
    [variable-name-2]: [value]  
  schema:  
    [zod-schema-definition]
```

**Elements**

* **context**: Named variables available during prompt execution

  * Variable names are schema-specific (e.g., `header_prompt`, `footer_prompt`, `example_prompt`)

  * Values can be referenced in prompts that use this schema

  * Provides additional instructions, examples, and constraints for generation

* **schema**: Zod-compatible schema definition

  * Defines object structure with `type`, `properties`, and nested definitions

  * Specifies field types (`string`, `array`, `object`, `literal`)

  * Enforces structure through nested property definitions

### Example 

```yaml
model: schema  
metadata:  
  name: fill_blank_schema  
  namespace: eivo  
  extra:  
    type: one_answer  
def:  
  context:  
    header_prompt: |  
     Creative mode enabled. Generate {{quantity}} exercises.  
    footer_prompt: |  
     Make examples unique in phrasing, vocabulary, or structure. Please ensure your examples are   
     different   
     from previous variants. All variants must target the same grammatical feature and require the same   
     level   
     of response. Validate answers for consistency and correctness.  
    example_prompt: |  
     # Output Format  
     Generate the exercises as a single YAML object. This object must strictly adhere to the provided ZOD   
     schema.   
     
     **CRITICAL: Each exercise must contain ONLY ONE blank [answer].** Do not create exercises with   
     multiple blanks. Each statement should have exactly one [answer] placeholder, as shown in the    
     example below.  
    …  
  schema:  
    type: object  
    properties:  
      model:  
        type: literal  
        value: aggregate  
      metadata:  
        type: object  
        properties:  
          name:  
            type: string  
      def:  
…
```

## Context 

Contexts define reusable variables and data that can be referenced in prompts using Mustache template syntax ({{variable}} or {{{variable}}}). They provide consistent values across multiple prompts during generation.

### Structure 

```yaml
model: context  
metadata:  
  name: [context-name]  
def:  
  [variable-name-1]: [value]  
  [variable-name-2]: [value]  
…
```

**Elements**

* **def**: Key-value pairs defining variables

  * Variable names can be any valid identifier

  * Values are typically strings but can include multi-line content using `|-` syntax

  * Variables are accessible in all prompts during generation

### Usage 

Context variables are referenced in prompts using Mustache syntax:

* {{variable-name}} - Escaped value

* {{{variable-name}}} - Unescaped value (useful for injecting structured content)

### Example 

```yaml
model: context  
metadata:  
name: context-for-eivolet  
def:  
 eivolet_tags: |-  
  "eivolet"  
 tag_for_french: |-  
  "french"  
 common_localizable_tags_for_language: |-  
  "language"  
 common_curriculum_translatable_tags: |-  
  "language"  
 common_tags_for_language: |-  
  "ia"
```
These variables can be referenced in prompts using: **{{{common_tags_for_language}}}**

Contexts are referenced in **Eivolet Definitions** under the **contexts array**.

# Content Artifacts 

Content Artifacts are the structured outputs generated by executing prompts with LLM services. Each artifact conforms to a schema specification and represents a specific type of educational content or organizational structure.

Artifacts use the platform's **Artifact Specification language** (YAML) and follow the standard format with `model`, `metadata`, `labels`, and `def` fields.

## Artifact Types Overview 

| Artifact Type | Used By | Description |
| ----- | ----- | ----- |
| Eivolet | Main Material container used by the runtime to render experiences. | Metadata and labels for content library |
| Aggregate | Navigation | Organizational container for hierarchical structure |
| Template | Learning Material | Static instructional content in markdown with interactive components |
| LLM Material | Gym, Championship, Playrooms, Challenge, Flashcards | Generation instructions for dynamic content |
| Exercise | Gym, Championship, Playrooms | Trial style activities.  |
| Challenge | Challenge | Extended multi-day assignments with LLM evaluation |
| Bundle | Challenge (coding) | Project files and workspace structure |
| Feedback | Challenge | LLM-generated evaluation and guidance |
| Asset | All capabilities | Images and media files |

## Eivolet 

An **Eivolet** is the primary organizational unit for content in the platform, defining a complete content library with its generation capabilities, structure, and presentation. It serves as the root container for all related materials, activities, and assessments organized under a common domain or theme. The Eivolet acts as the entry point for discovering and accessing the hierarchical content structure stored separately in the platform.

Its **Artifact Definition Language** specification provides multilingual labels for navigation and presentation, references the Eivolet Definition used to generate the library's content, and includes categorization tags.

### Structure 

```yaml
model: eivolet  
metadata:  
  name: [eivolet-name]  
  namespace: [namespace]  
  extra:  
    eivoletDefName: [definition-reference]  
tags: [global-tags]  
labels:  
  - culture: [locale]  
    title: [display-title]  
    overview: [description]  
    tooltip: [ui-hint]  
    tags: [localized-tags]
```

**Elements**

* **metadata**: Identification and reference to generating definition

  * **name**: Display name for the Eivolet

  * **extra.eivoletDefName**: References the Eivolet Definition used for generation

* **tags**: Global tags for categorization and search

* **labels**: Localized UI labels for different cultures/languages supporting multilingual presentation

### Example 

```yaml
model: eivolet  
metadata:  
  name: French Language Learning Library  
  namespace: lingv  
  extra:  
    eivoletDefName: trainning_french  
tags: [language, french]  
labels:  
  - culture: en-US  
    title: French Language Learning  
    overview: A comprehensive library of French language courses.  
    tooltip: Explore French courses

```

## Aggregate 

Aggregates are organizational containers that structure content hierarchically. An Aggregate specification defines the metadata and labels for a container that can hold other Aggregates or Capabilities.

### Structure 

```yaml
model: aggregate  
metadata:  
  name: [aggregate-name]  
  namespace: [namespace]  
labels:  
  - culture: [locale]  
    title: [display-title]  
    overview: [description]  
    tooltip: [ui-hint]  
    tags: [localized-tags]

```

**Elements**

* **metadata**: Identification information

* **labels**: Localized UI labels for navigation and presentation

### Example 

```yaml
model: aggregate  
metadata:  
  name: frenchA1Syllabus  
  namespace: lingv  
labels:  
  - culture: en-US  
    title: French A1 - Beginner Level  
    overview: An introductory French course for absolute beginners.  
    tooltip: French A1 Syllabus  
    tags: [french, a1, beginner, syllabus]
```

Aggregates are generated by modeler prompts and form the hierarchical structure of Eivolets.

## Hierarchical Storage and Container Pattern 

**Eivolets** and **Aggregates** function as container objects in the platform's content organization. Their Artifact Definition Language specifications define only the container's own attributes—metadata, labels, tags, and references—not the objects they contain.

The contained objects (child Aggregates, Templates, LLM Materials, Exercises, Challenges) are stored separately alongside the container, similar to how subdirectories and files are organized in a file system. This hierarchical storage pattern enables the platform to navigate content structures efficiently, loading container definitions independently from their contents.

When using file system-based object storage, this organization is reflected directly in the directory structure:

```
**[namespace]**/  
└── [eivolet-name]/  
   ├── [eivolet].eivolet.yaml          *# Container definition only*  
   ├── tags/  
   └── [aggregate-name]/  
       ├── [aggregate].aggregate.yaml   *# Container definition only*  
       ├── [template].template.yaml     *# Contained object*  
       ├── [llmmaterial].llmmaterial.yaml *# Contained object*  
       └── [nested-aggregate]/  
           └── ...

```
This separation allows the platform to query container hierarchies without loading all contained content, retrieve specific content on demand, and maintain clear organizational boundaries between containers and their contents.

## LLM Material 

LLM Material specifications define content that will be generated on-demand by LLM services at runtime. Unlike static artifacts, LLM Material contains generation instructions (prompts and schemas) rather than the actual content, enabling dynamic content generation.

### Structure 

```yaml
model: llmmaterial  
metadata:  
  name: [material-name]  
  namespace: [namespace]  
  kinds: [content-classifications]  
labels:  
  - culture: [locale]  
    title: [display-title]  
    overview: [description]  
    tooltip: [ui-hint]  
    tags: [localized-tags]  
def:  
  type: [material-type]  
  schema: [schema-id]  
  prompts: [generation-prompts]
```

**Elements**

* **metadata.kinds**: Classifications for the content type (e.g., `exercise`, `fill_blank`, `challenge`)

* **labels**: Localized UI labels supporting multilingual presentation

* **def**:

  * **type**: Material type (`exercise`, `challenge`, `flashcard`)

  * **schema**: Schema identifier defining the structure of generated content

  * **prompts**: Array of prompt variations for generating content

### Usage 

LLM Material is used for:

* **Trials**: Exercises that generate fill-in-blank, multiple choice, or programming activities

* **Challenges**: Extended assignments generating essays or coding projects

* **Flashcards**: Vocabulary, grammar, or concept review cards

### Example 

```yaml
model: llmmaterial  
metadata:  
  name: frenchA1GreetingsFillBlankExercises  
  namespace: lingv  
  kinds: [exercise, fill_blank]  
labels:  
  - culture: en-US  
    title: French A1 Greetings - Fill in the Blank  
    overview: Fill-in-the-blank exercises for basic greetings and introductions.  
    tooltip: French Fill-in-the-Blank Exercises  
    tags: [french, a1, greetings, fill_blank, exercise]  
def:  
  type: exercise  
  schema: fill_blank_schema  
  prompts:  
    - Create 3 fill-in-the-blank exercises for French A1 beginners focusing on  
      greetings. Each exercise should have 5-9 blanks testing common greetings  
      and courtesy phrases.  
    - Generate fill-in-the-blank exercises covering dialogues for meeting someone  
      and asking names. Use A1-level vocabulary with natural context.
```

LLM Material is generated by modeler prompts and referenced by runtime capabilities for dynamic content generation.

## Template 

Templates contain static educational content in **MDX** format (Markdown \+ JSX). Unlike LLM Material which generates content dynamically, templates provide pre-authored lessons, tutorials, or instructional materials that are rendered as-is.

### Rendering 

Templates are rendered at runtime to HTML using MDX (Markdown with JSX). This enables:

* Standard markdown syntax for content structure

* Embedded React components for interactivity

* Dynamic content rendering

* Code syntax highlighting with annotations

The rendering process transforms the MDX content and custom components into interactive HTML, combining static instructional content with dynamic, executable components.

### Structure 

```yaml
model: template  
metadata:  
  name: [template-name]  
  namespace: [namespace]  
  kinds: [content-classifications]  
labels:  
  - culture: [locale]  
    title: [display-title]  
    overview: [description]  
    tooltip: [ui-hint]  
    tags: [localized-tags]  
def:  
  type: [content-type]  
  format: markdown  
  content:  
    - culture: [locale]  
      body: [markdown-content]
```

**Elements**

* **metadata.kinds**: Content classifications (e.g., `lesson`, `tutorial`)

* **labels**: Localized UI labels for multilingual presentation

* **def**:

  * **type**: Content type (e.g., `lesson`)

  * **format**: Content format (currently `markdown`)

  * **content**: Array of localized content with culture-specific markdown

### Markdown Features 

Templates support rich markdown including:

* Tables (for vocabulary lists, conjugations, grammar patterns)

* Code blocks and inline code

* Lists and formatting

* Links and resources

* **Interactive components**: Custom React components embedded directly in markdown for exercises and activities

#### Interactive Components 

Templates can include multiple types of interactive components embedded directly in markdown, such as fill-in-the-blank or programming exercises that learners can complete as part of the learning material.

##### Exercise Components 

**FillBlankMultiExercise**: Fill-in-the-blank exercises with tips and validation.

Component structure:

* `<Statement>`: Exercise text with `[Fill:reference]` placeholders

* `<Solutions>`: Answer definitions with tips and correct answers

* Enables immediate feedback and practice within instructional content

Example interactive exercise:

```xml
<FillBlankMultiExercise name="greetings-ex1">  
  <Statement>  
    {`Complete the sentences:  
    Person A: Bonjour \! Je [Fill:name1] Marie.`}  
  </Statement>  
  <Solutions>  
    <Solution name="name1">  
      <Tip>Verb 's'appeler' for 'Je'</Tip>  
      <Answer>m'appelle</Answer>  
    </Solution>  
  </Solutions>  
</FillBlankMultiExercise>

```
**ProgrammingExercise**: Code exercises with test cases for validation

* Includes `lang` attribute for language specification

* `<Statement>`: Exercise description

* `<TestCases>`: Test cases with input/output validation

Example interactive programming exercise:

# Programming Exercise

```xml
<ProgrammingExercise lang="rust" name="control-flow-ex1">  
  <Statement>  
    Write a program that checks input commands.  
  </Statement>  
  <TestCases>  
    <TestCase name="start_command">  
      <Input>START</Input>  
      <Output>Processing started</Output>  
    </TestCase>  
  </TestCases>  
</ProgrammingExercise>
```

##### Content Enhancement Components 

**Diagram**: Mermaid diagrams for flowcharts, sequences, and visualizations

* Specified with `type='mermaid'` attribute

* Supports flowcharts, sequence diagrams, etc.

**Code Annotation:** Code blocks support special annotation syntax using comments:

* `!ref`: Reference comments explaining code purpose

* `!mark(line-range) color`: Highlight specific lines with colors

* `!callout[/pattern/]`: Call out specific code patterns with explanations

Example using enhancement components:


```yaml
# Diagram  
<Diagram type='mermaid'>{`  
  flowchart TD  
    A["Start"] --> B{"Check condition"}  
    B -- "True" --> C["Execute"]  
  `}  
</Diagram>
```

````yaml
# Annotated Code  
```rust  
// \!ref The main function entry point  
fn main() {  
  // \!mark(1:2) gold  
  if condition { // Line 1  
      println\!("True"); // Line 2  
  }  
  // \!callout[/if/] The if keyword starts a conditional  
}  
```
````
### Example 

```yaml
model: template  
metadata:  
  name: greetingsLesson  
  kinds: [lesson]  
labels:  
  - culture: en-US  
    title: 'Lesson: Greetings'  
    overview: Basic French greetings  
def:  
  type: lesson  
  format: markdown  
  content:  
    - culture: en-US  
      body: |  
       ## Greetings  
       - **Bonjour**: Hello  
     
      <FillBlankMultiExercise name="ex1">  
        <Statement>{`Je [Fill:verb] Paul.`}</Statement>  
        <Solutions>  
         <Solution name="verb">  
          <Tip>s'appeler for Je</Tip>  
          <Answer>m'appelle</Answer>  
         </Solution>  
        </Solutions>  
      </FillBlankMultiExercise>

```
## Exercises 

Exercises are interactive practice activities that provide immediate validation. They can be either runtime-generated from LLM Material specifications or pre-defined by content creators. The structure varies by archetype.

### Common Structure 

```yaml
model: exercise  
metadata:  
  extra:  
    archetype: [exercise-type]  
labels: []  
def:  
  statement: [exercise-text]  
  [archetype-specific-fields]
```

### Fill-in-the-Blank Exercises 

Structure for text completion exercises:

```yaml
def:  
  statement: [text-with-|answer|-placeholder]  
  answer: [correct-answer]  
  tip: [hint-text]

```

**Elements**

* **statement**: Exercise text with `[answer]` placeholder marking the blank

* **answer**: Correct answer to fill the blank

* **tip**: Hint or guidance for the learner

#### Example 

```yaml
model: exercise  
metadata:  
  extra:  
    archetype: fill_blank  
def:  
  statement: Je [answer] très heureux.  
  answer: suis  
  tip: Use 'être' in first person singular.
```

### Multiple Choice Exercises 

Structure for selection based exercises:

```yaml
def:  
  statement: [question-text]  
  choices: [answer-options]  
  answer: [correct-answer]  
  tip: [hint-text]
```

**Elements**

* **statement**: Question text

* **choices**: Array of possible answer options

* **answer**: The correct answer (must match one choice exactly)

* **tip**: Hint or guidance for the learner

#### Example 

```yaml
model: exercise  
metadata:  
  extra:  
    archetype: multiple_choice  
def:  
  statement: Comment répond-on à 'Comment ça va ?' ?  
  choices:  
    - Je vais bien, merci.  
    - Ça va, et toi ?  
  answer: Ça va, et toi ?  
  tip: Pensez à la manière informelle.
```

### Programming Exercises 

Structure for code validation exercises:

```yaml
def:  
  statement: [exercise-requirements]  
  testCases:  
    - name: [test-name]  
      input: [input-values]  
      output: [expected-output]
```

**Elements**

* **statement**: Exercise description and coding requirements

* **testCases**: Array of test cases for validation

  * **name**: Test case identifier

  * **input**: Array of input values

  * **output**: Array of expected output values

#### Example 

```yaml
model: exercise  
metadata:  
  extra:  
    archetype: programming
def:  
  statement: Write a function 'get_larger' that returns the larger integer.  
  testCases:  
    - name: first_larger  
      input: ['10', '5']  
      output: ['10']
```

## Challenges

Challenges are extended multi-day assignments that provide comprehensive LLM-based evaluations. They can be either runtime-generated from LLM Material specifications or pre-defined by content creators. Unlike exercises with binary feedback, challenges produce detailed assessments and guidance. The structure varies by archetype.

### Common Structure

```yaml
model: challenge  
metadata:  
  name: [challenge-name]  
  extra:  
    archetype: [challenge-type]  
labels: []  
def:  
  duration: [time-period]  
  statement: [assignment-description]  
  evaluation:  
   [archetype-specific-evaluation]
```

### Writing Challenges

Structure for essay and composition assignments:

```yaml
def:  
  duration: [timespan]  
  statement: [writing-prompt]  
  evaluation:  
    prompt: [llm-evaluation-instructions]  
    criteria:  
      words:  
        min: [minimum-word-count]  
        max: [maximum-word-count]
```

**Elements**

* **duration**: Time allowed for completion (e.g., `7d` for 7 days)

* **statement**: Assignment description and writing prompt

* **evaluation**:

  * **prompt**: Instructions for LLM evaluation

  * **criteria.words**: Word count validation

    * **min**: Minimum word count

    * **max**: Maximum word count

#### Example 

```yaml
model: challenge  
metadata:  
  name: french_a1_essay  
  extra:  
    archetype: writing  
def:  
  duration: 7d  
  statement: >-  
   Compose a 200-word essay in French about yourself or activities.  
  evaluation:  
    prompt: >-  
     Evaluate grammar, vocabulary, and organization. Provide feedback.  
    criteria:  
      words:  
        min: 150  
        max: 250
```

### Coding Challenges 

Structure for programming project assignments with environment-based execution:

```yaml
metadata:  
  extra:  
    archetype: environment  
def:  
  duration: [timespan]  
  statement: [project-requirements]  
  evaluation:  
    prompt: [llm-evaluation-instructions]  
    criteria:  
      testCases:  
        include: [boolean]  
        numbers:  
          min: [minimum-test-cases]
```

**Elements**

* **archetype**: `environment` (indicates isolated execution environment)

* **evaluation.criteria**: Quantifiable evaluation criteria

  * Currently unused; reserved for future automated validation

  * **testCases.include**: Future: whether test cases are required

  * **testCases.numbers.min**: Future: minimum test case count for validation

All evaluation is currently performed by LLM analysis based on the evaluation prompt.

#### Example 

```yaml
model: challenge  
metadata:  
  name: reactBasicCounterChallenge  
  extra:  
    archetype: environment  
def:  
  duration: 3d  
  statement: >-  
   Create a React 'Counter' component using useState hook with h1 display  
   and increment button.  
  evaluation:  
    prompt: >-  
      Evaluate useState usage, increment logic, component structure, and  
      React best practices.  
    criteria:  
      testCases:  
        include: true  
        numbers:  
          min: 2
```

## Bundle 

Bundles are collections of files and directories that define complete archive structures. They specify file hierarchies, content, and metadata for organizing related files into a cohesive package. Bundles are commonly used to provide initial project structure and supporting materials for coding challenges, including workspace setup, starter code, configuration files, and documentation.

### Structure 

```yaml
model: bundle  
metadata:  
  name: [bundle-name]  
  namespace: [namespace]  
labels: []  
def:  
  objects:  
    - relativePath: [path]  
      kind: [file-type]  
      name: [filename]  
      content: [file-content]
```

**Elements**

* **def.objects**: Array of file objects

  * **relativePath**: Directory path relative to project root

  * **kind**: File type (`document` for text files, `code` for source files)

  * **name**: Filename with extension

  * **content**: File content (text or code)

### Usage 

Bundles provide the initial workspace for coding challenges with archetype `environment`. They include:

* **Documentation**: README files with challenge instructions

* **Configuration**: package.json, tsconfig.json, build configs

* **Starter code**: Initial source files with TODO markers

* **Tests**: Unit test files for validation

* **Assets**: HTML, CSS, and other supporting files

### Example 

```yaml
model: bundle  
metadata:  
  name: reactTodoBundle  
def:  
  objects:  
    - relativePath: ''  
      kind: document  
      name: README.md  
      content: |  
        # Challenge: React Todo List  
        Create a Todo app using useState...  
    - relativePath: src/  
      kind: code  
      name: App.tsx  
      content: |  
       import React from 'react'  
       // TODO: Implement component  
    - relativePath: src/  
      kind: code  
      name: App.test.tsx  
      content: |  
        import { describe, it } from 'vitest'  
       // Test cases
```

## Feedback 

Feedback artifacts contain evaluations for challenge submissions, providing comprehensive assessment with overall summary, strengths, and suggestions. They are typically generated by LLM-based analysis but can also represent human-created evaluations. The structure varies by challenge archetype.

### Common Structure 

```yaml
model: feedback  
metadata:  
  name: [feedback-name]  
  extra:  
    archetype: [challenge-archetype]  
labels: []  
def:  
  summary: [overall-assessment]  
  strengths: [positive-points]  
  suggestions: [improvement-areas]  
  [archetype-specific-fields]
```

### Essay Feedback 

Structure for writing challenge feedback:

```yaml
def:  
  summary: [overall-assessment]  
  strengths: [positive-points]  
  suggestions: [improvement-areas]
```

**Elements**

* **summary**: Overall assessment of the essay

* **strengths**: Positive aspects of the writing

* **suggestions**: Areas for improvement (grammar, vocabulary, structure)

#### Example 

```yaml
model: feedback  
metadata:  
  name: travel_essay_feedback  
  extra:  
    archetype: essay  
def:  
  summary: >-  
   Your essay effectively recounts travel experiences with good A1-level  
   vocabulary and coherent organization.  
  strengths: >-  
   - Relevant topic and successful use of passé composé  
   - Well-organized paragraphs with descriptive vocabulary  
  suggestions: >-  
   - Review verb conjugations and article usage  
   - Introduce more sentence structure variety
```

### Coding challenge Feedback 

Structure for coding challenge feedback:

```yaml
def:  
  summary: [overall-assessment]  
  strengths: [positive-points]  
  suggestions: [improvement-areas]  
  fileFeedbacks:  
    - file: [filepath]  
      feedback: [file-specific-feedback]
```

**Elements**

* **summary**: Overall code assessment

* **strengths**: Positive aspects of the implementation

* **suggestions**: General improvement recommendations

* **fileFeedbacks**: File-specific feedback

  * **file**: Relative file path

  * **feedback**: Specific feedback for this file

#### Example 

```yaml
model: feedback  
metadata:  
  name: reactTodoReview  
  extra:  
    archetype: environment  
def:  
  summary: >-  
   Structural foundation excellent, but useState implementation needed.  
  strengths: >-  
   - Excellent TypeScript component structure  
   - Proper form submission handling  
  suggestions: >-  
   - Implement useState for state management  
   - Use immutable state updates  
  fileFeedbacks:  
    - file: src/App.tsx  
      feedback: >-  
       Uncomment useState declarations and implement handlers.

```

## Assets 

Assets are media files, primarily images, used throughout educational content. They are stored as base64-encoded data with MIME type information.

### Structure 

```yaml
model: image  
metadata:  
  name: [asset-name]  
  namespace: [namespace]  
labels: []  
def:  
  base64: [base64-encoded-data]  
  mimeType: [IANA-media-type]
```

**Elements**

* **def.base64**: Base64-encoded file data

* **def.mimeType**: IANA media type of the file (e.g., `image/png`, `image/jpeg`, `image/svg+xml`)

### Usage 

Assets are generated from Image Prompts specified in Eivolet Definitions. They can be referenced in:

* Templates (markdown content)

* Learning material illustrations

* UI navigation elements

* Challenge instructions

### Example 

```yaml
model: image  
metadata:  
  name: french_course_icon  
  namespace: lingv  
  labels: []  
def:  
  base64: iVBORw0KGgoAAAANSUhEUgAAAAUA...  
  mimeType: image/png
```

Assets are stored in the platform and referenced by name in other artifacts.

# Configuration Objects

## CRD Template

CRD Templates define reusable Kubernetes Custom Resource Definition specifications for environment provisioning. They are organized by archetype and traits, enabling template selection based on environment requirements.

### Structure

```yaml
model: crdtemplate  
metadata:  
 name: [template-name]  
 namespace: [namespace]  
 extra:  
   archetype: [archetype-type]  
   traits:  
     [trait-key]: [trait-value]  
     ...  
def:  
 [crd-specification]
```

**Elements**

* **metadata.name**: Unique identifier for the template

* **metadata.namespace**: Namespace where the template is defined

* **extra.archetype**: Template category (e.g., "challenge")

* **extra.traits**: Key-value pairs for template selection (e.g., progLang: typescript, runtime: node, platform: react)

* **def**: Complete CRD specification including group, version, kind, and spec fields with variable placeholders

### Variable Substitution

Templates use variable placeholders in the format ``{&variable-name}`` that are replaced with values from a ConfigDatasource during rendering. The ConfigDatasource consolidates parameters from multiple sources, including Config ADL objects that define default values and infrastructure configuration.:

* `{&environment-name}`: Unique environment identifier

* `{&infra.namespace}`: Target Kubernetes namespace

* `{&capability-id}`: Challenge identifier

* `{&user-id}`: Learner identifier

* `{&host-app}`: Ingress hostname for application

* `{&project.dir}`: Project directory path

### Template Composition

CRD Templates can reference other templates for modular composition using the syntax `{:template-name}`. This enables reusable configuration blocks shared across multiple templates.

### Usage

CRD Templates are loaded from the filesystem during CRDForge initialization and stored in a registry. The CRDForge class selects templates by matching archetype and traits, then renders them with environment-specific parameters to generate complete Kubernetes CRD specifications.

### Example

```yaml
model: crdtemplate  
metadata:  
 name: nodejs-typescript  
 namespace: lingv  
 extra:  
   archetype: challenge  
   traits:  
     progLang: typescript  
     runtime: node  
     platform: react  
def:  
 group: sandbox.eivo.ca  
 version: v1  
 kind: Environment  
 plural: environments  
 metadata:  
   name: {&environment-name}  
   namespace: {&infra.namespace}  
   labels:  
     eivo: v1  
 spec:  
   project:  
     name: {&environment-name}  
     projectDirSize: "{\&project.storage-size}"  
     projectDir: {&project.dir}  
   app:  
     image: reg.jobico.local/lingv-environment-ide:latest  
     command: ["npm", "run", "start:browser"]  
     ports:  
       - port:  
           containerPort: 3003  
           protocol: TCP  
           name: http  
         hostname: {&host-app}  
     resources:  
       requests:  
         memory: "1Gi"  
         cpu: "500m"  
       limits:  
         memory: "2Gi"  
         cpu: "1500m"
```

CRD Templates enable consistent, reusable environment provisioning configurations across the platform.

## Config Datasource

Config Datasources provide environment-specific parameters and infrastructure configuration used during CRD Template rendering. They define default values, paths, resource limits, and infrastructure endpoints that are substituted into template variables.

### Structure

```yaml
model: config  
metadata:  
 name: [config-name]  
def:  
 [configuration-parameters]
```

**Elements**

* **metadata.name**: Identifier for the configuration set (e.g., "defaults")

* **def**: Nested configuration structure containing all parameter definitions

### Common Configuration Sections

* **project**: Project directory structure, storage sizes, file limits, and PVC references

* **infra**: Infrastructure endpoints including namespace, domain, Redis connection, and API URIs

* **Account names**: Service accounts for different operations (runner, cleaner)

* **Resource specifications**: Images, timezones, and other operational parameters

### Variable Access

Configuration values are accessed in CRD Templates using dot notation: `{&section.subsection.parameter}`. For example:

* `{&project.dir}` → `/workspace/project`  
* `{&infra.redis.host}` → `redis.default`  
* `{&runner-account-name}` → `environment-poweruser`

### Usage

Config Datasources are passed to CRDForge along with archetype and traits. During template rendering, CRDForge substitutes all variable placeholders with corresponding values from the ConfigDatasource, generating environment-specific CRD specifications.

### Example

```yaml
model: config  
metadata:  
 name: defaults  
def:  
 project:  
   dir: /workspace/project  
   evalDir: /mnt/eval  
   name: project  
   storage-size: 5Gi  
   startFile: README.md  
   maxFileSzie: 20mb  
   maxTotalSzie: 20gb  
   evalPvc: challenge-eval  
   filesDir: /mnt/files  
   filesPvc: eivo-seeds  
   filesPvcDir: environments-seeds  
 runner-account-name: environment-poweruser  
 cleaner-account-name: environment-cleaner  
 cleaner-image: 'k8s.gcr.io/kubectl:v1.30.1'  
 ttl-timezone: Canada/Eastern  
 infra:  
   namespace: default  
   domain: jobico.local  
   redis:  
     host: redis.default  
     port: 6379  
   cloud:  
     uri: http://eivo-cloud-api.default
```

Config Datasources centralize environment configuration, enabling consistent parameter management across all provisioned environments.

## Commands

Commands define predefined operations that can be executed within provisioned environments. They are organized by archetype and traits, matching specific environment configurations with their appropriate command sequences.

### Structure

```yaml
model: commands  
metadata:  
 name: [commands-name]  
 namespace: [namespace]  
 extra:  
   archetype: [archetype-type]  
   traits:  
     [trait-key]: [trait-value]  
     ...  
def:  
 [operation-name]:  
   commands:  
     - [command-line]  
     - [arguments]  
     …
```

**Elements**

* **metadata.name**: Unique identifier for the command set

* **metadata.namespace**: Namespace where the commands are defined

* **extra.archetype**: Environment category (e.g., "challenge")

* **extra.traits**: Key-value pairs matching environment configuration (e.g., progLang: typescript, runtime: node, platform: react)

* **def.[operation]**: Named operation containing command sequence

* **def.[operation].commands**: Array of command-line components to execute

### Variable Substitution

Commands use variable placeholders in the format `{variableName}` that are replaced with environment-specific values during execution. Common variables include:

* `{projectDir}`: Project directory path  
* `{outputDir}`: Output directory for build artifacts  
* `{testDir}`: Test directory path

### Common Operations

* **initialize**: Install dependencies and set up the development environment  
* **compile**: Build or compile source code  
* **run**: Execute the application  
* **runTest**: Run test suites  
* **clean**: Remove build artifacts or temporary files

### Usage

Challenge classes select command definitions by matching archetype and traits with the provisioned environment. When operations are needed (dependency installation, builds, test execution), the class retrieves the appropriate command sequence, substitutes variables with environment-specific paths, and executes the commands within the environment's container.

### Example

```yaml
model: commands  
metadata:  
 name: nodejs-typescript  
 namespace: lingv  
 extra:  
   archetype: challenge  
   traits:  
     progLang: typescript  
     runtime: node  
     platform: react  
def:  
 compile:  
   commands:  
     - /bin/sh  
     - -c  
     - cd {projectDir} && npm run build  
 run:  
   commands:  
     - /bin/sh  
     - -c  
     - cd {projectDir} && npm run start  
 runTest:  
   commands:  
     - /bin/sh  
     - -c  
     - cd {projectDir} && npm test  
 initialize:  
   commands:  
     - /bin/sh  
     - -c  
     - cd {projectDir} && npm i
```

Commands provide standardized operations tailored to specific environment types, enabling consistent execution workflows across different technology stacks.

