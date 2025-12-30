---
layout: default
title: Basic Linux Navigation
nav_order: 3
parent: Setup
---

# Basic Linux Navigation

[&laquo; Return to Setup](index.md)

## Overview

Learn the essential Linux commands for navigating the file system and managing files in your development environment.

Please keep in mind that many of these commands can be run with "flags" to add more functionality. If you want to see the list of flags a command has, the help flag `-h` will often give you a list of flags and what they do. 

## Essential Commands

### Account Management
- `passwd` - Change your password
- `whoami` - Display current username
- `who` - Show who is logged in
- `exit` or `logout` - End your session

> **Important!**
>
> The first thing you should do when logging into a new account is change your password using the `passwd` command. The server will prompt you for your current password, then ask you to enter and confirm your new password.



### Basic Directory Navigation

- `ls` - List directory contents
- `cd` - Change directory
- `cd ..` - Go up one directory level
- `cd ~` - Go to home directory

> **An Example**
>
> In your home directory, if you look around with "ls" you should see a "www" folder.
> To navigate into the folder from your home directory, the command would be:
>
>```
>cd www
>``` 
>
> Once you've navigated into the www directory, to go back up a directory you could run `cd ..` to traverse up. Or just run `cd ~` to go back to the home directory wherever you are.

### File Operations

- `touch filename` - Create an empty file
- `mkdir directoryname` - Create a new directory
- `rm filename` - Remove a file
- `rm -r directoryname` - Remove a directory and its contents
- `cp source destination` - Copy files or directories
- `mv source destination` - Move or rename files/directories
- `nano filename` - Open file with nano editor

> While it will be mostly unnecessary for us to modify files with nano, sometimes it's more convenient to quickly modify something through the terminal without setting up our entire coding environment.

> **An Example**
> 
> If I wanted to create a file called `index.html`, then make a copy of that file and place it into a new directory called `staticpages/` I would run the following commands:
>
>```
>touch index.html
>mkdir staticpages
>cp index.html staticpages/
>```
>
> If I wanted to rename the file during the copy to something like "nav.html", the last command could be modified to be `cp index.html staticpages/nav.html`



### File Viewing

- `cat filename` - Display file contents
- `less filename` - View file contents with pagination
- `head filename` - Show first 10 lines of a file
- `tail filename` - Show last 10 lines of a file

> These commands are useful for quickly viewing what's in the file without going into an editor.

## Practice Exercises

### Exercise 1: Basic Navigation
1. Navigate to your home directory
2. Create a new directory called `practice`
3. Navigate into the `practice` directory
4. Create a file called `test.txt`
5. Open `test.txt` and add your name to the file.

### Exercise 2: Changing the index.html
1. Navigate to your www directory
2. Create a new directory called `backups`
3. Copy the `index.html` file located in `www/public_html` into `www/backups`
4. Navigate to `www/public_html`
5. Open your `index.html` file and look for the `<title>  </title>` tag.
6. Put something between opening and closing tag to change the title of your webpage.
7. Go to the umainecos.org site and confirm that the title of the page has changed.

## Tips and Best Practices

- Use `Tab` for auto-completion
- Use `Ctrl+C` to cancel a running command
- Use `history` to see previously run commands
- Use `clear` to clear the terminal screen

## Next Steps

Once you're comfortable with basic Linux navigation, proceed to [VSCode Extension](vsCodeConnection.md).

---