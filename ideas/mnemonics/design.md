# Eivo: Mnemonic Feature Capabilities (Generalization of Flashcards)

## 1. Concept: Content-Based vs. Scene-Based Learning

A core distinction in the development of Eivo's mnemonic engine is the transition from traditional **content-based** mapping to **scene-based** interaction.

* **Content-Based (Atomic):** Traditional flashcards using a static 1:1 mapping (Front:Back). The unit of knowledge is an isolated fact.
* **Scene-Based (Structural/Temporal):** A generalization where the unit of knowledge is a "journey," "arrangement," or "process." The implementation functions as a canvas with multiple hidden areas revealed through user interaction.

---

## 2. Multi-Step Recall

A technique for sequential information where a single prompt triggers a series of ordered retrievals.

* **Mental Model:** The "Path." Each step serves as the mental anchor for the next.
* **Practical Example:** Learning the intermediates of the Citric Acid Cycle or a multi-stage server troubleshooting workflow.
* **Behavior:**
* **Chain Prompting:** Completion of Step 1 unlocks the prompt for Step 2.
* **The "Spoiler Effect" Prevention:** Information is hidden by default to prevent the illusion of competence caused by seeing the full list.
* **Partial Credit:** Allows the system to track exactly where a "mental link" breaks in a long sequence.



---

## 3. Scene-Based Implementation Architecture

From a technical standpoint, a "Scene" replaces the "Card." It consists of a container (image, text, or workspace) with several "hotspots" or hidden areas.

### Functional Components

* **The Container:** Provides the environment (e.g., a diagram, a poem, or a code block).
* **Hidden Areas (Hotspots):** Defined coordinates or DOM elements that mask the underlying data.
* **Reveal Logic (State Machine):** * **Linear Reveal:** Areas must be uncovered in a specific sequence (1 → 2 → 3).
* **Free-Flow Reveal:** User selects hotspots in any order.
* **Cumulative Visibility:** Previous steps remain visible to provide context for the current retrieval.



---

## 4. Suggested Feature Support for Eivo

### Dynamic Cognitive Associations

* **Image Occlusion:** Hiding parts of a visual scene (e.g., anatomy, circuit boards) to test spatial memory.
* **Cloze Deletion/Overlapping:** Hiding keywords within a textual scene while maintaining the surrounding narrative context.
* **Peg System Support:** Linking items to a numerical "peg" list within a scene.
* **Method of Loci (Memory Palace):** High-level scenes where data is anchored to virtual "rooms" or "paths."

### Technical Engine Requirements

* **Spaced Repetition System (SRS) Integration:** Optimization using algorithms like FSRS or SM-2 based on interaction quality.
* **Latency Tracking:** Measuring the time taken to reveal specific hotspots to identify "friction points" in a sequence.
* **Technical Rendering:** Support for LaTeX (mathematics) and syntax-highlighted code blocks within the scene components.

---

## 5. Summary of Benefits

* **Reduced Cognitive Load:** Breaks complex "walls of text" into manageable, interactive steps.
* **Contextual Encoding:** Leverages the "where" and "when" of the interaction to create stronger neural paths.
* **Exploratory Learning:** Shifts the user experience from "testing" to "discovery," reducing study friction.


---

# Eivo: Mnemonic Feature Capabilities (Generalization of Flashcards)

## 1. Concept: Content-Based vs. Scene-Based Learning

A core distinction in the development of Eivo's mnemonic engine is the transition from traditional **content-based** mapping to **scene-based** interaction.

* **Content-Based (Atomic):** Traditional flashcards using a static 1:1 mapping (Front:Back). The unit of knowledge is an isolated fact.
* **Scene-Based (Structural/Temporal):** A generalization where the unit of knowledge is a "journey," "arrangement," or "process." The implementation functions as a canvas with multiple hidden areas revealed through user interaction.

---

## 2. Multi-Step Recall

A technique for sequential information where a single prompt triggers a series of ordered retrievals.

* **Mental Model:** The "Path." Each step serves as the mental anchor for the next.
* **Behavior:**
* **Chain Prompting:** Completion of Step 1 unlocks the prompt for Step 2.
* **Spoiler Prevention:** Information is hidden by default to prevent the "illusion of competence."
* **Partial Credit:** Allows the system to track exactly where a "mental link" breaks in a long sequence.



---

## 3. Scene-Based Implementation Architecture

From a functional standpoint, a "Scene" replaces the "Card." It consists of a container (image, text, or workspace) with several "hotspots" or hidden areas.

* **The Container:** Provides the environment (e.g., a diagram, a poem, or a code block).
* **Hidden Areas (Hotspots):** Areas that mask the underlying data until triggered.
* **Reveal Logic (State Machine):**
* **Linear Reveal:** Areas must be uncovered in a specific sequence (1 → 2 → 3).
* **Free-Flow Reveal:** User selects hotspots in any order (ideal for diagrams).
* **Cumulative Visibility:** Previous steps remain visible to provide context for the current retrieval.



---

## 4. Spaced Repetition System (SRS) Integration

Eivo uses SRS algorithms (like **FSRS** or **SM-2**) to transform the application from a static library into a dynamic "memory engine."

#### The "Scheduler" Logic

The SRS algorithm acts as a **Temporal Scheduler**. It analyzes the history of every interaction across all previous sessions to calculate the optimal "Review Interval."

* **Stability (S):** How long the memory is expected to last.
* **Retrievability (R):** The probability of recalling the scene at this exact moment.
* **The Session Filter:** The system filters the fixed set of cards to show only those where the "Next Review Date" is today or in the past, based on the accumulation of data from all previous sessions.

---

## 5. Premium Capability: LLM-Driven Infinite Simulator

The Premium layer shifts Eivo from **Repetition** to **Simulation** by using an LLM to treat information as a fluid source rather than a fixed set of static cards.

#### Key Premium Functions:

* **Infinite Variations:** The LLM re-writes or re-stages the scene for every review to prevent "pattern hacking."
* **Adaptive Scaffolding:** If a user fails a recall, the LLM generates a "Guided Scene" in the next session, pre-filling known steps and focusing on the specific "break point."
* **Indeterminate Card Generation:** Users upload "Source Material" (PDFs, notes), and the system generates dynamic scenes on-the-fly.
* **Socratic Hinting:** Instead of a binary right/wrong, the LLM provides contextual, open-ended questions or subtle cues. This forces the user to "earn" the answer through active reasoning rather than passive consumption.

---

## 6. Reference Materials

For a deeper dive into the mathematical and practical differences between these systems:

* **Algorithm Comparison:** [Anki's FSRS vs SM-2](https://www.youtube.com/watch?v=OqRLqVRyIzc) — Side-by-side comparison of how these algorithms affect daily study workload and long-term memory efficiency.
* **AI & Future of Learning:** [AI-powered flashcards and study aids](https://www.youtube.com/watch?v=Eo1HbXEiJxo) — This video discusses how AI is being integrated into study platforms to move beyond traditional rote memorization toward more interactive and adaptive learning models.
* **LLM Practical Use:** [How I use LLMs](https://www.youtube.com/watch?v=EWvNQjAaOHw) — A practical walkthrough of using LLMs for deep research, file analysis, and creating personalized learning workflows.

This video demonstrates how experts leverage LLMs to process complex documents and build custom interactive learning tools.

That is a smart move. Building the **Standard Flashcard Infrastructure** first provides the structural "scaffold" for the more advanced scene-based features later.

From a functional and architectural perspective, here is how you can set up that foundational MVP (Minimum Viable Product).

### 1. The Core Infrastructure Components

To build this effectively, you need three "pillars" in your architecture:

* **The Ingestion Engine (LLM):** This handles converting raw text/PDFs into atomic  pairs.
* **The Persistence Layer:** A database that stores not just the content, but the **DSR** (Difficulty, Stability, Retrievability) metadata for each card.
* **The Scheduler (SRS Engine):** The logic that determines the "Next Review Date" based on session results.

---

### 2. Functional Workflow for the MVP

| Phase | Functional Action | Behind the Scenes |
| --- | --- | --- |
| **Generation** | User uploads a text snippet. | LLM identifies "Atomic Facts" and formats them as JSON. |
| **Storage** | User confirms/edits generated cards. | Cards are saved with `stability = 0` and `difficulty = default`. |
| **Session Start** | User clicks "Study." | System queries: `NextDate <= Today`. |
| **Interaction** | User rates recall (Hard/Good/Easy). | SRS algorithm (FSRS) calculates the new interval. |
| **Update** | Session ends. | `NextDate` and `Stability` are updated in the database. |

---

### 3. "Infrastructure Ready" Database Schema

Even for a basic flashcard, you should design your data structure to support future "Scenes." Instead of a simple "Front/Back" table, use a **Component-based** model:

* **Source Table:** Stores the original PDF/Text (the "Seed").
* **Card Table:** * `source_id`: Links back to the origin.
* `content`: A JSON object (e.g., `{ "type": "flashcard", "q": "...", "a": "..." }`).
* `srs_state`: `{ "s": 5.2, "d": 3.1, "r": 0.9 }`.


* **Review Logs:** Every single click (Hard/Easy) is logged to allow for later "re-training" of the FSRS parameters.

### 4. Implementation Tip: Use `ts-fsrs` or `py-fsrs`

Don't write the math for the FSRS algorithm from scratch. There are open-source libraries (like `ts-fsrs` for TypeScript or `py-fsrs` for Python) that act as a "black box":

1. You give it the **Current Card State** + **User Rating**.
2. It returns the **New Card State** + **Next Review Date**.

### Why this is a "Premium" Bridge

By starting with LLM generation, you are already training your users to expect **automated creation**. When you eventually launch "Scene-based" features, the transition will be seamless:

* **Free:** LLM generates a basic question.
* **Premium:** LLM generates a multi-step interactive scene from the same "Seed."

Would you like me to draft a **System Prompt** for the LLM that ensures it creates "Atomic" flashcards (the most important rule for SRS success)?

Building a **stateless** MVP is a great architectural choice. It allows you to focus on the core "generation-to-interaction" loop without the complexity of database migrations, user accounts, or persistent state management.

In this model, Eivo acts as a **Real-Time Knowledge Processor**.

### The Stateless Functional Flow

Since you aren't storing anything, the "state" only exists in the **current application memory (RAM)** and the **LLM's context window** during that specific session.

1. **Input:** The user provides the "Seed" (a PDF, a URL, or a pasted paragraph).
2. **Streaming Generation:** The LLM converts the seed into a set of flashcards.
3. **The Ephemeral Deck:** These cards are stored in a simple array in your frontend state (e.g., React `useState` or a simple JS variable).
4. **The Session:** The user reviews the cards. Since there is no SRS storage, the "Next Review" logic is purely for the current session (e.g., "Show failed cards again at the end of this list").
5. **The "Wipe":** As soon as the user closes the tab or refreshes, the data is gone.

---

### Functional Advantages for an MVP

* **Privacy by Default:** Because nothing is stored, you have zero data liability. It's an "Incognito Mode" for learning.
* **Zero Infrastructure Overhead:** You don't need a backend, a database, or authentication. The app can be a single-page frontend that calls an LLM API.
* **Rapid Iteration:** You can change the card format (from simple Q&A to Multi-step) just by changing the LLM prompt, without worrying about breaking existing database records.

### Basic SRS "Simulation" (Without Storage)

Even without a database, you can still provide a **pseudo-SRS experience** during a single session:

* **Session Weighting:** If a user marks a card as "Hard," move it to the back of the queue so they see it again before the session ends.
* **Confidence Filtering:** Let the user "discard" cards they already know, narrowing the session down to only the difficult ones.

### Preparing for the "Stateful" Evolution

To make the transition to storing data easier later, you should still structure your data as if it *were* going into a database. Use a clean JSON format for the cards:

```json
{
  "id": "temp_uuid", 
  "front": "What is the capital of France?",
  "back": "Paris",
  "metadata": { "source": "Geography 101" }
}

```

When you are ready to evolve, you simply add a `POST` request to save this JSON to a database and start tracking the `last_reviewed` timestamp.

---

### Next Step for Eivo

Would you like me to generate a **"Stateless Prompt"** that you can use with the LLM to ensure it produces clean, atomic cards that are easy for your frontend to parse?

[AI Flashcards in NotebookLM: Generate Custom Study Decks Instantly](https://www.youtube.com/watch?v=4c44X0GRHTQ)

This video demonstrates a "low-friction" workflow where documents are turned into interactive flashcard sessions, mirroring the basic, generation-focused approach you are planning for Eivo's MVP.