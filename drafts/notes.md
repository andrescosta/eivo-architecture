- Giving a prompt with words ending in a specific preffix will be a problem for the LLMs. [Searching how this can be done.]
- Context window:
The LLM doesn’t just take any text and produce any text. It takes a text with a number of tokens that’s smaller than the context window size, and its completion is such that the prompt plus the completion cannot have more tokens than the context window size either. Context window sizes are typically measured in thousands of tokens, and that’s nothing to sneeze at, in theory: it’s several, often dozens, and sometimes hundreds of pages of A4 size. But practice tends to sneeze at it nevertheless: however long your context window, you’ll be tempted to fill it and overfill it, so you need to count tokens to stop that from happening.





Read:
Tokenizer:
https://huggingface.co/docs/transformers/main_classes/tokenizer
https://github.com/openai/tiktoken


Refs: 
https://platform.openai.com/tokenizer
