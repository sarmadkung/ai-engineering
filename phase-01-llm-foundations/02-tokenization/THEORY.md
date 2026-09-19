# Topic 2 — Tokenization

**Why this topic:** in Topic 1 one token was one character. Real LLMs don't do that.
This topic explains what they do instead, and you build it.

---

# Part 1 — Theory

## 2.1 Words vs tokens

Two obvious ways to cut text into pieces, both bad:

**One token per character.** Vocabulary is tiny (~100 entries). But "hello" becomes 5
steps instead of 1, so the model needs a much longer context to see the same amount of
text, and it must learn spelling from scratch.

**One token per word.** Now "hello" is 1 step. But the vocabulary explodes — English has
millions of word forms — and you will still meet words you've never seen ("unfriending",
"Sarmad", "gpt-4o"). An unknown word becomes `<UNK>` and its meaning is lost.

So: characters are too small, words are too many, and neither handles new text well.

## 2.2 Subword tokenization: the compromise

Split text into **pieces of words**, chosen by how common they are:

```
"tokenization"  ->  ["token", "ization"]
"unfriending"   ->  ["un", "friend", "ing"]
"Sarmad"        ->  ["Sar", "mad"]
```

Common words stay whole ("the" = 1 token). Rare words break into familiar parts. Nothing
is ever unknown, because the pieces bottom out at single characters. Vocabulary lands
around 30,000–100,000 entries.

This is why token counts look odd in practice: a common English word is ~1 token, but a
rare name, a long number, or non-English text may be 4–5.

## 2.3 BPE (Byte Pair Encoding)

The algorithm that finds those pieces. Training it is a loop:

1. Start with every character as its own token.
2. Count every **adjacent pair** of tokens in the corpus.
3. Find the most frequent pair and **merge** it into one new token.
4. Record that merge in an ordered list.
5. Repeat until you have as many tokens as you want.

Tiny example on the word `low low lower`:

```
start:      l o w   l o w   l o w e r
merge 1:    "l"+"o" is most common  ->  lo
            lo w   lo w   lo w e r
merge 2:    "lo"+"w"                ->  low
            low   low   low e r
```

The **merge list, in order**, is the whole trained tokenizer. To tokenize new text you
start from characters and re-apply the merges in the same order. Order matters: `low`
could not be made before `lo` existed.

## 2.4 SentencePiece and byte-level BPE

Two practical problems BPE alone has, and how real systems fix them:

- **Spaces.** Is "dog" after a space the same token as "dog" after a quote? Most
  tokenizers attach the leading space to the token (`" dog"`), so detokenizing is exact
  and reversible.
- **Any possible input.** What about emoji, Arabic, or corrupt bytes? **Byte-level** BPE
  starts from the 256 possible bytes instead of characters, so *any* input can be
  encoded. This is what GPT-style models use.

**SentencePiece** is a library that does this training and encoding, treats text as a raw
stream (no pre-splitting on spaces), and is standard in the Llama/Mistral family.

## 2.5 Token IDs and vocabulary

Models do not work with strings. Each token gets an integer **ID**:

```
"the"  -> 1820        text -> tokens -> IDs -> model -> IDs -> tokens -> text
" dog" -> 3290
```

The **vocabulary** is the two-way mapping between tokens and IDs. Its size is a hard
constraint on the model: the final layer produces exactly one number per vocabulary
entry, so vocabulary size directly sets the model's output width and a big chunk of its
parameters.

Also: **special tokens** are added by hand — end-of-text, padding, and (for chat models)
markers for where a system/user/assistant turn begins. Chat "roles" are just text wrapped
in special tokens.

## 2.6 Why this matters commercially

Tokens are the billing unit, the context unit, and the speed unit. Every API charges per
token, every context window is measured in tokens, and every "words per second" is really
tokens per second. Compressing text into fewer tokens is a direct cost saving — and
languages that tokenize badly cost more for the same meaning.

---

# Part 2 — Questions to implement

Build `bpe.py` in this folder. Standard library only. Use `corpus.txt` from Topic 1 (or a
bigger text of your own — bigger is better here).

### Q1. Character baseline, measured
**Build:** tokenize your corpus by character and by whitespace-separated word. For each,
report vocabulary size and total token count.
**Check:** words give far fewer tokens and a far larger vocabulary.
**Explain:** name one concrete failure of each scheme on text it wasn't trained on.

### Q2. Count the pairs
**Build:** represent the corpus as a list of token lists (start: characters). Write a
function that counts every adjacent pair across the whole corpus and returns the most
frequent one.
**Check:** on a small hand-made input, the answer matches what you count by hand.
**Explain:** why pairs, rather than the most common triple or the most common character?

### Q3. Train the merges
**Build:** the loop from 2.3. Repeat "find best pair → merge it everywhere → record the
merge" until you reach a target vocabulary size (try 300, then 1000). Keep the merge list
**in order**.
**Check:** print the first 20 merges. They should be letter pairs, then fragments, then
whole common words. Vocabulary size grows by exactly one per merge.
**Explain:** why must the merge list stay ordered?

### Q4. Encode
**Build:** `encode(text)` — start from characters, apply your merges in training order,
return tokens.
**Check:** a word that appears often in training becomes 1–2 tokens; a word you invent
becomes several. Text with characters never seen in training still encodes without
crashing (decide how: byte fallback, or a character-level floor).
**Explain:** what stops your encoder ever producing an "unknown token"?

### Q5. Decode, and prove it's lossless
**Build:** `decode(tokens)` back to text. Handle spaces so the round trip is exact.
**Check:** `decode(encode(x)) == x` for: plain prose, text with double spaces, text with
newlines, text with an emoji.
**Explain:** where did you decide spaces belong, and why does that choice make decoding
unambiguous?

### Q6. IDs and vocabulary
**Build:** assign every token an integer ID. Write `token→id` and `id→token` maps, and
`encode_ids` / `decode_ids`. Add two special tokens with reserved IDs.
**Check:** IDs round-trip. Special token IDs never collide with real text.
**Explain:** if your vocabulary doubles, what gets more expensive in the model itself?

### Q7. Compression measurement
**Build:** for vocabulary sizes 300, 1000, 5000, tabulate: tokens per 1000 characters,
and average tokens per word.
**Explain:** where do the gains flatten out, and what is the cost of pushing vocabulary
higher?

### Q8. Plug it into Topic 1
**Build:** swap your Topic 1 n-gram model's tokenizer for this one. Change nothing else.
**Check:** it still trains, generates, and reports perplexity.
**Explain:** perplexity numbers are now **not comparable** to Topic 1's. Why not? (Think
about what "per token" means when tokens changed size.) Also: with BPE tokens, what does
an `order = 5` context window now cover in characters?

### Q9. Compare with a real tokenizer (optional, needs install)
**Build:** if you set up a virtual environment later, run the same text through
`tiktoken` and compare token counts with yours.
**Explain:** where does a production tokenizer beat yours, and why?

---

# Done when you can answer

1. Why not characters? Why not words?
2. What exactly does BPE training produce, and what does encoding do with it?
3. Why can a subword tokenizer never hit an unknown word?
4. What is a token ID, and how does vocabulary size affect the model?
5. Why is a rare name more tokens than a common word — and why does that cost money?
6. Why did your Topic 1 perplexity change when you swapped tokenizers?

Write answers in `notes.md`.
