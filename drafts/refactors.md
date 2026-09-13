The idea is to explain first each component and experience and go deeper in the runtime section. I.e: Provide details of the building blocks and then go deeper in the runtime. 
- Spec
	- Format
	- Usage
	- Examples
- Implementation Details
	- Introduction
	    - Talk about:
			- General architecture (Core components, experience, runtime)
			- Stack
			- Monorepo
	- Core Components
		- Packages
			- SDK (modules, APIs)(not many details because capabilities will bring more details)
				- ContentKit
			- Crafter and Foundry (deps, LLM, APIs, etc.)
			- Commons
		- Cloud Services
			- Diagram
			- Support and Data Svc
			- Application Services
				- Cloud API
					- Intro (nestjs based, postgres, etc.)
					- Modules(APIs), Database model
					- Diagrams
					- External Dependencies (NestJS)
					- Internal Dependencies (Crafter and Foundry)
	    		- Realtime (socketio based)
	- Experiences
	  - Lingv (not many details because runtime capabilities will provide more details)
	    - Diagram
	    - External Dependencies (NextJS, AuthJS)
	    - Internal Dependencies (SDK, Cloud)
	  - Anvil (now a CLI but in the future will be an agent and its UX)
	- Runtime Capabilities (core)
		Eivolet System
			Eivolet Definitions (specification format)
			Eivolets (generated artifacts)
			Generation flow (Definition → Foundry → Eivolet)
		Content Capabilities
			Aggregates (organizational containers)
			Interactive Learning Material
			Trials (Gym, Championship, Playrooms)
			Challenges
			Flashcards    

when we finish this we will have to refactor the document and split it into:
- Introduction (describing the three main areas: Auth, Back office, Content)

Moder System Architecture
- Based on K8s
- Highly Modular
- Extensible
- AI at its core

Modules
	- Auth/AuthZ
	- Cloud API
	- Content Engine
	- Learning Assistant 
