# Agenda:
1. **Repeated Switches (Serena)**
2. **Insider Trading (Dylan)**
3. **Long Function (Valerie)**

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


---

# Code Smell #3: Long Function (Valerie)

## What is Long Function?

Long Function is a code smell when a function does too many things at once. Thus makes the code harder to understand and change because everything is in one place. Martin Fowler explains that even though longer functions can work, smaller functions with more descriptive names are easier to understand. One way to fix this is through the menthod of Extract Function, when you take a section of code and move it into its own function.

## My Example

My example is from a Unity game I worked on. The script handles an armor pickup that gives the player additional armor when they collide with it. It works by checking if the player is touching the pickup and checking how much armor the player already has and then calculating how much armor to give them without exceeding their maximum. This is the original `OnCollisionEnter2D()` function from my script:

```C#
void OnCollisionEnter2D(Collision2D collision)
{
    if (collision.gameObject.CompareTag("Player"))
    {
        //print("Armor picked up by Player");
        if (currentArmor < maxArmor)
        {
            int newArmor = addedArmor + currentArmor;
            if (newArmor < maxArmor)
            {
                playerStats.SetArmor(newArmor);
            }
            else
            {
                playerStats.SetArmor(maxArmor);
            }
        }

        Destroy(gameObject);
    }
}
```

## Why This Is a Problem

The main issue is that `OnCollisionEnter2D()` is handling several different responsibilities. It detects the collision, checks whether the object is the player, calculates the new armor value, makes sure the player doesn't exceed their maximum armor, and destroys the pickup. Even though this function isn't extremely long. Also the nested if statements make it more complicated than it needs to be. For example, if I wanted to change how armor pickups worked in the future, i would have to go searching around. This can become a huge problem as more features get added.

## My Solution

I would use the Extract Function to separate the armor calculation from the collision detection. I would also get rid of my nested conditionals using Mathf.Min() to make sure the player doesn't exceed max armour.

First, I would extract the armor calculation into its own function:

```C#
void GiveArmor()
{
    if (currentArmor < maxArmor)
    {
        int newArmor = Mathf.Min(
            currentArmor + addedArmor,
            maxArmor
        );

        playerStats.SetArmor(newArmor);
    }
}
```

This function handles calculating and updating the player's armor without using the collision detection.
Next, I would simplify the original collision function:

```C#
void OnCollisionEnter2D(Collision2D collision)
{
    if (collision.gameObject.CompareTag("Player"))
    {
        GiveArmor();
        Destroy(gameObject);
    }
}
```

Now the collision function is much easier to read it has less extra code and is more defined

## Refactored Code

Heres the script after refactoring:

```C#
using UnityEngine;

public class ArmorPickup : MonoBehaviour
{
    PlayerStats playerStats;

    public int addedArmor = 25;
    public int maxArmor;
    public int currentArmor;
    public float armorDmgReduct;

    void Start()
    {
        playerStats = FindObjectOfType<PlayerStats>();
    }

    void OnCollisionEnter2D(Collision2D collision)
    {
        if (collision.gameObject.CompareTag("Player"))
        {
            GiveArmor();
            Destroy(gameObject);
        }
    }

    void GiveArmor()
    {
        if (currentArmor < maxArmor)
        {
            int newArmor = Mathf.Min(
                currentArmor + addedArmor,
                maxArmor
            );

            playerStats.SetArmor(newArmor);
        }
    }

    void Update()
    {
        maxArmor = playerStats.GetMaxArmor();
        currentArmor = playerStats.GetArmor();
    }
}
```

## Why This Is Better

The biggest advantage is that the two functions now have separate responsibilities. `OnCollisionEnter2D()` handles detecting the player and destroying the pickup, while `GiveArmor()` handles calculating and updating the player's armor. This makes the code easier to read because I can understand what each function does just by looking at its name. It also makes future changes easier like if I wanted to change how armor is calculated or add different armor pickup amounts. I could modify `GiveArmor()` without changing how the collisions work and doing this made the code less cluttered.

## Testing the Refactoring

I tested these situations:

| Current Armor | Max Armor | Armor Pickup | Expected Result |
|---|---|---|---|
| 25 | 100 | 25 | 50 |
| 90 | 100 | 25 | 100 |
| 100 | 100 | 25 | 100 |

I would also test that the pickup is destroyed after the player collects it, even when their armor is already full.I would check that collisions with objects that aren't tagged as the player don't give armor or destroy the pickup. These tests make sure the refactored code matches the orginals functionallity.

## Final Thoughts

I originally wrote everything inside `OnCollisionEnter2D()` because it was easier to keep all the pickup logic in one place.Later on having too much logic inside one function can make it harder for me to modify the code. Using the method Extract Function to break the code into smaller sections that are easier to read and maintain without changing the way the armor pickup works. I think its a good example of how even a short function can improve from refactoring when it handles multiple tasks.

## References

- Martin Fowler, *Refactoring: Improving the Design of Existing Code*, 2nd Edition, Chapter 3: "Bad Smells in Code" — Long Function.
- Martin Fowler, *Refactoring*, Chapter 6: "Extract Function."
- [Refactoring.com — Extract Function](https://refactoring.com/catalog/extractFunction.html)
