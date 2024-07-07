+++
title = 'What Disco Elysium Taught Me About Dialogue Trees'
date = 2024-07-03T14:43:24-07:00
draft = false
tags = ['python', 'gamedev']
+++

## Introduction

Recently, I started playing the fantastic 2019 video game *Disco Elysium*, and I was blown away by the sheer amount of dialogue, choices, dice rolls, and consequences. It made me think, "How do you write a dialogue tree like that?"

![](disco_elysium_screenshot.jpg)

My immediate first thought was [SCUMM](https://en.wikipedia.org/wiki/SCUMM), or the *Script Creation Utility for Maniac Mansion*, a game engine developed by then Lucasfilm Games for, you guessed it, *Maniac Mansion (1987)*. I remember hearing about it a few years ago and how you could create locations, items, and dialogue sequences without writing code.

Years later, here we are, and without looking at *SCUMM*, *Disco Elysium*'s development, or any other approaches to not get too many outside influences, I will write a dialogue tree from scratch. The idea is to keep the input, aka the script, as human-readable and as close to something like a movie screenplay as possible.

## The Plan

The plan is simple: develop a format for the script that lets me print basic dialogue, provide choices that result in forks in the dialogue, and lastly, in the vain of *Disco Elysium*, add some dice rolls to the mix. Each block of dialogue will have a unique identifier and at least one target identifier for the next block of dialogue. After parsing, I should have a dictionary of all dialogue blocks and their respective IDs. 🤞

## Passthrough Dialogue

```plaintext
simple.txt



1x01
You enter the tavern, *The Iron Tankard*. 
In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.
-> 1x02

1x02
        **Potion Seller**
Hello there, traveller. What can I do for you?
I have the strongest potions in the land.
-> 1x03




1x03
        **Potion Seller**
But my potions are too strong for you traveller!
-> EXIT

```

Above, I wrote three of what I call Passthrough blocks. Their purpose is to break up continuous stories and dialogue without the option for choices and forks. Note a couple of details:
1. The newlines are inconsistent before, in between, and after the blocks. It's always good to have a bit of a scrambled input (within reason) when developing to account for user mistakes and, in this case, writing styles.
2. I chose a mixture of letters and numbers for the block identifier to ensure that both types are handled correctly.
3. I snuck in some markdown syntax that we can parse and style using the [*colorama* Python library](https://pypi.org/project/colorama/). Feel free to make this a markdown file for a better preview.

```python
# main.py

class Dialogue:
    def __init__(self) -> None:
        self.dialogue_dict: dict[str, Block] = {}

    def load_script(self, file: str) -> None:
        with open(file, "r", encoding="utf-8") as fp:
            text = fp.read()

        for block in text.split("\n\n"):
            if not block:
                continue
                
            print(block)
            return

    def run(self) -> None:
        NotImplemented


def main() -> None:
    dialogue = Dialogue()
    dialogue.load_script("simple.txt")
    # dialogue.run()
```

Here is some boilerplate code for creating the *dialogue class* and loading the script. We load in our `simple.txt` file and split it by double newlines, or better, by blocks. Using an if-statement at the beginning of the for-loop, we can account for the inconsistent newlines and keep jumping to the start until we find a valid block. We then return immediately, so we can only worry about the first block for now. We also have a `run()` method that we will use to start the dialogue. But let's not get ahead of ourselves.

Let's modify the `load_script()` method to extract the relevant information from the block.

```python
# main.py

	# --snip--

    def load_script(self, file: str) -> None:
        with open(file, "r", encoding="utf-8") as fp:
            text = fp.read()

        for block in text.split("\n\n"):
            if not block:
                continue

            identifier, content = block.split("\n", 1)
            
            lines = content.split("\n")
            last = lines.pop()
            content = "\n".join(lines)
            target = last.split("->")[-1].strip()
            print(f"Identifier: {identifier!r}, Target: {target!r}\n")
            print(f'Content:\n"{content}"')
```

We split the block by newlines and store the first item as the `identifier` and the second as `content`. We start with this approach since it is shared across all types of dialogue blocks; everything below is specific to the passthrough kind. Ultimately, we have three essential strings: the block identifier, the target identifier, and the block's content.

```bash
$ python main.py
Identifier: '1x01', Target: '1x02'

Content:
"You enter the tavern, *The Iron Tankard*. 
In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them."
```

## What data structure is suitable for this?

Now, we need to store this information to be easily accessed. At first glance, a dictionary, or better, a `TypedDict`, is a good choice. But for this example, I will use a data class. In the run phase, we want to check the type of dialogue block to handle passthrough, choices, and dice rolls differently.

```python
# main.py

@dataclass
class Block:
    content: str
    target: dict[str]

class Passthrough(Block):
    def __init__(self, content: str, target: str) -> None:
        super().__init__(content, {"next": target})
```

We create a data class `Block()` that stores the content and the targets. Note how the target is a dictionary with `"next"` as the key and the target as the value. This is because we can have multiple targets, for example, when we have choices. We also create a *Passthrough* class inherited from *Block* with only one target. We will add more classes later on.

```python
# main.py

    def load_script(self, file: str) -> None:

                # --snip--

                block = Passthrough(content, target)
                self.dialogue_dict[identifier] = block
                pprint(self.dialogue_dict)
                return
```

```bash
$ python main.py
{'1x01': Passthrough(content='You enter the tavern, *The Iron Tankard*. \n'
                             'In the corner, you see a hooded '
                             'figure sitting alone. \n'
                             'You approach the figure '
                             'and sit down across from them.',
                     targets={'next': '1x02'})}
```

Great! We have our first block stored in our dictionary. Now, we need to implement the `run()` method to start the dialogue.

```python
# main.py

    def run(self, start: str) -> None:
        if start == "EXIT":
            print("Goodbye!")
            return

        if start not in self.dialogue_dict:
            raise ValueError(f"Block {start!r} not found")

        block = self.dialogue_dict[start]

        print(block.content)
        _ = input("\n> Press Enter to continue... ")
        print()
        if isinstance(block, Passthrough):
            self.run(block.targets["next"])


def main() -> None:
    dialogue = Dialogue()
    dialogue.load_script("simple.txt")
    dialogue.run("1x01")
```

Like all recursive functions, we must be incredibly mindful of our base case.

Our first base case is returning out of the function when we encounter the string *"EXIT."* We will use *"EXIT"* for all dialogue branches that end the conversation.

 Our second "base case" throws an error if we encounter an unknown key. This will be useful when we [attempt to jump to a block we still need to create](https://en.wikipedia.org/wiki/Foreshadowing).

 We also added a `start` argument to the `run()` method to get the conversation started and a seemingly useless input function to pause the dialogue until the player presses enter. We also throw an error if we encounter an unknown block type.

Let's run this and check out the output.

```
$ python main.py
You enter the tavern, *The Iron Tankard*. 
In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.

> Press Enter to continue... 

Traceback (most recent call last):
[...]
ValueError: Block '1x02' not found
```

The beginning looks promising! The content gets printed, and after pressing enter, the next block is loaded, or at least it tries to. Currently, we are getting an error because, remember, we put a **return** in the `load_script()` method. Let's remove that and try again.

```
$ python main.py
You enter the tavern, *The Iron Tankard*. 
In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.

> Press Enter to continue... 

        **Potion Seller**
Hello there, traveller. What can I do for you?
I have the strongest potions in the land.

> Press Enter to continue... 

        **Potion Seller**
But my potions are too strong for you traveller!

> Press Enter to continue... 

Goodbye!
```

Fantastic. We have a basic dialogue system that can handle passthrough blocks. 
I will quickly create two helper functions that style the words wrapped in markdown syntax.

```python
import re

import colorama
from colorama import Style
from colorama import Fore

colorama.init(autoreset=True)

CHOICE_SEP = "-*-"
BOLD_PATTERN = re.compile(r"\*{2}(.*?)\*{2}")
ITAL_PATTERN = re.compile(r"\*{1}(.*?)\*{1}")

# --snip--

class Dialogue:

	# --snip--
	
    def load_script(self, file: str) -> None:

			# --snip--

            content = self.fmt_bold(content)
            content = self.fmt_italic(content)

            block = Passthrough(content, target)
            self.dialogue_dict[identifier] = block

    @staticmethod
    def fmt_bold(text: str) -> str:
        return re.sub(
            BOLD_PATTERN,
            f"{Style.BRIGHT}{Fore.GREEN}\\1{Style.RESET_ALL}",
            text,
        )

    @staticmethod
    def fmt_italic(text: str) -> str:
        return re.sub(ITAL_PATTERN, f"{Style.DIM}\\1{Style.RESET_ALL}", text)
```
We create two static methods that identify occurrences of words wrapped in asterisks and replace them with *colorama* styles and colours. This approach is flawed because it only works if we run `fmt_bold()` before running `fmt_italic()` since, technically, `**this bold string**` would be matched against the italic patterns. But as long as we know that, there's no need to get into the wild world of negative *RegEx* lookups.

![](formatted_dialogue.png)

Fantastic! In the next part, we will add choices and dice rolls. 

Stay tuned! 🎲
***
Part two is coming soon!
[](/posts/adding-choices-and-dice-rolls-to-my-dialogue-tree)