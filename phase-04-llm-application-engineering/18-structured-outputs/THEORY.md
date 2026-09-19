# Topic 18 — Structured Outputs

**Why this topic:** an LLM that returns prose is a chat toy. An LLM that returns data your code
can rely on is a component you can build systems from. This topic is the bridge between
"language model" and "software".

---

# Part 1 — Theory

## 18.1 The problem

You ask for JSON. Sometimes you get:

````
Sure! Here's the JSON you requested:
```json
{"name": "Sarmad", "age": 30,}
```
Let me know if you need anything else!
````

Three failures in one: conversational wrapper, markdown fence, trailing comma. Your
`json.loads` throws in production at 3am. And it will be *intermittent*, because generation is
sampled — which makes it the worst class of bug.

Why it happens is clear from Topic 1: the model samples tokens, and a plausible-looking wrapper
is a plausible continuation. Nothing in the generation loop enforces a grammar.

## 18.2 The escalation of solutions

Four approaches, from weakest to strongest. Know all four, because you'll meet all four.

**1. Ask nicely.** "Respond with only valid JSON." Works most of the time. "Most" is the
problem.

**2. Ask nicely + parse defensively.** Strip fences, find the outermost braces, then parse.
Necessary insurance; not a solution.

**3. Validate and retry.** Parse, validate against a schema, and on failure send the error back
and ask for a correction. Reliable in practice, at the cost of extra calls and latency.

**4. Constrained decoding (structured outputs).** The provider restricts sampling so that only
tokens which keep the output valid under your schema can be chosen. Invalid output becomes
*impossible* rather than unlikely.

This last one is the real answer, and it's worth understanding *how* it works, because it
follows directly from Phase 1: at each step the sampler masks out every token that would break
the grammar, then samples from what remains. It is Topic 5's top-k filtering with the filter
derived from a schema instead of from probability.

## 18.3 Schemas

A schema is a machine-readable contract: field names, types, which are required, allowed
values, nesting. **JSON Schema** is the format APIs speak.

In Python you write it as a **Pydantic** model and let the library emit the schema — that gives
you one definition used for the API contract, for parsing, and for type checking:

```
class Ticket(BaseModel):
    category: Literal["billing", "technical", "other"]
    urgency: int          # with a constraint: 1..5
    summary: str
```

Design rules that matter more than they look:

- **Use enums/literals for closed sets.** A free-text `category` will eventually return
  "Technical Support (billing related)".
- **Make optional things genuinely optional**, so the model isn't forced to invent a value.
  This is where hallucinated fields come from.
- **Flat beats deeply nested.** Deep nesting raises failure rates and is harder to validate.
- **Name fields descriptively.** The field name is a prompt: `is_urgent` is clearer to the model
  than `flag2`.
- **Add an escape hatch** — an `unknown` enum member or a nullable field — or the model must
  choose wrongly when the answer isn't in your schema.

## 18.4 Validation is not optional, even with constrained decoding

Constrained decoding guarantees the output is *shaped* correctly. It does not guarantee the
content is *correct*:

- the schema says `urgency: int`, and you get 3 when the right answer was 5;
- the schema says `date: str`, and you get "next Tuesday";
- every required field is present and one of them is invented.

So: validate types and ranges with the schema, then validate *semantics* with your own checks
(does the referenced id exist? is the total equal to the sum of the lines?). Structured output
converts *format* errors into *content* errors, which is a big improvement and not a
resolution.

## 18.5 Typed responses in your code

The payoff is that the model's output enters your program as a typed object rather than a
string: your editor autocompletes it, type checkers catch mistakes, and the boundary between
"LLM guessed this" and "my code computed this" is explicit.

Current Claude SDKs support this directly — a `parse`-style call that validates the response
against your schema and returns the typed object, and `output_config.format` on a normal
`create` call to constrain the format. (The older `output_format` parameter is deprecated; use
`output_config`.)

## 18.6 Tool schemas are the same idea

A tool/function definition is a schema describing the arguments you'll accept. When the model
calls a tool, its arguments are a structured output — and providers support a strict mode that
guarantees those arguments validate against the schema exactly.

So this topic is also the foundation of Phase 6. A tool is: a name, a description (which is a
prompt — the model chooses tools by reading it), and a parameter schema.

## 18.7 Where structured output earns its keep

Extraction (documents → records), classification with confidence, routing decisions, form
filling, generating API calls, and any step whose output feeds another program — which in
practice means every agent step. Prose output is for humans; structured output is for
everything else.

---

# Part 2 — Questions to implement

Build `extract.py` and `schemas.py` here. Install `pydantic`. Use a real extraction task: pick
messy source text (emails, invoices, job postings, recipes) and a target record shape.

### Q1. Provoke the failure
**Build:** ask for JSON with plain prompting, 20 times, at temperature 0.7. Count how many
outputs `json.loads` accepts.
**Check:** record the failures verbatim.
**Explain:** categorize the failure types you saw. Why does temperature affect this?

### Q2. Defensive parsing
**Build:** a parser that strips markdown fences, trims prose, extracts the outermost JSON
object, and then parses.
**Check:** re-run Q1's 20 outputs through it and report the new success rate.
**Explain:** which failure class can defensive parsing never fix?

### Q3. A schema with Pydantic
**Build:** a model for your record, using `Literal` for closed sets, constrained ints, and
genuinely optional fields. Print the generated JSON Schema.
**Explain:** why did you make each optional field optional? What would the model do if it were
required and the source text lacked it?

### Q4. Validate and retry
**Build:** a loop: call → parse → validate → on failure, send the validation error back and ask
for a correction, up to 3 attempts. Log attempts per input.
**Check:** report the distribution of attempts over 20 inputs.
**Explain:** what is the latency and cost cost of this approach? When is it the right choice
anyway?

### Q5. Constrained decoding
**Build:** the same extraction using the provider's structured-output support (schema-constrained
response, or the typed parse helper).
**Check:** 20 out of 20 parse without any defensive code. Compare cost and latency to Q4.
**Explain:** what class of error disappeared entirely, and what class remains?

### Q6. Shaped but wrong
**Build:** deliberately feed ambiguous or incomplete source text.
**Check:** the output still validates.
**Explain:** show one example where every field is valid and at least one is wrong or invented.
State the lesson in one sentence.

### Q7. Semantic validation
**Build:** validators beyond types — a date that must parse and be in the past, a total that
must equal the sum of line items, an id that must exist in a list you control.
**Check:** each rejects a crafted bad case.
**Explain:** which of these could a schema alone have caught? Which could not, ever?

### Q8. Enum vs free text
**Build:** the same classification with `category: str` and with `category: Literal[...]`. Run
30 inputs through each.
**Check:** count distinct values produced by the free-text version.
**Explain:** report both. What happens to downstream code with the free-text version?

### Q9. Escape hatch
**Build:** add an `unknown` category and a nullable confidence. Feed 5 inputs that genuinely
don't fit your categories.
**Check:** the model uses the escape hatch rather than mis-classifying.
**Explain:** what did the model do *before* you added it?

### Q10. Nesting cost
**Build:** a flat schema and an equivalent deeply-nested one (3+ levels). Run both on the same
20 inputs.
**Explain:** compare failure rates, token counts, and how hard each was to validate. Which
would you ship?

### Q11. A tool schema
**Build:** define one tool schema (name, description, parameters) for a function like
`search_orders(customer_id, status, limit)`. Ask the model to produce arguments for a natural
language request.
**Check:** arguments validate against the schema.
**Explain:** rewrite the *description* badly (vague, or wrong about what it does) and observe the
effect. What does that tell you about tool descriptions as prompts? (Phase 6 continues here.)

### Q12. A small extraction pipeline
**Build:** 20 messy documents in → validated typed records out, with per-record status
(succeeded / retried / failed with reason), and a summary report.
**Check:** it never crashes on bad input; every failure is recorded with a reason.
**Explain:** what is your end-to-end success rate, and what would you have to do to raise it?

---

# Done when you can answer

1. Why does an LLM return malformed JSON *intermittently*?
2. What are the four levels of solution, and what does each cost?
3. How does constrained decoding work, in terms of Topic 5's sampler?
4. What does constrained decoding guarantee, and what does it not?
5. Why use enums instead of free-text strings?
6. Why does a schema need an escape hatch?
7. Why is a tool definition the same thing as a structured output?

Write answers in `notes.md`.
