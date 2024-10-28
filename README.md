# ln-diff

## Overview

The ln-diff format is a specialized diff format designed for precise line-based code modifications. It uses "editblocks" to explicitly define code changes while maintaining strict line number references.

## Why?

If you are working with LLMs and to generate code, you have probably faced the challenge that while they may be good at generating new code, they struggle much more to apply it to *existing* files and codebases. Having tried multiple approaches like instructing it to generate edits in the well known [unidiff](https://www.gnu.org/software/diffutils/manual/html_node/Example-Unified.html) format or some variation of [diff-match-patch](https://github.com/google/diff-match-patch/wiki/Unidiff).

These formats are too algorithmically complex for LLMs, though, because they were designed with various additional goals like human-readability, conciseness and patching efficiency.

I propose a different approach here: to leverage LLMs strong recitation capabilities to generate patches that apply cleanly to the source format and provide enough it with enough context for generating high-quality replacements.

At the expense of human readability and smaller patches, I observed that assistants that can generate patches that apply correctly preserve the same code quality as if asked to generate a full file from scratch.


## Format Specification

The format is simple at it's core:

### The EditBlock Structure
1. File path
2. Opening tag `<editblock>`
3. Remove section marker `<<<<<<< REMOVE`
4. Numbered lines to remove
5. Separator `=======`
6. Lines to insert
7. Insert section marker `>>>>>>> INSERT`
8. Closing tag `</editblock>`

### Rules and Constraints
- SOURCE files use consecutive, unbroken line numbers
- The REMOVE section uses consecutive, unbroken line numbers
- The REMOVE section line numbers and contents must match the SOURCE file exactly
- The INSERT section uses aligned space padding
- All lines use an aligned pipe separator (│)
- One continuous block of lines per editblock section
- Independent blocks (no sequential dependencies)

## Design Decisions

This format was conceived with LLMs in mind, with the most common use case being code generation and assistance. As such, many of its design decisions were made to guide code generation models attention into sourcing from the correct content and generating high quality responses.

**Q: Why line numbers?**

A: Line numbers serve multiple purposes:
- Line numbers in *both* the source and editblocks helps llms recite the exact line more accurately.
- It is algorithmically simpler for LLMs to understand.
- It allows clients to stream the response and detect the exact line an edit will apply, removing and inserting lines as they come, theoretically reducing the time-to-response. (TODO: Need a working demo to verify this)

**Q: Then why no line numbers in the INSERT section?**

A: Technically, prefixing line numbers to INSERT section lines should not prevent the line to be inserted into the document. But as mentioned, prefixed line numbers "prime" the model to recite the source line. So, if the conversation is filled with unnecessary line numbers in both user and assistant-generated code then it becomes noise and the assistant might choose its own content as source content. This breaks the format's contract and decreases the quality of the generated code.

**Q: Why numbered source files?**

A: Source files need to be prefixed the line numbers so that the lines to REMOVE can be recited accurately and sequentially.

**Q: Why use continuous blocks?**

A: Placing code in the REMOVE section means placing it in a very privileged position for AI code generation: close to the generated replacement in the INSERT section. Continuous, recited code probably retains more of the underlying correctness and semantics. This kind of high quality context is what we need to be close to the generated code.

**Q: Why use an aligned pipe?**

A[ The "pipe" symbol used is not your regular keyboard pipe but actually unicode's [U+2502](https://www.htmlsymbols.xyz/unicode/U+2502) called "Box drawings light vertical". This is a common symbol used to draw lines in text-only applications and may be understood by LLMs (TODO: verify this) to be the divider between the line number and the actual line contents. Alignment may help with reducing indentation errors in generated code.

**Q: Why the name?**

A: ln-diff stands for line-number-diff. But even though the format is called like that for human reference, I found no use in mentioning this to language models. In fact, just mentionin the word "diff" will poison the reasoning abilities of smaller LLMs, making it generate edits in the Unified Diff format or simply degrading the quality of ln-diff editblocks. I avoided the word "diff" in the [first]() version, and used the word *editblock* (instead of *hunk* or other similar words, for the same reason).

**Q: Why independent, non-sequential editblocks?**

A: Again, this comes down to leveraging the LLM strengths: it is easier for them to recite numbered lines from source if the line numbers are considered immutable, even if multiple editblocks have already been generated.

Even with these considerations, additional prompting is commonly required to make the model actually follow this simple format. The smaller/older the model is the harder it will be. This issue should be no surprise and is not unique to this format.

I did most of the testing with openai/gpt-4o-mini-2024-07-18. I consider this model to be a good baseline, eventually reaching ([with extensive prompting and additional guidelines]()), more than 99 percent conformance to the format on Aider's [code editing benchmark](https://github.com/Aider-AI/aider/tree/main/benchmark), comparable to the state-of-the-art for code generation anthropic/claude-3-5-sonnet-20241022.

I will not adventure trying to teach the format to smaller or older models, but feel free to try. Trends suggest that subsequent models will require less prompting to use the format correctly.

You can read more about Aider's benckmark and previous attempts at similar edit formats here:

- [GPT code editing benchmarks](https://aider.chat/docs/benchmarks.html)
- [Aider LLM Leaderboards](https://aider.chat/docs/leaderboards/)

## Implementation Notes

### Relocation
Relocation is done using two separate editblocks: one to remove from the original position, and one to insert in the new position.

### Appending
Appending content to the end of the file is possible by leaving the REMOVE section empty.

### Parsing Requirements
This format was developed with LLM usage in mind, so some parsing flexibility is unavoidable as some models will be more capable of following the format than others.

Some of the things I found useful:

- Don't enforce INSERT section prefixing or correct pipe alignment.
- Try matching REMOVE sections three times: once as-is and then with a line offset of plus/minus one.
    Models are usually off by one line number, but be careful with this: Offsetting the whole section as a whole is safer than offsetting each line independently.
- Try matching each line in the REMOVE section two times: once as-is and once using a similarity algorithm.
    I used difflib with a similarity ratio of > 0.95 but you can be more sophisticaded.

Ideally this flexibility will be less important as models get more capable.

> [!NOTE]
> This flexibility only applies to the client's side, though. Depending on the model's instruction-following capabilities, the wording of the assistant's system prompt must range from 'follow the editblock format' to instructing it to treat the format as an unbreakable, sacrosanct contract. See the /examples folder for examples for common language models.

### Applying non-sequential editblocks

Even though editblocks are non-sequential, clients will indeed need to track how many have already been applied, or store and apply all of them at the same time, so that they modify the source file cleanly. This is expected and in line with moving the algorithmic complexity away from the assistant's reasoning into the actual client software.

## Current Status

- This format will evolve based on futher benchmarking
- Need to add more examples on how to properly use the format with popular LLMs
