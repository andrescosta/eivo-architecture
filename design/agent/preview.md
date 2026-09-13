Tasks: 
- Add the user id to chat backend APIs and redis keys <DONE>
- New Preview API:
    - Template <DONE>
    - Assets
    , this properties must be provided:
        - User ID
        - Session ID (the client)
    
    The AI will push:
        /template/{ID} 
        /eivolet

- Template:
    - pass the session id as parameter from the UI to the provider (?sessionId = ....) <DONE>
    - get user id, session id, and eivolet id  <DONE>
    - get parents using template name  <DONE>
    - get template using parents (EiBot?)  <DONE>
    - call content kit renderPreview from a new provider  <DONE>
    - configure facets for template rendering  <DONE>
    - create the screen for rendering the template  <DONE>

- Eivolet:
    - get user id, session id, and eivolet id  <DONE>
    - get eivolet (EiBot?)  <DONE>
    - call lingv new endpoint preview component (?)
        - facets new routes for managing previews  <DONE>
        - new screens for:
            - One Cell with The Eivolet Name
            - Grids for each activity: 
            - Learning (all units)
            - Gym(Access to One exercise of each), 
            - Challenges (just creating one and rendering the result), 
            - Spaces (just informational)

TODO:
    - Fix what the tools are returning
    
Testing strategy:
    - Template rendering:
        - Create an Eivolet with one template


New Strategy:
    - Root
        |
        -> Learning, action: ...
        