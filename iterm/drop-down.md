# Configure iterm2 pop-up/drop-down window

This is often called a `quake` or `guake` style terminal (I think)

## Steps

### One - Set up basic hotkey window

#### Path
  - `iterm` Preferences
    - under `Keys`
        - under `Hotkey`

Click on `Create a Dedicated Hotkey Window...`

A new window will pop up. Here are my current settings:

`Hotkey:` left blank, but can be your prefered key combo

`Double-tap key:` I prefer this, so the checkmark is checked, and I selected `Option` as my double-tap key.

  [] `Pin hotkey window (stays open on loss of keyboard focus)`

    - left unchecked, I prefer the window to disapper if I click out of it.

  [] `Automatically reopen on app reactivation`

    - left unchecked, I haven't tried this one, but I don't think I would like that setting

  [x] `Animate showing and hiding`

    - checked, this is just personal preference

  [x] `Floating window`

    - checked, I consider this a must-have. This allows the terminal to open over other windows.

  `On Dock icon click:` I've selected `Don't do anything special`


TODO: I'm pretty sure there's one more promt after hitting select here, check if there is, and add what I chose.

### Two - Make the terminal actually look/behave like a drop-down (or pop-up) terminal

#### Path
  - `iterm` Preferences
    - under `Profiles`
      - `Profile Name`: select your drop-down profile (`guake`)
        - under `Window`

`Style:` menu: select your prefered window location

  - I like `Full-Width Top of Screen`

`Screen:` menu: If you have more than one monitor, this will determin which monitor will display the terminal and the right option depends on personal preference.

  - I like the `Screen with Cursor` option, that way, wherever my mouse is (usually on the screen that I'm looking at) is where the terminal window will appear.

`Space:` menu: If you use multiple workspaces, aka have several full-screen apps open at once and use `ctrl+left/right arrow key` to swap between them, this setting determines in which space the drop-down menu is allowed to appear.

  - I like the `All Spaces` option.

Most of the other settings are up to personal preference, but here are some of my. settings (which may just be the defaults):

[] `Hide after opening`

[] `Open toolbelt`

[] `Custom window title: [blank]`

[] `Force this profile to always open in a new window, never in a tab`

[x] `Use transparency` (If you use this option, there is a transparency slider on this page that is useful)

### Three - Misc. settings

#### `Preferences` -> `Appearance` -> `General`

I have the `Exclude from Dock and ⌘-Tab Application Switcher` box checked, but not it's child one checked

#### `Preferences` -> `General` -> `Closing`

I have unchecked the `Quit when all windows are closed`. This one is pretty important if you plan on primarily using just the hotkey window

#### `MacOs System Preferences` -> `Security & Privacy` -> `Privacy` tab -> `Accessibility`

I have added `iterm` to the allow list (but I don't remember if getting this working was part of that)

### Four - Things I've forgotten

There are probably a few steps left out here. The search terms you're going to want to google will most likely include `quake` `drop-down` and Stack Overflow (and its sub communities) are great resources.

