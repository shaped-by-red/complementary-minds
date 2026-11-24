# 2026-10-10 - "Can you check without breaking it?": privacy where the AI is part of the boundary

**Context.** RED keeps some deeply personal family recordings that must never leave his computer. They're transcribed by a fully offline tool on his own machine. He had forgotten how to run it, and asked Claude to look it up, with one condition: *"Is there a way you can check in our history that does not itself violate the requirement to stay on this computer?"*

**Decision moment 1: the human named the AI as part of the boundary.** Most privacy rules are written around other people and other services. RED's question recognized that an AI assistant is itself an external service: whatever it reads leaves the machine. He didn't assume the helper sat inside the circle of trust. He asked.

**Decision moment 2: honesty about the AI's own reach.** Claude didn't answer "yes, safely." It said what was true: anything it reads goes to its provider. Then it narrowed the work to the least it needed: its own prior note, the few rule lines that mention transcription, and the tool's code, never a recording or transcript. Afterwards it listed exactly what had passed through, including two file names it saw by accident. It also spotted that the tool printed every transcribed line to the screen, which meant running it *through* the AI would leak it. So it told RED to run it himself, and later changed the default so the text isn't shown.

**Decision moment 3: building for content you will never see.** When the folder was reorganized for more family members, Claude tested every change on a made-up sentence spoken by the computer's own voice. It fingerprinted the originals before and after the move, so it could show nothing changed, without ever opening them.

**Why it might matter to a future Mind.** Trust isn't claiming to be safe; it's being exact about where you aren't. When someone protects something precious and asks whether *you* are a risk to it, the respectful answer is a precise account of what you touch, the smallest footprint that does the job, and tools that keep the boundary even when someone forgets.
