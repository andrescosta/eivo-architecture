* Look and feel
  - New look and feel <DONE>
  - New view for the exercises with description. <DONE>
  - Change alt to tooltips. Update the UI. <DONE>
  - New UI for one answer form <DONE>
  - New exercises support
    - Several exercises in one page
    - Multiple choises
    - Long answer
  - New material support (two side cards) 
  - Internacionalization 

* Functionality:
  - Store generated exercises in its table using hash as deduplication strategy
  - Eivo API (user, course, classes,  member)
  - Integrate Eivo couse APIs and create the user experience (activities, history?, number of users took the course/exercise(UI))
  - Settings
  - Simple Games
    - Add to the exercises UI new functionality
      - Cancel
      - Skip
      - Display answer if available (go to server onclick)
      - Display tip (go to server onclick)
      - Display verb (go to server onclick)
      - Add some limits to infinite exercises.
    - Types:
      - Competitive (no tip, no verb)
      - Timed
      - Infinites
  - Group Games
    - Multi choise

* LLM:
  - Errors
    - Wrong answer
    - When using different prompts to get the same result sometimes the exercise type changes like:
      - Sometimes It asks for the verb aller + the infinitive and others just for the verb aller for the same exercise. 
  - Overall improvements
    - Tip, verb(if correspond)
    - Improve prompts to avoid repetitions 
  - Support for list of values like verbs in LLM. Needed for repetitions.
  - 
  - Generate images for the aggregates
  
* Big Themes
  - Generic solution. <DONE>
    - Transform everything to aggregates 
  - Exercises style 
    - easy, complex, etc.
    - Needs the IA.
  - How to detect user errors and use it for new exercises? Like collecting the verbs with issues.
  - Mobile with React Native?

* Others(IMPORTANT for going to production):
  - Clean up entity classes
  - Improve DefaultExecutor
  - Distributed cache (for exercises Queue). 
  - Security:
    - prompt injection attacks
    - model guardrails are violated, e.g. by generating inappropriate content.
    - JWT validations in Lingv(Next) and Eivo APIs(Nest)
  - Observavility (Jobico-cloud)
    - Telemetry 
      - For components: Next, Nest, Vercel ai
    - Logging (send server and client to Loki stack)
  - Performance
  - CI/CD (Jobico-cloud)


* Experimental:
  - Cards:
    - https://domenic.me/fsrs/
  - Freeware vs Paid users

* EiBook for Coding
  - Basic Rust
  - Generate exercises for the gym->Expected answer->Launch the IDE
  - Generate examples for the EiBook->Expected answer->Launch the IDE
