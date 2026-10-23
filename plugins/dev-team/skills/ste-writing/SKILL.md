---
description: 'Write or rewrite prose in ASD-STE100 Simplified Technical English (STE). Use it for PRDs, research reports, architecture documents, plans, issues, review reports, commit messages, PR descriptions, code comments, and user documentation.'
name: ste-writing
---

# Simplified Technical English (ASD-STE100)

ASD-STE100 is the ASD standard for technical documentation. Issue 9 (January 2025) is the
current issue. The standard has two parts: a set of writing rules and a dictionary of approved
words. This skill gives the writing rules in a form for software text. It also gives a short list
of plain words. The list is not the official dictionary.

## Scope

- Apply STE to prose: documents, reports, issue text, commit messages, PR descriptions, code
  comments, and messages to the user.
- Do not change code, commands, file paths, identifiers, configuration keys, or quoted text.
  Do not change error strings that tests or callers compare.
- Use technical nouns and technical verbs from the domain (rules 1.5 and 1.12). Examples are
  `goroutine`, `mutex`, `lint`, `refactor`, `deploy`, `merge`, and product names.
- If the project has a glossary, use its terms with the meanings that it gives.
- Headings, table cells, and labels can be names or short phrases.
- The word limits apply to each sentence. Do not remove facts, conditions, or limits to make
  the text shorter. Write more sentences.

## Words

- Use one word for one meaning. When you select a term for a concept, use only that term.
  For example, do not write "issue", "ticket", and "task" for the same thing.
- Use each word with only one meaning. For example, if "check" is a verb for "examine", do not
  also use it as a noun for "a bank check".
- Use a verb to show an action. Do not change the verb into a noun: write "Configure the
  timeout", not "Do the configuration of the timeout".
- Use the plain word. See [Plain Words](#plain-words).
- Do not use a phrasal verb when one verb has the same meaning: "find", not "find out".
- Do not use contractions: write "do not", not "don't".
- Do not use Latin abbreviations. Write "for example", "that is", or "and other", or write the
  full list.
- Do not use a pronoun ("it", "this", "they") if the reader cannot find its noun immediately.
  Write the noun again.

## Noun Clusters

- Do not write a noun cluster of more than three words. Use a preposition or a verb to break
  it: write "the timeout value of the connection pool", not "connection pool timeout value".
- If a technical name has more than three words, write the full name one time. Then use a
  short form.

## Verbs

- Use only these verb forms: the infinitive, the imperative, the simple present, the simple
  past, the simple future, and the past participle.
- Use the past participle only as an adjective or with "is", "are", "was", or "were" in the
  passive.
- Do not use the perfect tenses or the progressive: write "The test failed", not "The test has
  been failing".
- Use the "-ing" form only in a technical noun, for example "logging" or "error handling". Do
  not write "When running the tests". Write "When you run the tests".
- Use the active voice in instructions.
- Use the active voice in descriptions where possible. Use the passive only when you do not
  know the agent, or when the agent is not important.
- Do not use a helping verb to make a complex verb structure: write "The service can retry",
  not "The service should be able to have retried".

## Sentences

- Write one topic in each sentence.
- Do not remove words to make a sentence shorter. Keep the subject, the verb, the article, and
  "that".
- Do not use an arrow or a symbol in place of a verb. Write "gives", "goes to", or "becomes".
- Use connecting words (if, then, because, but, also) to link sentences.
- Use a vertical list for complex text, for example three or more steps or conditions.

## Instructions

- Write a maximum of 20 words in an instruction.
- Write instructions in the imperative: "Run the tests".
- Write one instruction in each sentence. Write two instructions in one sentence only if the
  reader does them at the same time.
- Put a condition before the instruction. Use a comma after the condition: "If the build fails,
  read the first error".
- Put the steps of a procedure in a numbered list, in the sequence that the reader does them.
- Use a note to give information. Do not put an instruction in a note.

## Descriptions

- Write a maximum of 25 words in a descriptive sentence.
- Start each paragraph with a topic sentence.
- Write one topic in each paragraph.
- Write a maximum of six sentences in a paragraph.
- Give information gradually. Put the most important information first.

## Safety Instructions

Use a safety instruction before a step that can cause data loss, a security risk, or a change
that you cannot reverse.

- Use **Warning** for a risk of data loss, a security risk, or an irreversible change.
- Use **Caution** for a risk of damage that you can repair.
- Start with a clear command. Then give the risk in a short sentence.

Example:

> **Warning:** Do not force-push to `main`. A force-push deletes commits that other people use.

## Punctuation and Word Count

- Do not use semicolons. Write two sentences.
- Do not use a dash to join two clauses. Write two sentences, or use a comma and a connecting
  word.
- Use a colon to introduce a list or an example.
- Use parentheses only for a short addition, for example an abbreviation or a file path.
- When you count words, count each number, identifier, abbreviation, and name as one word.

## Plain Words

Use the word in the right column. This list is a starting point. It is not the ASD-STE100
dictionary.

| Do not use | Use |
|------------|-----|
| additional | more |
| approximately | about |
| assist, facilitate | help |
| commence, initiate | start |
| ensure | make sure |
| e.g., i.e., etc. | for example, that is, the full list |
| in order to | to |
| numerous | many |
| obtain | get |
| prior to, subsequent to | before, after |
| regarding | about |
| sufficient | enough |
| terminate | stop |
| utilize, leverage | use |
| via | through, with |
| whether | if |

## Examples

| Before | After |
|--------|-------|
| Leveraging the cache layer, response times have been significantly reduced. | The cache makes the response time shorter. |
| Unclear scope → state your assumption and proceed. | If the scope is not clear, write your assumption and continue. |
| Don't re-run tests; lint already passed. | Do not run the tests again. The lint step passed before this step. |
| Ensure the user session token refresh logic is idempotent. | Make sure that the logic that refreshes the session token is idempotent. |

## Check

Before you return the text, do these checks:

1. Find each sentence that is longer than 20 words (instructions) or 25 words (descriptions).
   Divide it.
2. Find each "-ing" verb, perfect tense, and progressive form. Change it.
3. Find each passive verb. Change it to active if you know the agent.
4. Find each noun cluster of more than three words. Break it.
5. Find each semicolon, contraction, Latin abbreviation, and arrow in prose. Remove it.
6. Make sure that each term has one meaning in the full text.
