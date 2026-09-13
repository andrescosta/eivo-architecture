https://n9o.xyz/posts/202603-building-gordon/


Using Agents to Improve the Agent
Remember the sneak peek about not writing prompts by hand? Here’s what that looks like in practice.

We built a custom agent - running on a more powerful model like Claude Opus 4.6 - whose job is to improve Gordon’s system prompt. The workflow: give it Gordon’s current agent definition, a set of failing evals, and the results. The agent analyzes the failures, proposes prompt changes, and outputs an updated YAML. We run the eval suite against the new version. If scores improve and nothing regresses, we ship it.

This creates a tight improvement loop. A user reports that Gordon asks too many clarifying questions instead of just reading the files? We add an eval for it, point the optimizer agent at the failure, and let it figure out the right prompt change. It might add a rule like “ALWAYS read files directly using filesystem tools. NEVER ask users to paste content.” - which is exactly the kind of specific, actionable instruction that makes the difference between a good agent and a frustrating one.

Using a more powerful model as the “teacher” to improve the “student” is deliberate. Opus has the reasoning capacity to understand subtle behavioral issues and craft precise instructions that steer Haiku in the right direction. It’s agents all the way down.

Most of the detailed behavioral rules in Gordon’s prompt - the banned filler words, the file access patterns, the response sizing guidelines, the debugging sequences - were written or refined by the optimizer agent, not by a human. We set the direction and define what good looks like through evals. The agent figures out how to get there.