Memory for Eivolet Definition:
https://www.letta.com/blog/agent-memory
https://ai-sdk.dev/cookbook/guides/custom-memory-tool
https://ai-sdk.dev/docs/ai-sdk-core/mcp-tools
https://ai-sdk.dev/cookbook/node/mcp-tools
https://modelcontextprotocol.io/docs/learn/architecture
https://ai-sdk.dev/cookbook/next/stream-object
https://ai-sdk.dev/docs/reference/ai-sdk-core/tool-loop-agent
https://ai-sdk.dev/cookbook/guides/custom-memory-tool
https://gemini.google.com/app/139735d1ba7886a3
https://docs.nestjs.com/techniques/queues
https://gemini.google.com/app/62faa183be0044cd?utm_source=app_launcher&utm_medium=owned&utm_campaign=base_all
https://gemini.google.com/app/139735d1ba7886a3?utm_source=app_launcher&utm_medium=owned&utm_campaign=base_all



Protocol:
    We stream objects from the client to the agent.

model:agent
def:
    ...

# Models:

## Containers

agent 
    
## General message (just text)

narrative

## Input request

select
multiselect
decision
proposal

## Actions

definition
preview
previewReady





# Toools:

model: tool
def:
    description: prompt_id | Prompt
    inputSchema: schema_id | Schema
    outputSchema: schema_id | Schema
    type: toolType

export const RedisTool = tool`
    description: prompt_id | Prompt
    inputSchema: schema_id | Schema
    outputSchema: schema_id | Schema
`(class {
  // Code part
  execute() {
    return this.priority * 10;
  }

});

Tasks
=====

Editor
    Test without UI
    Prompts
        Expose tools YAML
            MCP
            Internals
        Editor Role
        Crafts
            EiBot editor crafts
                Speciality: Write Eivolet
            Update to rest of crafts
            Implement: $references, see below <DONE>
    Lingv build UI
        New UI
        New Auth Token for Chat API
    Preview
        Contentkit
            Preview Rendering
        Lingv or Chat
            Preview API (Assets API)
    Anvil
        Call forge
        Move Anvil to a model where it owns the prompts too and sends Evilet definition complete [REVIEW]
    API Cloud        
        Deprecate Editorial API and move the important parts to Forge

-----> Done
    Chat Service
        editor API <DONE>
            calls to Eibot capability Editor <DONE>
    Eibot capability at Core
        Models <DONE>
        Eibot Editor <DONE>
        Memory <DONE>
            methods <DONE>
            Redis buckets <DONE>
                Memory <DONE>
                    Eivolet <DONE>
                    Messages <DONE>
                    Capability itself <DONE>
    Foundry <DONE>
        new foundryAgent support <DONE>
        Tool support <DONE>
            tool registry <DONE>
            Definitions <DONE>
                Tools list:
                    publish (Library, MCP)<DONE>
                    save eivolet (locals)  <DONE>
                    recover eivolet (locals)  <DONE>
                    getDomains (custom) <DONE>
                    getCultures (custom) <DONE>
                    getProgLangs (custom) <DONE>
                    getPlatforms with prog langs(custom) <DONE>
    Crafter
        Craft prompt extension <DONE>
        CraftAgent with tools and streaming support<DONE>
    Tools
        Model <DONE>
    Contracts Updates
        Eivolet definition support in line objects (no references) <DONE>
    API Cloud
        Library API<DONE>
            Publishing<DONE>
            Expose methods as MCP and Rest<DONE>



I will add to the Role two new fields with ids for crafts and tools. Eibot will load the current session and the domain and old messages. It will pass to Foundry the role, the domain, the new message, the current messages and the response object. Foundry will load and merge the crafts from the role and the ones associated to the domain, and the tools.  It will pass to Crafter: the instructions, the tools, the new message, the old messages, the response and handler to store messages. Crafter will call Vercel with tools, messages and instructions. Then it will call the handler to store the messages from onFinish. And at last It will pipe the response.

The mechanism we already have ... but we could have a pollution problem by adding crafts that we don't need. So we have two options, in the role we can list optional crafts and we use them to filter out by domain or we add a field to the craft defining for what role they are available. I think the optional list is better


# References

So $references under def inlines the file content into def.prompt, using the directory of source as the base path for resolution:
yaml# before resolution
def:
  $references: prompts/bundlewriter.md

# after resolution
def:
  prompt: |
    ... content of bundlewriter.md ...
Convention-based — $references under def always targets def.prompt. The base path is derived from source (the file being parsed). Simple, no extra config needed.