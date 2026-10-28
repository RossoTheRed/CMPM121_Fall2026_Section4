## Group:
- Serena
- Dylan
- Valerie

## Smells:
1. Long Parameter list
2. Duplicated Code
3. Insider Trading

## Insider Trading  
_Dylan McDermott_  
  
Sometimes, especially when editing multiple files of code, you want to get information from one place to another, nice and easy. So you create some backdoors, maybe a few quiet references, and use those to get the info a function or class needs. But doing so comes at a cost of organization; if all of your scripts are tied together with red string, is there really much benefit to using separate files? In any case, if you want a stronger codebase, you need to separate contact between these scripts, or at the very least, make it visible to the user. Here's a simple Typescript example:

/////////////////////////////////////////////////  
  
let currentBeepNumber = 0;  
  
function beep() {  
  currentBeepNumber++;  
}  
  
function boop() {  
  currentBeepNumber--;  
}  
  
/////////////////////////////////////////////////  

In this example, beep and boop are directly editing a value without checking with anyone. While this can be okay in small scenarios such as this, it is also something that could become very messy if we want to add more values or decisions in the future. So, lets remove the direct coupling of the value of currentBeepNumber from beep and boop:

/////////////////////////////////////////////////  
  
let currentBeepNumber = 0;  
  
function beep(beepNumber: number) : number {  
  return beepNumber++;  
}  
  
function boop(boopNumber: number) : number {  
  return boopNumber--;  
}  
  
///////////////////////////////////////////////// 

With these changes in place, a programmer now has to be much more direct and clear when it comes to changing the value of currentBeepNumber. instead of simply calling "beep();", the developer would probably write something closer to, "currentBeepNumber = beep(currentBeepNumber);". While this does use more code, it illuminates the action taking place, while also doubling as a way to make this code base less hard-coded.

...  

For a second, slightly more real example, here's a snippet of code from a project I'm currently working on (language is c#, being used in the Unity Game Engine):

/////////////////////////////////////////////////  

InputAction camInput;  
InputAction objectRotateInput;  
**objectGrabber objGrab;**  

// Start is called once before the first execution of Update after the MonoBehaviour is created  
void Start()  
{  
    camInput = InputSystem.actions.FindAction("Look");  
    objectRotateInput = InputSystem.actions.FindAction("Rotate Object");  
    **objGrab = this.GetComponent<objectGrabber>();**  
}  

/////////////////////////////////////////////////  

Here, we see what I would consider an example of **Insider Trading**. This script is using getComponent to find the objectGrabber script, with the assumption that both scripts are attached to the same object. While I could spend more time in the future inspecting the need for these scripts to be coupled in this way, I at the very least have a (if somewhat suspect) simple solution that will suffice for the present. Observe:

/////////////////////////////////////////////////  

InputAction camInput;  
InputAction objectRotateInput;  
**<ins>public</ins> objectGrabber objGrab;**  

// Start is called once before the first execution of Update after the MonoBehaviour is created  
void Start()  
{  
    camInput = InputSystem.actions.FindAction("Look");  
    objectRotateInput = InputSystem.actions.FindAction("Rotate Object");  
}  

/////////////////////////////////////////////////  

What I have done is convert the script reference into a public variable. That is, when in the game scene and inspecting the player, I will now see a script reference that needs to be set manually. While this has not resolved the coupling of these two scripts, it has at the very least shown light on it. That way, future developers (such as myself in a few month's time) will understand from a glance that this script requires objectGrabber to function, rather than attempting to use it without realizing the silent communication happening under the hood.
