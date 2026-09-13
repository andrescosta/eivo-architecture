common
------
class: enum: "conjugation|text ..."
desription: ....
quantity: int

conjugation
-----------
groups: []
type: enum: simple|phrase

Examples:

Exercise Definition:
{
    kind: conjugation,
    type: simple,
    exercise: groupe,
    short_description: "blabla",
    long_description: "blabla",
    metadata: {
        groups: [1]
    }
}

{
    kind: conjugation,
    type: simple,
    exercise: ending
    metadata: {
        endings: ["er"]
    }
}

{
    class: "negation",
    type: phrase
}


Prompt Definition:
{
    class: conjugation,
    type: simple,
    exercise: group,
    prompt: `Generate ${quantity} ${level} phrases in French using a verbe for the group ${group}. In each of these phrases the verb is missing and the student must complete it. Use a mix of personal pronouns and normal French people names. Provide the solution and verb in the infinitive in the corresponding fields.`
}


Responses:
{ 
    request: $request
    exercise[] {
        phrase: "",
        solution: "",
        verb: ""
    }
}
