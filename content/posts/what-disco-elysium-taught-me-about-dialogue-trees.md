+++
title = 'What Disco Elysium Taught Me About Dialogue Trees'
date = 2024-07-03T14:43:24-07:00
draft = true
+++

## Introduction

Recently, I've played the fantastic 2019 video game *Disco Elysium* and I was blown away by the sheer amount of dialogue, choices, dice rolls and consequences. It made me think: "How do you write a dialogue tree like that?".

My immediate first thought was *SCUMM*, or the *Script Creation Utility for Maniac Mansion*, a game engine developed by then Lucasfilm Games for, you guessed it, *Maniac Mansion (1987)*. I remember hearing about it a couple of years ago and how you could use it to create locations, items, and dialogue sequences without writing code.

Years later, here we are, and without looking at *SCUMM*, *Disco Elysium*'s development, or any other approaches, so to not get too many outside influences, I'm going to try to write a dialogue tree from scratch. The idea is to keep the input, the script, as human-readable and as close to something like a movie script as possible.

## The Plan

The plan is simple: come up with a format for the script that let's me print basic dialogue, provide choices that result in forks in the dialogue, and lastly, in the vain of *Disco Elysium*, add some dice rolls to the mix. Each block of dialogue will have an unique identifier, and at least one target identifier for the next block of dialogue. After parsing, I should be left with a dictionary of all dialogue blocks and their respective IDs. 🤞

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
-> 103
```

Above I wrote what I call two **Passthrough** blocks. Their purpose is just to break up continues dialgue, without the option for choices and forks.

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

Here is some basic code that creates a Dialogue object and reads the script file. We load in our `passthrough.txt` file and split it by double newlines, or better, by blocks. For now we return right away so we can worry about the first block only right now. We also have a `run` method that we will use to start the dialogue. But let's not get ahead of ourselves.

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

We split the block by newlines, store the first item in `identifier`, and the last item in `target`. The lines in between are joined together to form the `content`. If we print out our values we should see the following:

```bash
> python main.py
Identifier: 101, Target: 102

Content:
"You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You walk up to them."
```

## What data structure is right for this?

Now we need to store this information in a way that we can easily access it. At first glance, a dictionary, or better, a `TypedDict` seems like a good choice. But for this example, I will use a Dataclass. The reason for that is that in the run phase, we want to check the type of the dialogue block to handle passthrough, choices, and dice rolls differently.

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

We create a `Block` dataclass that stores the content and the targets. Note how the target is a dictionary with `"next"` as the key and the target as the value. This is because we can have multiple targets, for example, when we have choices. We also create a `Passthrough` class that inherits from `Block` and only has one target. We will add more classes later on.

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
> python main.py
{'101': Passthrough(content='You walk into a tavern. In the corner, you see a '
                            'hooded figure sitting alone. \n'
                            'You walk up to them.',
                    target={'next': '102'})}
```

Great, we have our first block stored in our dictionary. Now we need to implement the `run` method to start the dialogue.

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

Like all recursive functions, we need to be incredibly mindful of our base case.

Our first base case is returning out of the function when we encounter the string "EXIT". We will use "EXIT" for all dialogue branches that end the conversation.

 Our second "base case" is throwing an error if we encounter an unknown key. This will be useful for when we try to jump to a block that we [haven't implemented yet](https://en.wikipedia.org/wiki/Foreshadowing).

 We also added a `start` argument to the `run` method to get the conversation started and a seemingly useless input function to pause the dialogue until the player presses enter. We also throw an error if we encounter an unknown block type.

Let's run this and check out the output.

```bash
> python main.py
You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.

> Press Enter to continue... 

Traceback (most recent call last):
  File "/Users/lennartbreede/Code/dialog-tree/main.py", line 56, in <module>
    main()
  File "/Users/lennartbreede/Code/dialog-tree/main.py", line 52, in main
    dialogue.run("101")
  File "/Users/lennartbreede/Code/dialog-tree/main.py", line 44, in run
    self.run(block.target["next"])
  File "/Users/lennartbreede/Code/dialog-tree/main.py", line 37, in run
    raise ValueError(f"Block {start!r} not found")
ValueError: Block '102' not found
```

The beginning looks promising! The content gets printed and after pressing enter, the next block is loaded, or at least it tries to. Currently we are getting an error because, remember, we put a `return` in the `load_script` method. Let's remove that and try again.

```bash
> python main.py
You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.

> Press Enter to continue... 

    **Potion Seller**
Hello there traveler. What can I do for you?
I have the strongest potions in the land.

> Press Enter to continue... 

Goodbye!
```

Fantastic. We have a basic dialogue system that can handle passthrough blocks. In the next part, we will add choices and dice rolls to the mix. Stay tuned! 🎲

## Choices

Let's update our script to include a choice block.

```plaintext
choices.txt

101:
You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.
-> 102

102:CHOICE
    **Potion Seller**
Hello there traveler. What can I do for you?
I have the strongest potions in the land.
1. Ask about the potions. -> 103
2. Ask for directions. -> 104
3. Leave. -> EXIT

103:
    **Potion Seller**
I have potions that can heal any wound, cure any disease, and even bring the dead back to life.
-> 102

104:
    **Potion Seller**
The forest to the east is dangerous. I would avoid it if I were you.
-> 102

```

The new script now includes a choice block. The three choices are prefixed with their respective number and a period. Two of the choices lead back to the potion seller, while the third choice ends the conversation.

```python
    def load_script(self, file: str) -> None:
        with open(file, "r", encoding="utf-8") as fp:
            for block in fp.read().split("\n\n"):
                identifier, *lines = block.split("\n")
                identifier, type_ = [x.strip() for x in identifier.split(":")]
                identifier = identifier.strip()
                type_ = type_.strip()
                if type_ == "CHOICE":
                    pass
                elif type_ == "DICEROLL":
                    pass
                else:
                    target = lines.pop()
                    content = "\n".join(lines)
                    target = target.removeprefix("->").strip()
                    block = Passthrough(content, target)
                    self.dialog_dict[identifier] = block
```

The parts that a passthrough-specific are now in an `else` block. We check the type of the block and if it's a choice block, we will handle it later. If it's a dice roll block, we will also handle it later.

Let's start with choice.

```python
# main.py

class Choice(Block):
    pass

class Dialogue:

    # --snip--

    def load_script(self, file: str) -> None:

                # --snip--

                if type_ == "CHOICE":
                    n = int(lines[-1].split(".")[0])
                    choices = lines[-n:]
                    lines = lines[:-n]
                    targets: dict[str, str] = {}
                    for i, choice in enumerate(choices):
                        *choice, target = [x.strip() for x in choice.split("->")]
                        choice = "->".join(choice)
                        targets[str(i + 1)] = target
                        lines.append(choice)

                    content = "\n".join(lines)
                    block = Choice(content, targets)
                    self.dialog_dict[identifier] = block
```

There are a million different ways to separate the choices from the content and extract the targets. This is just one way. First, I access the last line, split it by a period and keep the first element. In this example, this should result in the string "3", since our final line starts is "3. Leave.". With this, we have to assume that A. the last line is always the number of choices and B. that the choices are labeled 1. to n. without any gaps in the sequence. Once we have that, we can slice the choices from the rest of the lines and get parsing. We loop over our choices and separate the text from the target IDs by splitting at the arrow. For completeness, we join the text using the same error, in the offchance that the text contains an arrow (There's probably some whitespace issue now since I already stripped before joining but hey, that's your homework). We then use the index of the enumeratiion as the key for the target dictionary and the target as its value. We also append the choices back onto the content since we have chopped off the target IDs. Lastly, we create an instance of our `Choice` class and add it to our `dialog_dict`.

Phew. That was a lot. Good thing all this hard work will make the `run` method a breeze.

```python
# main.py

    def run(self, start: str) -> None:
        if start == "EXIT":
            print("Goodbye!")
            return

        if start not in self.dialog_dict:
            raise ValueError(f"Block {start!r} not found")

        block = self.dialog_dict[start]

        if isinstance(block, Passthrough):
            print(block.content)
            _ = input("\n> Press Enter to continue... ")
            print()
            self.run(block.targets["next"])
        elif isinstance(block, Choice):
            print(block.content)
            choice = input("\n> Choose: ")
            print()
            self.run(block.targets[choice])
        else:
            raise ValueError(f"Unknown block type {block!r}")
```

As you can see, all we do is take the input from the player and use it as the key to get the target.

Easy peasy!

```bash
python main.py
You walk into a tavern. In the corner, you see a hooded figure sitting alone. 
You approach the figure and sit down across from them.

> Press Enter to continue... 

    **Potion Seller**
Hello there traveler. What can I do for you?
I have the strongest potions in the land.
1. Ask about the potions.
2. Ask for directions.
3. Leave.

> Choose: 1

    **Potion Seller**
I have potions that can heal any wound, cure any disease, and even bring the dead back to life.

> Press Enter to continue... 

    **Potion Seller**
Hello there traveler. What can I do for you?
I have the strongest potions in the land.
1. Ask about the potions.
2. Ask for directions.
3. Leave.

> Choose: 2

    **Potion Seller**
The forest to the east is dangerous. I would avoid it if I were you.

> Press Enter to continue... 

    **Potion Seller**
Hello there traveler. What can I do for you?
I have the strongest potions in the land.
1. Ask about the potions.
2. Ask for directions.
3. Leave.

> Choose: 3

Goodbye!
```

We set ourself up for success and are now reapin' the rewards. In the next part, we will add dice rolls to the mix. Stay tuned! 🎲