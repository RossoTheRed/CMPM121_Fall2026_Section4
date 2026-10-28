## Group:
- Serena
- Dylan
- Valerie

## Smells:
1. Repeated Switches
2. Duplicated Code
3. Insider Trading

## Code Smell: Repeated Switches
This smell permeates when there is a large chain of If/Else or Switch statements in a given block of code. It's bad for a couple of reasons: One, it's a pain to maintain as everytime you want to add some new variant, you have to find every instance of this giant conditional and add a new clause; This just adds to the seccond issue which is that it makes the code really hard to read because it clutters the function body with every possible variant and hides some of the more important parts of the function.
<br>
This smell is probably quite common among beginners because the logic behind writing it like this is quite easy to understand, but it is especially bad because it desemates the readability of the code.
<br><br>
The solution is usually to *Replace Conditional With Polymorphism* (Pg 272 | PDF 294) where you use a Command like structure to place each variant case into its own object with an execute method that has the same name among all versions so that the calling function is more simple.

### My Example
This code is from a game that I made in Python a couple years ago. I was using the project as a way to practice programming outside of class. At the time, I knew that this formatting of the function was bad, but I didn't know how to fix it and wasn't able to come up with a solution before I dropped the project.

    ```Python
    def useItem(user,item,target = None):
    if isinstance(item,ItemClass):
        if item.itemType == 'potion':
            say(f"{user} drinks a {item.name}")
            if item.name == "Health Restoration Potion":
                healing = (user.maxHp//4)*item.level
                user.heal(healing)
                if user is player or player.stats['int'] >= 7:
                    say(f"{user.name} recovered some health")
            elif item.name == "Mana Restoration Potion":
                restoration = (user.stats['int']//4)*item.level
                user.mp += restoration
                if user is player or player.stats['int'] >= 7:
                    say(f"{user.name} recovered some mana")
            elif item.name == "Slick Skin Potion":
                user.inflict('evasion boost L',item.level)
                if user is player or player.stats['int'] >= 7:
                    say(f"{user.name}'s evasion increased!")
            elif item.name == "Thick Skin Potion":
                user.inflict('defense boost L',item.level)
                if user is player or player.stats['int'] >= 7:
                    say(f"{user.name}'s defense increased!")
            elif item.name == "Fierce Fist Potion":
                user.inflict('attack boost L',item.level)
                if user is player or player.stats['int'] >= 7:
                    say(f"{user.name}'s attack increased!")
        elif item.itemType == 'throw':
            projectile = WeaponClass(item.name,random.randint(item.level,item.level+1),user.stats['per'],0,item.level)
            if item.name == 'Rock':
                say(f"{user.name} throws a {item.name} at {target.name}",'')
                tell('...',1)
                attackRanged(target,user,projectile)
            elif item.name == 'Bomb':
                if phase == 'battle':
                    say(f"{user.name} throws a bomb!")
                    attackAll(target,player,projectile)
                elif phase == 'explore':
                    say("You light the fuse and step back")
                    tell('...',1)
                    gameMap[floorLevel].destroyWalls(player.mapPos)
        elif item.itemType == 'field':
            say(f"{user.name} uses {item.name}")
            if item.name == 'Smoke Bomb':
                if fieldConditions.get('smoky') == None: 
                    fieldConditions['smoky'] = 3*item.level
                else:
                    fieldConditions['smoky'] += 3*item.level
    ```
### My Solution
This function does a lot of things wrong, but it's mainly concentrated in that conditional chain issue. What's happening is it's being told what item is being used through the `item` parameter and it's checking what logic it should run by checking the item's name. To refactor this I would first create a base class for all items that has a simple `Use()` function that can be reimplemented by the subclasses.

```Python
    class Item:
        __init__(self, name) {
            this.name = name;
        }

    def Use(self) {
        print(f"{this.name} was used!")
        raise RuntimeError("This method should be reimplimented by a subclass not used as is.")
    }

```
---
In this example making the base class could be considered Speculative generality since it's never used for anything more than just setting the items name and warning about failed implimentation. However, given that the scenario is making a whole RPG, I know that an item will need more methods than just this `Use()` later in the development; I just kept it simple for the sake of example.

---

Next I would make subclasses for these types of items. There are two types of items being used in this function that I'll need to extract: consumable potions and throwable items. It'll be easy for the potions since all they do is put a status on the character and so the constructor can take a reference to the stat to boost and an amount to boost it by. Since the throwables have more unique effects I'll have to make a separate class for each of them. Since some of the items have different requirements for their effects, we can make their invocation gather those values during runtime in order to keep the `UseItem()` function simple.

```Python
    class Potion(Item):
        def __init__(self, name, status, statusStrength): 
            super().__init__(name)

            this.status = status
            this.strength = statusStrength
        
        def Use(self):
            user = PickTarget_Allies()
            user.inflict(this.status, this.strength)

    class RecoveryPotion(Item):
        def __init__(self, name, recoveryFuncName, recoveryAmount):
            super().__init__(name)
            this._funcName = recoveryFunc
            this.strength = recoveryAmount

        # https://www.danielmorell.com/blog/dynamically-calling-functions-in-python-safely
        def Use(self):
            user = PickTarget_Allies()
            func = getattr(user,self._funcName)
            func(this.strength)

    class Bomb(Item):
        def __init__(self, name, damageAmount):
            super().__init__(name)
            this.strength = damageAmount

        def Use(self):
            targets = PickTargets_Enemies()
            for target in targets:
                Attack(target, this.strength)
    
    class SmokeBomb(Item):
        def __init__(self, name, strength):
            super().__init__(name)
            this.stength = strength
        
        def Use(self):
            CreateFieldEffect(SMOKE_EFFECT_NAME, this.strength)
```

Now with these classes I can simplify my original function a little. First, can type the `item` parameter to be the `Item` class that way I don't need the explicit pre-condition.

```Python
    def UseItem(item: Item):
        say(f"{item.name} was used!")
        item.Use()
```
Now we turned that 45 line long If/Else chain into 5 classes and a one line long function which is much nicer to read.
<br>
This isn't a perfect solution as these objects still use some global functions and values that could probably locally scoped, but this is one step closer to better code.

#### Bonus:
If I wanted to preserve the parameter requests for each items `Use()` method, I could replace the the check for `if (item.itemType == ___):` to `if (isinstance(item, ___)):` in `UseItem()`.
<!-- Now, since the various items do still have different parameters for their `Use()` methods, I'll have to keep some of the conditional logic, but it will be much more simple. I can change the check for `if (item.itemType == ___):` to `if (isinstance(item, ___)):` and handle the `Use()` method. -->

```Python
    class Potion(Item):
        def __init__(name, status, statusStrength): 
            super().__init__(name)

            this.status = status
            this.strength = statusStrength
        
        def Use(user: Creature):
            user.inflict(this.status, this.strength)

    class RecoveryPotion(Item):
        def __init__(name, recoveryFunc, recoveryAmount):
            super().__init__(name)
            this._func = recoveryFunc
            this.strength = recoveryAmount

        def Use(user: Creature):
            this._func(this.strength)

    class Bomb(Item):
        def __init__(name, damageAmount):
            super().__init__(name)
            this.strength = damageAmount

        def Use(targets: Creature[]):
            for target in targets:
                Attack(target, this.strength)
    
    class SmokeBomb(Item):
        def __init__(name, strength):
            super().__init__(name)
            this.stength = strength
        
        def Use():
            CreateFieldEffect(SMOKE_EFFECT_NAME, this.strength)
```
```Python
    def UseItem(item: Item):
        say(f"{item.name} was used!")
        if (isinstance(item, Potion)):
            item.Use(global player)
        else if (isinstance(item, Bomb)):
            item.Use(PickTargets_Enemies())
        else:
            item.Use()
```
This is still easier to read than it was before, but it's still dangerous because it can grow into another Repeating Switches problem. It would be like if you pulled out the weeds, but replanted a seed in it's place.