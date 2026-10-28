# Agenda:
1. **Repeated Switches (Serena)**
2. **Insider Trading (Dylan)**
3. **Insider Trading (Valerie)**

# Code Smell #1: Repeated Switches
This smell permeates when there is a large chain of If/Else or Switch statements in a given block of code. It's bad for a couple of reasons: One, it's a pain to maintain as everytime you want to add some new variant, you have to find every instance of this giant conditional and add a new clause; This just adds to the seccond issue which is that it makes the code really hard to read because it clutters the function body with every possible variant and hides some of the more important parts of the function.
<br>
This smell is probably quite common among beginners because the logic behind writing it like this is quite easy to understand, but it is especially bad because it desemates the readability of the code.
<br><br>
The solution is usually to ***Replace Conditional With Polymorphism*** (Pg 272 | PDF 294) where you use a Command like structure to place each variant case into its own object with an execute method that has the same name among all versions so that the calling function is more simple.

### My Example
This code is from a game that I was making in Python a couple years ago. I was using the project as a way to practice programming outside of class. At the time, I knew that this formatting of the function was bad, but I didn't know how to fix it and wasn't able to come up with a solution before I dropped the project.

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
---
This code is pasted straight from the file complete with spelling mistakes and weird writing quirks. Please forgive this things, i've change from when I wrote this.
<br>
In this method I renamed the `self` attribute to `user` for some strange reason.

---
### My Solution
This function does a lot of things wrong, but it's mainly concentrated in that conditional chain issue. What's happening is it's being told what item is being used through the `item` parameter and it's checking what logic it should run by checking the item's name. To refactor this I would first create a base class for all items that has a simple `Use()` function that can be reimplemented by the subclasses.

```Python
    class Item:
        __init__(self, name) {
            self.name = name;
        }

        def Use(self) {
            print(f"{self.name} was used!")
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

            self.status = status
            self.strength = statusStrength
        
        def Use(self):
            user = PickTarget_Allies()
            user.inflict(self.status, self.strength)

    class RecoveryPotion(Item):
        def __init__(self, name, recoveryFuncName, recoveryAmount):
            super().__init__(name)
            self._funcName = recoveryFunc
            self.strength = recoveryAmount

        # Thanks: https://www.danielmorell.com/blog/dynamically-calling-functions-in-python-safely
        def Use(self):
            user = PickTarget_Allies()
            func = getattr(user,self._funcName)
            func(self.strength)

    class Bomb(Item):
        def __init__(self, name, damageAmount):
            super().__init__(name)
            self.strength = damageAmount

        def Use(self):
            targets = PickTargets_Enemies()
            for target in targets:
                Attack(target, self.strength)
    
    class SmokeBomb(Item):
        def __init__(self, name, strength):
            super().__init__(name)
            self.stength = strength
        
        def Use(self):
            CreateFieldEffect(SMOKE_EFFECT_NAME, self.strength)
```

Now with these classes I can simplify my original function a little. First, can type the `item` parameter to be the `Item` class that way I don't need the explicit pre-condition.

```Python
    def UseItem(self, item: Item):
        say(f"{item.name} was used!")
        item.Use()
```
Now we turned that 45 line long If/Else chain into 5 classes and a one line long function which is much nicer to read.
<br>
This isn't a perfect solution as these objects still use some global functions and values that could be locally scoped & generalized, but this is one step closer to better code.

---
*Given how short it became, I could even consider removing the `UseItem()` function all together and just putting this functionality straight into the code that is handling item picking, but this might make the code more convoluted and introduce other problems such as **Long Function** (Pg 73 | PDF 95)*

---

#### Bonus:
If I wanted to preserve the parameter requests for each items `Use()` method, I could replace the the check for `if (item.itemType == ___):` to `if (isinstance(item, ___)):` in `UseItem()`.

```Python
    class Potion(Item):
        def __init__(name, status, statusStrength): 
            super().__init__(name)

            self.status = status
            self.strength = statusStrength
        
        def Use(user: Creature):
            user.inflict(self.status, self.strength)

    class RecoveryPotion(Item):
        def __init__(name, recoveryFuncName:, recoveryAmount):
            super().__init__(name)
            self._funcName = recoveryFunc
            self.strength = recoveryAmount

        def Use(user: Creature):
            func = getattr(user,self._funcName)
            func(self.strength)

    class Bomb(Item):
        def __init__(name, damageAmount):
            super().__init__(name)
            self.strength = damageAmount

        def Use(targets: Creature[]):
            for target in targets:
                Attack(target, self.strength)
    
    class SmokeBomb(Item):
        def __init__(name, strength):
            super().__init__(name)
            self.stength = strength
        
        def Use():
            CreateFieldEffect(SMOKE_EFFECT_NAME, self.strength)
```
```Python
    def UseItem(self, item: Item):
        say(f"{item.name} was used!")
        if (isinstance(item, Potion)):
            item.Use(self)
        else if (isinstance(item, Bomb)):
            item.Use(PickTargets_Enemies())
        else:
            item.Use()
```
This is still easier to read than it was before, but it's still dangerous because it can grow into another Repeating Switches problem. It would be like if you pulled out the weeds, but replanted a seed in it's place.

# Code Smell #2: Insider Trading  
_Dylan McDermott_  
  
Sometimes, especially when editing multiple files of code, you want to get information from one place to another, nice and easy. So you create some backdoors, maybe a few quiet references, and use those to get the info a function or class needs. But doing so comes at a cost of organization; if all of your scripts are tied together with red string, is there really much benefit to using separate files? In any case, if you want a stronger codebase, you need to separate contact between these scripts, or at the very least, make it visible to the user. Here's a simple Typescript example:

 ``` Typescript 
let currentBeepNumber = 0;  
  
function beep() {  
  currentBeepNumber++;  
}  
  
function boop() {  
  currentBeepNumber--;  
}  
  ```

In this example, beep and boop are directly editing a value without checking with anyone. While this can be okay in small scenarios such as this, it is also something that could become very messy if we want to add more values or decisions in the future. So, lets remove the direct coupling of the value of currentBeepNumber from beep and boop:

 ``` Typescript 
let currentBeepNumber = 0;  
  
function beep(beepNumber: number) : number {  
  return beepNumber++;  
}  
  
function boop(boopNumber: number) : number {  
  return boopNumber--;  
}  
```

With these changes in place, a programmer now has to be much more direct and clear when it comes to changing the value of currentBeepNumber. instead of simply calling "beep();", the developer would probably write something closer to, "currentBeepNumber = beep(currentBeepNumber);". While this does use more code, it illuminates the action taking place, while also doubling as a way to make this code base less hard-coded.

...  

For a second, slightly more real example, here's a snippet of code from a project I'm currently working on (language is c#, being used in the Unity Game Engine):

``` C#
InputAction camInput;  
InputAction objectRotateInput;  
objectGrabber objGrab;  

// Start is called once before the first execution of Update after the MonoBehaviour is created  
void Start()  
{  
    camInput = InputSystem.actions.FindAction("Look");  
    objectRotateInput = InputSystem.actions.FindAction("Rotate Object");  
    objGrab = this.GetComponent<objectGrabber>();  
}  
```

Here, we see what I would consider an example of **Insider Trading**. This script is using getComponent to find the objectGrabber script, with the assumption that both scripts are attached to the same object. While I could spend more time in the future inspecting the need for these scripts to be coupled in this way, I at the very least have a (if somewhat suspect) simple solution that will suffice for the present. Observe:

``` C#
InputAction camInput;  
InputAction objectRotateInput;  
public objectGrabber objGrab;  

// Start is called once before the first execution of Update after the MonoBehaviour is created  
void Start()  
{  
    camInput = InputSystem.actions.FindAction("Look");  
    objectRotateInput = InputSystem.actions.FindAction("Rotate Object");  
}  
```

What I have done is convert the script reference into a public variable. That is, when in the game scene and inspecting the player, I will now see a script reference that needs to be set manually. While this has not resolved the coupling of these two scripts, it has at the very least shown light on it. That way, future developers (such as myself in a few month's time) will understand from a glance that this script requires objectGrabber to function, rather than attempting to use it without realizing the silent communication happening under the hood.
