# Learning

#### Future Capabilities

Planned enhancements include:

* Learning Assistant integration for contextual support  
* Video and audio content embedding  
* Interactive animations and simulations  
* Branching content paths based on learner responses  
* Collaborative annotations and discussions

# Practice

#### Future Capabilities

Planned enhancements include:

* Adaptive difficulty based on learner performance  
* Peer review and collaborative feedback  
* Practice recommendations based on learning progress

# Evaluation

#### Future Capabilities

Planned enhancements include peer review workflows, adaptive difficulty adjustment, portfolio-based assessment, and formal certification programs.  
Regular evaluations (set of questions)  
Evaluation activities: Regular Evaluation plus Challenges

# Editorial and Agent

#### Future Capabilities

Planned enhancements include:

* Real-time learner guidance and advice during learning and practice activities  
* Contextual assistance based on current material and learner progress  
* Multi-modal content support (video, audio, interactive simulations)  
* Adaptive content difficulty  
* Collaborative authoring workflows with version control  
* Real-time assistance during content creation

# Anvil future

Future Vision

As a Content Production (Studio) experience, Anvil could evolve into a complete authoring environment. This would provide:

* Visual content editing and management  
* Collaborative authoring workflows  
* Content organization and versioning  
* Preview and testing capabilities  
* Template and asset libraries

A studio experience would make Editorial Services accessible to non-technical content creators while maintaining the efficiency of programmatic content generation for technical users.

# Builder

**Eivo Builder**: A product that allows creating Experiences using LLM and user input.

That makes total sense as a core product \- it's the natural culmination of Eivo's AI-first philosophy. If AI can generate content (Anvil), why shouldn't it generate entire experiences (Builder)?

**This fundamentally changes Eivo's positioning:**

* Not just a platform for developers to build learning apps  
* A platform where educators can *create* learning apps through conversation  
* Dramatically democratizes custom educational software

**Why exploration is needed:**

*Technical complexity:*

* Generating coherent experiences that compose multiple capabilities correctly is hard  
* UI/UX generation is notoriously difficult to get right  
* Validation: ensuring generated experiences are actually functional and pedagogically sound  
* The prompt engineering to reliably guide experience generation

*Design questions:*

* What's the output? Deployable experience? Configuration consumed by a meta-runner?  
* How much customization depth? Templates \+ variations vs truly custom?  
* Iteration model: Generate → Test → Refine loop with the user?  
* Where does Builder-generated content live in the architecture?

*Product questions:*

* Who's the user? Teachers creating one-off courses? Institutions building branded platforms?  
* What level of control do they need after generation?  
* How do you prevent generating experiences that work technically but fail pedagogically?

**Biggest opportunity:** "I want to teach calculus through interactive problem-solving with adaptive hints and peer collaboration" → working learning application. That's transformative.

This would be *the* differentiator. No one else is doing conversational experience generation for education at this level.

When does exploration start? After platform stabilization or in parallel?

**Learning API Future Enhancements**

* Learning statistics and progress tracking  
* Bookmark management for content navigation  
* Personalized learning recommendations

## **Future: Agent-Driven Eivolet Creation**

The platform's architecture is designed with a future capability in mind: **conversational AI-driven content creation** where agents collaborate with creators to generate complete Eivolets through natural dialogue, eliminating the need for manual YAML authoring.

### **Current State vs. Future Vision**

**Today**, creating an Eivolet requires technical expertise. Content creators must:

* Write Eivolet Definitions in YAML with precise syntax  
* Craft Generative Objects (prompts, schemas, contexts) following ADL specifications  
* Understand model configurations and prompt engineering  
* Know how to structure hierarchical content through modelers

This technical barrier limits content creation to those comfortable with structured data formats and LLM concepts.

**Tomorrow**, creators will describe their educational vision conversationally, and AI agents will transform these descriptions into complete, executable Eivolet Definitions. A subject matter expert with no technical knowledge could create a sophisticated learning experience through dialogue alone.

### **How Agent-Driven Creation Works**

The agent-driven workflow transforms natural conversation into technical specifications:

**1 Discovery Through Dialogue**

The creator describes their educational vision in natural language:

"I want to create a beginner Spanish course focused on conversation skills. It should cover greetings, introducing yourself, ordering food, and asking for directions. Include lots of practice with fill-in-the-blank exercises and flashcards for vocabulary. Add a final project where students write a short dialogue demonstrating what they've learned."

The agent asks clarifying questions to understand requirements:

* What proficiency level? (A1, A2, B1...)  
* How many lessons per topic?  
* Exercise difficulty progression?  
* Assessment preferences?  
* Visual design style?  
* Competitive or collaborative features?

**2 Specification Generation**

Based on the conversation, the agent analyzes requirements and generates the complete technical specification:

* **Eivolet Definition**: Selects appropriate models (faster models for exercises, more capable models for complex content), references or creates necessary prompts, defines contexts with extracted parameters (proficiency level, topic focus, exercise types)  
* **Generative Objects**: Creates modeler prompts for hierarchical structure (course → units → lessons), writes object prompts for specific content types (fill-blank exercises, flashcards, dialogue challenges), generates schemas for content validation, defines contexts with course-specific variables, creates image prompts for visual assets

The agent produces valid ADL specifications without requiring the creator to understand YAML syntax, prompt engineering, or schema design.

**3 Preview and Refinement**

The agent generates sample content using the created specification:

* Example lessons with a few paragraphs  
* Sample exercises (2-3 fill-blanks, a flashcard set)  
* Mock dialogue challenge

The creator reviews these samples and provides feedback:

"The exercises are too easy. Make them more challenging with longer sentences and less common vocabulary."

"I want more flashcards per lesson—at least 20 per topic."

"The final dialogue project should be specifically about ordering food in a restaurant, not a general conversation."

The agent adjusts the Eivolet Definition and Generative Objects based on feedback:

* Modifies prompt instructions to increase difficulty  
* Updates context variables (flashcards\_per\_lesson: 20\)  
* Refines challenge prompt to focus on restaurant scenarios

**4 Iteration Until Approval**

The process repeats—agent generates new samples, creator reviews, provides feedback, agent refines—until the creator approves the specification. Each iteration improves the Eivolet Definition without the creator touching YAML.

**5 Full Generation and Publication**

With the approved Eivolet Definition, the agent triggers the complete generation pipeline:

* Foundry executes the definition  
* Complete Eivolet with all Content Objects is generated  
* Agent reviews output for consistency  
* Creator performs final approval  
* Eivolet is published and available to learners

### **Agent Capabilities**

**Pedagogical Intelligence**: Agents learn from existing successful Eivolets to understand effective patterns:

* Exercise distribution and difficulty progression  
* Content sequencing for different learning styles  
* Assessment placement and frequency  
* Optimal lesson length and structure

**Pattern Library**: Agents maintain libraries of proven structures:

* Language learning progressions (phonetics → grammar → conversation)  
* Programming course layouts (syntax → concepts → projects)  
* Mathematics sequences (fundamentals → application → problem-solving)  
* Science curriculum patterns (theory → experiment → analysis)

**Adaptive Generation**: Agents adjust specifications based on:

* Target audience characteristics (age, background, learning goals)  
* Learning objective complexity  
* Available content types and platform capabilities  
* Creator preferences and style

**Continuous Improvement**: Agents analyze usage data from deployed Eivolets:

* Which exercise types drive highest engagement?  
* Where do learners commonly struggle?  
* What progression speeds work best?  
* Which assessment formats predict success?

Insights from this analysis improve future generation, making each created Eivolet better than the last.

### **Why This Architecture Enables Agents**

The platform's current design makes agent-driven creation feasible without fundamental architectural changes:

**Declarative Specifications**: Everything is defined in YAML data structures, not code. Agents produce structured data—a natural fit for LLMs trained on text generation.

**Composable Components**: Generative Objects are modular and reusable. Agents can mix and match existing prompts, schemas, and contexts, or generate new ones as needed.

**Separation of Concerns**: Eivolet Definitions reference Generative Objects rather than containing them. Agents can modify definitions without regenerating all supporting artifacts.

**Validation and Schemas**: Strong typing through ADL ensures agent-generated definitions are valid. Schema validation catches errors before execution.

**Unified Pipeline**: The same Foundry/Crafter pipeline processes both manually-written and agent-generated definitions. No special handling required.

The architecture doesn't just *permit* agent-driven generation—it was designed with this future in mind. Agent capabilities are an inherent potential of the current system, not a future refactoring project.

### **Impact on Content Creation**

Agent-driven generation democratizes sophisticated educational content creation:

**Accessibility**: Subject matter experts create content without technical skills. A history professor, language teacher, or professional developer can build complete learning experiences through conversation alone.

**Speed**: Hours or days of manual YAML authoring reduce to minutes of dialogue. Rapid iteration enables experimentation and refinement.

**Quality**: Agents incorporate pedagogical best practices automatically. Content creators focus on domain expertise while agents handle technical implementation.

**Scale**: Create multiple course variations (beginner/advanced, different focuses, adapted for specific audiences) from a single conversational session.

**Personalization**: Generate variations tailored to specific learner populations, institutional requirements, or teaching philosophies.

This represents a fundamental shift: educational technology that serves educators and subject matter experts directly, rather than requiring technical intermediaries to translate vision into implementation.

# Trials

**Future Possibilities:**

- **Tests**: Mixed exercise types in single session

- **Assessments**: Formal evaluation with strict timing

- **Practice Mode**: Unlimited attempts, no time pressure

- **Adaptive Learning**: Dynamic difficulty adjustment

- **Peer Review**: Collaborative exercise evaluation

## Vision and Strategic Direction

\[THIS SECTION MUST BE MOVED CANNOT BE HERE, AND REWRITTEN\]

The Facets SDK serves a broader architectural vision beyond supporting current Eivo experiences. The framework is designed to become the foundation for AI-generated learning experiences through a multi-layer architecture:

**\[REWRITE THIS IS A REALTY MORE THAN A VISION\]**

**Foundation Layer: Facets SDK**

A clean, generic framework with well-defined abstractions free from business-specific logic and styling concerns. The SDK provides composable primitives—navigation patterns, content organization, feature integration—that can be assembled in different configurations.

\[STYLING IS NEEDED IN SOME WAY\]

**Specification Layer: Domain-Specific Language (DSL)**

A higher-level specification language built on top of Facets that describes learning experiences declaratively. This DSL abstracts the technical details of the framework into concepts LLMs can understand and manipulate—learning flows, content organization patterns, interaction models.

**Generation Layer: LLM-Powered Creation**

Large language models use the DSL to write specifications for custom mini experiences. An LLM can understand a learning objective, select appropriate patterns from the DSL, and generate a complete experience specification that the framework renders.

**Application Layer: EiContent**

The primary application of this architecture: a platform where learning experiences are generated on-demand through AI. Users describe what they want to learn, and the system generates tailored mini experiences using the LLM generation capabilities.

This vision drives the refactoring priorities. Facets must be:

- **Generic**: No hardcoded business logic or domain assumptions

- **Composable**: Clear primitives that combine predictably

- **Learnable**: Patterns and abstractions an LLM can understand

- **Predictable**: Consistent behaviors that LLM-generated specs can rely on

- **Minimal**: Clean separation between framework capabilities and visual presentation

The refactoring extracts business logic and styling to achieve these goals, transforming Facets from an experience-specific implementation into a general-purpose framework suitable for AI-driven generation.

