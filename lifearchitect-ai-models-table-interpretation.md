# Life Architect AI - Models Table - Interpretation

Interpretation of the data attributes found on [lifearchitect.ai - Models Table](https://lifearchitect.ai/models-table/)

**Parameters (B)**

* Think of parameters as the "knobs and dials" inside an AI model — they're the numbers the model learned during training that determine how it responds to everything.  
* The "B" stands for billions.  
  * A model with 7B parameters has 7 billion of these learned values  
  * More parameters generally means the model can store more knowledge and handle more complex tasks  
*  It's roughly analogous to "brain size" — bigger isn't always better, but it's a common measure of model capacity

**Tokens Trained (B)**

* A "token" is roughly a word or part of a word (e.g., "running" might be 1 token, "unbelievable" might be 2-3). Tokens Trained means how much text the model read and learned from during training. Again, "B" \= billions.  
* A model trained on 1,000B tokens read through roughly a trillion words worth of text  
* More tokens trained generally means the model has seen more of the world's knowledge  
* It's like the difference between someone who read 100 books vs. someone who read 10,000 books

*A small brain that read a lot, or a big brain that read a little — both matter, and the combination shapes how capable the model is.*

**Ratio Tokens:Params**

* Params is simply dividing one by the other:  
* Tokens Trained ÷ Parameters \= Ratio  
* So if a model has 7B parameters and was trained on 140B tokens, the ratio is 20\.  
* Why does this ratio matter?  
  * A landmark 2022 research paper (called the ["Chinchilla" paper](https://arxiv.org/pdf/2203.15556)) figured out that there's a roughly optimal ratio for training efficiency. Their finding: you should train with about 20 tokens per parameter to get the best performance for your compute budget.  
    * Ratio below 20 — the model is "undertrained." It has a big brain but didn't read enough books to fill it. You left potential on the table.  
    * Ratio around 20 — considered the classic "Chinchilla optimal" sweet spot.  
    * Ratio well above 20 — the model read a lot relative to its size. More recent research (like Meta's LLaMA models) found that training smaller models on far more data can actually be very efficient, especially for inference costs.

*The ratio is essentially asking: "Did this model reach its full potential, or was it undertrained?" A very high ratio can also mean the developers intentionally built a smaller but well-read model because it's cheaper to run day-to-day, even if training took longer.*

**ALScore**

* The ALScore is a single number invented by the table's author (Alan D. Thompson, hence "AL") to give you a quick, one-glance power rating for a model.  
* How it's calculated:  
  * Square Root of (Parameters × Tokens Trained) ÷ 300  
  * It mashes together both the brain size (parameters) and the reading (tokens trained) into one score  
  * A score of 1.0 or higher meant you had a seriously powerful model as of mid-2023  
  * Higher \= more powerful  
* Parameters alone and tokens alone can each be misleading. ALScore forces you to consider both together. A huge model that was barely trained scores lower than a well-trained mid-sized model.

*One caveat worth knowing: because AI has advanced so fast, the "1.0 \= powerful" benchmark is anchored to mid-2023. Models released today routinely score much higher, so you'd want to compare ALScores relative to each other rather than treating 1.0 as the absolute bar anymore.*

**MMLU (Massive Multitask Language Understanding)**

* A test covering 57 different subjects — history, math, law, medicine, science, coding, ethics, and more  
  * [https://arxiv.org/pdf/2009.03300](https://arxiv.org/pdf/2009.03300)  
* Each question is multiple choice (A/B/C/D)  
* The score is simply: what percentage did the model get right (0–100)  
* A random guesser would score \~25. An average human scores around 57\. Top AI models now score in the high 80s–90s.

**MMLU-Pro**

* Released in 2024 because the original MMLU got too easy — top models were acing it, making it hard to tell them apart  
  * [https://arxiv.org/pdf/2406.01574](https://arxiv.org/pdf/2406.01574)  
*   Harder questions, and 10 answer choices instead of 4, so guessing is much less likely to work  
*   Same idea though: percentage correct out of 100

Why do these matter?

They give you a rough, apples-to-apples way to compare models across general knowledge and reasoning — not just one topic, but dozens at once.

Think of MMLU and MMLU-Pro as standardized tests for AI models — like the SAT or ACT for humans.

*One thing to be aware of: these benchmarks have become somewhat controversial because AI labs know about them and can (intentionally or not) train their models in ways that inflate the scores. So a high MMLU score is a good sign, but it's not the whole story.*

**GPQA (Google-Proof Q\&A)**

* [https://arxiv.org/pdf/2311.12022](https://arxiv.org/pdf/2311.12022)  
* The name says it all: these are questions so hard that even googling the answer won't easily help you  
* Written by PhD-level experts in biology, chemistry, and physics  
* The questions require genuine deep reasoning, not just memorizing facts  
* Regular smart humans with access to Google score around 30–40%. PhD experts in the relevant field score around 65%. Top AI models are now pushing into the 70s–80s.

**HLE (Humanity's Last Exam)**

* [https://arxiv.org/pdf/2501.14249](https://arxiv.org/pdf/2501.14249)  
* Released in early 2025 and considered one of the hardest AI benchmarks ever created  
* Crowdsourced from thousands of experts across hundreds of fields worldwide  
* The name is a bit tongue-in-cheek — it's meant to be the final exam that AI can't pass yet  
* When it launched, even the best models scored in the single digits to low teens. Scores have been climbing since.  
* Think of it as the benchmark designed to stay ahead of AI capability

**Training Dataset**

This field tells you what the model was fed during training — where did all those billions of tokens actually come from?

Common sources you'll see listed:

* Web crawls (basically a big chunk of the internet — Reddit, Wikipedia, news sites, blogs)  
* Books and academic papers  
* Code repositories (like GitHub)  
* Synthetic data — this is an increasingly common one worth noting: it means AI-generated text was used to train other AI. Think of it as an AI learning from its own homework, or a student learning by reading textbook answer keys rather than real-world examples.

*The field is described as a "rough guide" because most labs are deliberately secretive about exactly what they trained on (for legal and competitive reasons).*

**Arch (Architecture)**

This describes the fundamental structural design of how the model is built under the hood. There are two types:

Dense

* Every single parameter in the model is used for every single question you ask  
* It's the traditional, straightforward design  
* Like a single expert who has to answer every question themselves, no matter the topic

MoE (Mixture of Experts)

* The model is divided into many specialized "expert" sub-networks  
* For any given question, only a small handful of the relevant experts activate and do the work  
* The rest sit idle  
* Like a large office where you route each question to whichever specialist is most relevant — the accountant handles finance, the lawyer handles legal, etc.  
* The clever part: MoE models can have a huge total parameter count but only use a fraction at a time, making them faster and cheaper to run than their size suggests

**Tags**

Simple yes/no flags that label notable characteristics of a model:

Reasoning

* Marks models specifically built to think through problems step by step before answering  
* These models deliberately slow down and "show their work" rather than blurting out an immediate answer  
* Examples: OpenAI's o1/o3, DeepSeek-R1

  SOTA (State of the Art)

* Marks whether the model was the best in the world at the time it launched  
* SOTA is a snapshot — a model tagged SOTA gets dethroned quickly as newer models arrive  
* It's a historical badge of honor, not a permanent title

Diffusion (in the context of language models)

Most standard language models work like this: they write text one word at a time, left to right, never going back — like a person typing without a backspace key.

  Diffusion models work completely differently:

* They start with a blob of random noise (think: scrambled gibberish)  
* Then they gradually refine the whole response at once, removing the noise step by step until a coherent answer emerges  
* It's an iterative polishing process rather than a one-direction word-by-word march

*This approach was already well-established in image generation (Stable Diffusion, DALL-E, Midjourney all work this way) and is now being explored for text generation too.*

| Column | Analogy |
| :---- | :---- |
| Parameters (B) | The size of a person's brain |
| Tokens Trained (B) | How many books they've read    |
| Ratio Tokens:Params | Books read per unit of brain size — are they well-read for their brain size, or did they coast? |
| ALScore | Their overall "smartness score" on a report card — combines brain size and how much they studied into one grade |
| MMLU | A broad standardized test score across 57 subjects (the original SAT) |
| MMLU-Pro | A harder version of the same test, introduced when models started acing the original (the harder SAT) |
| GPQA | A PhD-level exam so hard even Google can't help you cheat |
| HLE | The final boss exam — designed by thousands of world experts to be nearly impossible |
| Training Dataset | What the model ate for breakfast — the raw material it learned from |
| Arch | How the brain is wired — one generalist (Dense) vs. a team of specialists (MoE) |
| Tags | Reasoning: Does it stop and think before answering, or just blurt something out? SOTA: Was it the world's best model on the day it launched? Diffusion: Instead of writing an essay word by word, you scribble a messy draft of the whole thing at once and keep erasing and refining until it looks right. |

