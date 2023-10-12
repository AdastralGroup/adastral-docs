# The Adastral Sort-of-Technical Document (aka The Blue Book), v0.0.1 (2023-10-12)

## ISSUES WITH THIS DOCUMENT  

- seperation between specification and the implementations is a bit jank
- needs more diagrams
- possibly too semantic?
- terminology isn't even that frequently used
- doesn't cover everything probably

## Terminology
Important shorthand to know, even if you haven't read this before. This keeps the document consise. This list will most likely expand.

- SM: Sourcemod
- GS: Game Server
- SB: southbank
- DS: Download Server, central location where the games are stored

*Text in italics implies something relating to this specific document version, such as the current state / plans as of the time of writing.*

## Adastral Design Overview

Adastral is composed of various components, modules and submodules within.
This modularity stops us from running into the classic issue of "Oh no, it's a big mess",
and importantly, allows us to use work other people (probably smarter than anyone working on this) have done.  
The top level is made up of a few components:

1. Clientside (what people'll be using)
2. Serverside (what runs on the DS)
3. Devside (What the SM developers will use to upload their games)

Note that clientside also includes GS maintainers, as they'll use a CLI version of this. 
This'll come up later (probably).

*Currently as of v0.0.1, there is the most documentation for the clientside, for a number of reasons, mainly that there's nothing solid for the devside as of yet (though ideas have been floated, and will be in this document) and there's no specific server-side application - it's mainly just static hosting.*

## The clientside, in detail

The clientside is made up of several tiered "levels". These levels perform operations separate from each other,
and can be swapped out. In theory, there's a unified API between layers, but in practice, it's only practical for L2->L1.

The layers are:

- L3, GUI. That's it. This module is written using the Godot game engine for reasons specified later, and the current implementation is codenamed "belmont".
- L2, The "glue" layer. This handles everything that isn't UI and versioning related. More specifically, this handles steam integration, sanity checks, finding installations, self-updating, importing custom Southbank files (discussed later in the document), and importantly, downloading. These are handled in seperate subcomponents, so that different downloading backends can be easily swapped in. Current implementation is codenamed "palace".
- L1, versioning. This has one job - update a game. There can be multiple L1's, in the case that the SM devs prefers one method. Current implementation is based off of TF2CDownloader's implementation (kachemak), and is called emley.

### The L3 

The L3 is primarily the GUI component. It avoids handling as much logic as possible, so generally not much technical occurs here.  
However, one main feature is the ability to dynamically add/remove games based on what's on the server, instead of baking it in. This contains colours and images necessary to properly and cleanly display games on the UI, and this data is contained within the SB protocol.   

#### Our implementation (And a defense of Godot)
*as of v0.0.1, the L3 has zero logic hooking it into the L2. As this is implemented, more will be written here.*  

As stated before, we're using the Godot game engine for the L3, and this comes with several advantages:

- Blissfully simply UI editing and prototyping
- Advanced graphical effects easily/simply done
- Cross Platform packaging built in
- Small size.

The L2 and L3 communicate via the GDExtension interface. There's currently a "shim" which is a L2 sub-component which's main task is just to handle the Godot<->C++ interop.

### The L2

The L2 handles most tasks, and as such is modular within itself. These tasks include, but are not limited to:

- Sanity checks (disk space, network, security)
- Downloading
- Finding the sourcemods folder and other steam related files needed (eg. libraryfolders.vdf)
- Server communication

It's sort of the brain of the whole client.
#### Our implementation
Notably our implementation it also includes the shim needed to communicate with the godot L3.  
*As of v0.0.1, The main part of the L2, palace, hasn't been written. The shim has been worked on (this is adastral-manager).*

### The L1

This handles versioning. In its most simplest form, it'd provide the list of versions, and then when updating, it'd take in a target version and an old/existing version.  
It should do all the updating needed, but any network IO should call back to the L2 (mainly downloading files).  

#### Our implementation
*As of v0.0.1, this currently has the most work as Emley.*  
  
This is, as stated before, based off of the existing TF2CDownloader solution called kachemak. We've made / are planning to make alterations to it but it's best to explain how it works first.
##### How Kachemak works
It leverages the Butler utility for applying patches between versions and "healing" existing versions, and uses .tar.zstd archives when patching isn't faesible. The client looks for a "versions.json" file which contains all the necessary data needed to update the game. It uses aria2c for downloading. The general format is as such:
```  
{
    "versions": {
        "0": {},
        "1": {"heal": "example.zip"}
        "2": {"url": "example", "file": "example.tar.zstd", "heal":"example.zip", "presz": 100000000, "postsz": 1000050000, "signature": "example.sig"}
    }
    "patches": {
        "1": {"url": "example", "file": "example.pwr", "tempsize": 100000000}
    }
}
```
Note that all the fields specifing a file are relative to the location of the `versions.json` file.
To explain the specific fields:

- versions contains a key of every revision, even if no server-side data is available for it. The client uses this to check if a version is valid. Inside these keys are objects containing data specific to that revision.
    - `url`: This contains a URL to any file that downloads the archive of the revision.
    - `file`: the name of the archive of the revision. The seperation between this and `url` will be explained later.
    - `heal`: A .zip archive of the revision, notably with no containing directory. Needed for butler verification.
    - `signature`: The name of the signature file on the server.
    - `presz` and ``postsz``: (ok I have no idea something to do with size)
- patches contains a key of every version a patch to the latest version is available for. Inside the keys are objects containing data specific to the patch.
    - `url`: This contains a URL to any file that downloads the patch.
    - `file`: Name of the patch.
    - `tempsize`: the temporary space required to apply the patch.

This works as follows:

1. If client has a version that matches a key in `versions`, then it will verify the instally by running `butler heal x` where x is the url specified in the heal field of the version. if that doesn't exist, do a full reinstall.
2. If the client successfully heals, look in `patches` for the current version. If it's there, then download the file to some temp location using the `url` provided - this could be a link to effectively any file that ends up downlading the file in `file` using aria2.
3. Apply the patch using `butler apply`.
4. Write the latest revision to the version file (see on disk format).

##### Our planned changes  
Kachemak has some shortcomings, namely:

- There's functionally the same archive stored twice since butler can't support .tar.zstd archives
- every time a new version is prepared, every patch needs to be updated, leading to very slow builds after a while
- it uses metalinks and aria2c, which isn't ideal for reasons out-of-scope for this document
- To get the latest version, you need the version numbers to be incremental (since it's obtained by doing a sort on the keys since dict key order isn't guaranteed, apparently), which is inconvenient if you use something that isn't easily an incremental number
- Butler is a seperate binary that needs to be baked in.

*as of v0.0.1, the following changes haven't occured and aren't finalised.*  

To allieviate these issues, the following changes are to be made:

- Merge the heal and archives to reduce space taken
- use torrent files with webseeds in place of metalinks, and use libtorrent to download the files
- Figure out a way to more efficiently store patch files (i.e tiered patches)
- Avoid having to store the entirety of the butler binary
  
*Some of these have already been implemented in ofinstaller-beans.*



## Protocol(s)

Adastral is centralised, as in that it has a central server, but it's entirely possible for third parties to host their own DS for their own games. There should be the ability to import games from external DS instances. How the client and the DS communicate is via a few JSON files. 

The main file lies at the root of the server, and specifies:

- what games are available on the server, and the path to them
- required GUI configuration to display the games
- the versioning/downloading method for each game

The specification for this file is known as southbank. Parsing for this is done on the L2 on the client, and generally it shouldn't be modified too much as it's not changed on a game update.

## On Disk Format
### Clientside
*As of v0.0.1, this isn't final, or even relatively solid.*  

The client primarily stores information about itself (GUI configs, etc) in either `%appdata%/Adastral/config.json` on Windows or `~/.config/Adastral/config.json` on Linux.

Game specific data, like the version number and any exclude paths, is stored in `.adastral`, in the game's root folder. This file is a JSON file, and generally follows the format:
```
{
    "version": "0",
    "exclude": "gameinfo.txt"
}
```
Do note that the version number must be a numerical string.


