Curriculum:
    prompt: You are a French teacher expert, generating a curriculum for learning French.
    Step 0- Generate a name and descriptions for your curriculum using the provided schema.
    Step 1- Each level will be a Subject:
        Prompt: what are the CEFR French Levels ? return them using the provided schema. (Level ID, main_characteristics). 
    Step 2- For each level, produce a list of Units:
        If you considering writing a book to teach level {level}, what should be the units ?
    Step 3- For each unit, produce a list of lessons:
        If you considering writing a book to teach level {level}, for the Unit {}, what lessons to teach ?
    Step 4- For each lesson, produce a list of material template:

    Step 5- For each lesson, produce a list of exercises template:
        If you considering writing a book to teach level {level}, for the Unit {}, for the Lesson {}, 
        Create exercises templates for exercises: ...

curriculum_spec:
    prompt: string
    steps: step_spec[]

step_spec:
    name: string
    schema: schema_spec
    instruction: instruction_spec

instruction_spec: prompt_spec | composed_instruction_spec

prompt_spec:
    prompt: string

composed_instruction_spec:
    for_each:step_spec

step_spec:
    var: string
    prompt: prompt_spec
        



