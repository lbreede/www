+++
title = 'Adding choices and dice rolls to my dialogue tree'
date = 2024-07-04T16:30:00-07:00
draft = true
tags = ['python', 'gamedev']
+++
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

As you can see, we take the player's input and use it as the key to retrieve the target.

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

We set ourselves up for success and are now reapin' the rewards. In the next part, we will add dice rolls to the mix. Stay tuned! 🎲