# StatsAPI-2.0
Mod for The Binding of Isaac Afterbirth+ whose aim is to provide better support for modders to modify stats - both to better mesh with existing game items, and to better mesh with other mods. If you find any bugs, feel free to send them my way. Most of these bugs will likely be where the synergy of 2 items changes the stat effect of an item where I missed it. I am always around in the Modding of Isaac discord server if you want to just ping me and let me know about anything (name is Nine on that server).

# Design:
The main idea of the mod is that the only thing we can use as modders currently is the output stat after all base game calculations, and that sucks. To fix it, I took some time reverse-engineering how the game generally does stats and categorized them into a few various stages. Then, this mod takes the effort to undo the effect of all the stages to get back down to the base stats before all the formulas and multipliers and cancer, and then slowly rebuild the stats step by step, this time including modded stat items into the actual build. What this means is that you can get a reasonably accurate change change with the base game for stat items, while theoretically leaving most of the base stats, and other modded items, unaffected (Sidenote: This isn't always true though. Because of some floors on stuff like tears specifically, sometimes the output value at lower tears numbers varies when you have certain synergies that affect tear rate. This is mostly because they haven't been hard coded in stat api to fix that offset).

The setup of this mod was designed in such a way that it requires no end-user excess input to get everything working functionally, and to retain a similar usage to how the base game's callback system works with some added functionality to allow better mod integration. There was originally a stats api 1.0, and that had functionally the same features as 2.0, but was designed in a confusing and convoluted way, which prompted me to rewrite it from the ground up. 

The current way it is setup, you simply drag a copy of the statAPI.lua into your mod folder. Then you run a require function, and use the stat api as normal. With the way it is built, any other mods also requiring this same file will check the version number and overwrite various functions with newer versions. This means that stat api will run the newest version for all installed mods without any work on your part, and makes sure that everything stays synced nicely.

The last thing is that, because some variables need to be saved in order to keep track of things which the game does not normally provide, such as d8 use stat changes or damage taken per level with bloody lust, stat api needs to make use of the save file to store its own data. Unfortunately, it cannot save data to a separate file - however, I did write a nice set of functions which can be used before and after save/load in your mod to automatically add compatability to allow the mod to save things as it needs to with no other changes to the save/load code on your end. 

# Base Setup:

The two main parts to getting stats api working with your mod are to drag the mod into your folder and load it, and then allow it to piggyback off of your savefile.

To get stat api loaded into your mod and allow it to be used, run the following line at the top of your code:
> pcall(require, "statAPI")

(The reason for the pcall is that stat api intentionally throws an error at the end of itself so that all other versions can still require a file with the same name.)

Once that is done, some basic save/load code most be set up. If you already have save/load code in your mod, then you only need to add 2 quick lines.
First in your save function, append the output from
> stats.GenerateSave()

to the beginning of your save data.
> saveData = stats.GenerateSave() .. saveData
  
In your load file, then we need to do the opposite. After loading your mod data into some string run the following:
> modData = stats.LoadSave(modData)

This will remove the stat api stuff from the string and allow you to just continue with your save data as normal, while allow stat api the save functionality it needs for a couple items.

If you do not already have some save/load functions, then just copy these two default ones in and make sure to update your mod name as needed.

> function mod.load(fromSave)  
>   if fromSave then  
>      local modData = mod:LoadData()  
>      stats.LoadSave(modData)  
>   end  
> end  
> 
> mod:AddCallback(ModCallbacks.MC_POST_GAME_STARTED, mod.load)  
> 
> function mod.save()  
>   local saveData = stats.GenerateSave()  
>   mod:SaveData(saveData)  
> end  
>
> mod:AddCallback(ModCallbacks.MC_POST_NEW_LEVEL, mod.save)  
> mod:AddCallback(ModCallbacks.MC_PRE_GAME_EXIT, mod.save)  
 

# Using Stat Api:

There is one main function that stat api uses, and one secondary function that can be used to undo that effect. There is also one enumeration for the various stages that stat api goes through in its process of rebuilding the stats. To use stat api, simply replace your mod:AddCallback(ModCallback.MC_EVALUATE_CACHE,...) functions with a stats.AddCache(...) function with the proper changes. Your functions for evaluating stats will then be given an additional input variable which is the current stage that stat api is processing, so you can just check for the correct stage to apply various changes much like you would check for the right cache flag before changing the stats.

> **StatStage:** Enumeration of the various stages in stat api. 
> 
>  **MULTI** = 4, -- Last stage, used by mostly every multiplier item in the game, from magic mush to soy milk.  
  **FLAT** = 3, -- These are changes that are unaffected by any base stat changes (such as the cancer trinket's -2 firedelay).  
  **BREAK_MULTI** = 2, -- These are the multiplier used for things such as character damage multipliers, and some niche items like polyphemus.  
  **BASE** = 1, -- Basic stat changes. These are affected by whatever equations the base game uses, such as tears being capped at 5 and damage scaling decreasingly with each increase.  
  **ALL** = 0 -- This is mostly an internal thing, but you can theoretically use it. Every stage will run in order when it is used as an input to a function.  


> **stats.AddCache(mod, func, cacheFlag, stage, name)**  
  Adds your functions to the stats api instead of just the base stats callback. The syntax is designed to be similar and simple to the basic AddCallback function but with some improvements and changes to just focus on stats. 
>
> **(mod, player, cacheFlag, stage)** (same as the regular evaluate_cache, but now with stage as another returned variable)  
>
> **Inputs:**
>  **mod:** your mod variable, same as with most functions. If you don't care that it has your mod function in your callbacks specifically, just use a : and it will use whichever stat api mod version it cares for instead. Using your own mod allows you to remove functions from the cache based on if your mod added them.  
  **func:** the function which you want to add to the cache. This is the same thing you would give to the AddCallback function.
  **cacheFlag:** since this is a stat-specific function, you can give a cacheFlag to only run your function on that specific flag. Giving nil will cause it to work on all flags like normal.  
  **stage:** the stage you want the function to run on. Just like above, give nil to run on all stages.  
  **name:** whatever you want to name this specific function that you add. This allows you to specifically remove functions from the cache later based on name.  


> **stats.RemoveCache(mod, name)**  
  Removes functions from stats api. It will remove all functions matching your input conditions from stat api, so if two things are named the same under different mods and you don't supply a mod, it will remove the other one as well. Alternatively, you can simply supply a mod and no name to remove all callbacks from your mod, or supply nothing to clear the cache of all callbacks.   
>
> **Inputs:**  
  **mod:** the mod variable that you want to remove the callbacks from. If nil, runs on every mod.  
  **name:** string that represents the name of the functions you want to remove. If nil, does everything under the specified mod.  

