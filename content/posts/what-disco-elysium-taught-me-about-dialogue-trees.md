+++
title = 'What Disco Elysium Taught Me About Dialogue Trees'
date = 2024-07-03T14:43:24-07:00
draft = true
tags = ['python', 'gamedev']
+++

## Introduction

Recently, I played the fantastic 2019 video game *Disco Elysium*, and I was blown away by the sheer amount of dialogue, choices, dice rolls, and consequences. It made me think, "How do you write a dialogue tree like that?"

![](disco_elysium_screenshot.jpg)

My immediate first thought was [SCUMM](https://en.wikipedia.org/wiki/SCUMM), or the *Script Creation Utility for Maniac Mansion*, a game engine developed by then Lucasfilm Games for, you guessed it, *Maniac Mansion (1987)*. I remember hearing about it a few years ago and how you could create locations, items, and dialogue sequences without writing code.

Years later, here we are, and without looking at *SCUMM*, *Disco Elysium*'s development, or any other approaches to not get too many outside influences, I will write a dialogue tree from scratch. The idea is to keep the input, aka the script, as human-readable and as close to something like a movie script as possible.

## The Plan

The plan is simple: develop a format for the script that lets me print basic dialogue, provide choices that result in forks in the dialogue, and lastly, in the vain of *Disco Elysium*, add some dice rolls to the mix. Each block of dialogue will have a unique identifier and at least one target identifier for the next block of dialogue. After parsing, I should have a dictionary of all dialogue blocks and their respective IDs. 🤞

## Passthrough Dialogue

```plaintext
passthrough.txt

101:
You walk into a tavern. In the corner, you see a hooded figure sitting alone. You walk up to them.
-> 102

102:
    **Potion Seller**
Hello there traveler. What can I do for you?
I have the strongest potions in the land.
-> EXIT
```

Above, I wrote what I call two Passthrough blocks. Their purpose is to break up continuous dialogue without the option for choices and forks.

```python
# main.py

class Dialogue:
    def __init__(self) -> None:
        self.dialog_dict: dict[str, str] = {}

    def load_script(self, file: str) -> None:
        with open(file, "r", encoding="utf-8") as fp:
            for block in fp.read().split("\n\n"):
                print(block)
                return

    def run(self, start: str) -> None:
        pass


def main() -> None:
    dialogue = Dialogue()
    dialogue.load_script("passthrough.txt")
    dialogue.run()
```

Some basic code creates a Dialogue object and reads the script file. We load in our `passthrough.txt` file and split it by double newlines, or better, by blocks. We return immediately, so we can only worry about the first block for now. We also have a `run` method that we will use to start the dialogue. But let's not get ahead of ourselves.

We modify the `load_script` method to extract the relevant information from the block.

```python
# main.py

    def load_script(self, file: str) -> None:
        with open(file, "r", encoding="utf-8") as fp:
            for block in fp.read().split("\n\n"):
                identifier, *lines, target = block.split("\n")
                identifier, *_ = identifier.split(":")
                content = "\n".join(lines)
                target = target.removeprefix("->").strip()
                print(f"Identifier: {identifier}, Target: {target}\n")
                print(f'Content:\n"{content}"')
                return
```

We split the block by newlines and store the first item as the *identifier* and the last as the *target*. The lines in between are joined together to form the *content*. If we print out our values, we should see the following:

```bash
$ python main.py
Identifier: 101, Target: 102

Content:
"You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You walk up to them."
```

## What data structure is suitable for this?

Now, we need to store this information to be easily accessed. At first glance, a dictionary, or better, a *TypedDict*, is a good choice. But for this example, I will use a data class. The reason is that in the run phase, we want to check the type of dialogue block to handle passthrough, choices, and dice rolls differently.

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

We create a data class *Block* that stores the content and the targets. Note how the target is a dictionary with "next" as the key and the target as the value. This is because we can have multiple targets, for example, when we have choices. We also create a *Passthrough* class inherited from *Block* with only one target. We will add more classes later on.

```python
# main.py

    def load_script(self, file: str) -> None:

                # --snip--

                block = Passthrough(content, target)
                self.dialog_dict[identifier] = block
                pprint(self.dialog_dict)
                return
```

```bash
$ python main.py
{'101': Passthrough(content='You walk into a tavern. In the corner, you see a '
                            'hooded figure sitting alone. \n'
                            'You walk up to them.',
                    target={'next': '102'})}
```

Great! We have our first block stored in our dictionary. Now, we need to implement the `run` method to start the dialogue.

```python
# main.py

    def run(self, start: str) -> None:
        if start == "EXIT":
            print("Goodbye!")
            return

        if start not in self.dialog_dict:
            raise ValueError(f"Block {start!r} not found")

        block = self.dialog_dict[start]
        print(block.content)
        _ = input("\n> Press Enter to continue... \n")

        if isinstance(block, Passthrough):
            self.run(block.target["next"])
        else:
            raise ValueError(f"Unknown block type {block!r}")


def main() -> None:
    dialogue = Dialogue()
    dialogue.load_script("passthrough.txt")
    dialogue.run("101")
```

Like all recursive functions, we must be incredibly mindful of our base case.

Our first base case is returning out of the function when we encounter the string "EXIT." We will use "EXIT" for all dialogue branches that end the conversation.

 Our second "base case" throws an error if we encounter an unknown key. This will be useful when we [attempt to jump to a block we still need to create](https://en.wikipedia.org/wiki/Foreshadowing).

 We also added a **start** argument to the **run** method to get the conversation started and a seemingly useless input function to pause the dialogue until the player presses enter. We also throw an error if we encounter an unknown block type.

Let's run this and check out the output.

```
$ python main.py
You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.

> Press Enter to continue... 

Traceback (most recent call last):
[...]
ValueError: Block '102' not found
```

The beginning looks promising! The content gets printed, and after pressing enter, the next block is loaded, or at least it tries to. Currently, we are getting an error because, remember, we put a **return** in the **load_script** method. Let's remove that and try again.

```
$ python main.py
You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.

> Press Enter to continue... 

    **Potion Seller**
Hello there traveler. What can I do for you?
I have the strongest potions in the land.

> Press Enter to continue... 

Goodbye!
```

Fantastic. We have a basic dialogue system that can handle passthrough blocks. In the next part, we will add choices and dice rolls. 

Stay tuned! 🎲
***
Part two is coming soon!
[](/posts/adding-choices-and-dice-rolls-to-my-dialogue-tree)