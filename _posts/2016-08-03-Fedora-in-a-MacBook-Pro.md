---
layout: post
title: Fedora in a MacBook Pro
subtitle: Tweaks & Tricks to make it work
bigimg: /img/fedora.jpg
---

Some months ago I got a MacBook Pro 13.3 for work, and since I'm much more comfortable in a Linux environment, I decided to install Fedora. I really like Apple's hardware, especially the retina display, but for me, Fedora was the right OS to work with.

So let's see what I did in order to install Fedora 24.

## Boot Media

Still running OS X, download Fedora 24 Workstation ISO file from the following [link](https://getfedora.org/workstation/download/). Then, follow these steps:

* Open terminal
* diskutil list
* Insert the USB pendrive that you want to make bootable
* diskutil list
* Take the new disk in the list. It should appear as /dev/diskX being X a certain number. (i.e. /dev/disk1)
* diskutil unmountDisk /dev/disk1
* sudo dd if=~/Downloads/Fedora-Live-Workstation-x86_64-24-10.iso of=/dev/rdisk1 bs=1m (see there is an 'r' in front of the disk name. This will make this process faster)
* Wait for some minutes to finish.

If you want to make a dual boot system (in order to keep Mac OS X), I suggest to open diskutil graphic application and make an empty partition of your disk.

## Installation

Insert the USB drive where Fedora 24 ISO has been located and restart the laptop. Make sure to press Alt when it's booting up. That way, you will be able to choose Fedora from the USB stick.

There are dozens of tutorials about how to install Fedora in general. Just follow the steps from the screen and configure date/timezone, users, network, partition table, etc.

Once the process has finished, remove the USB stick and reboot the system. Now, GRUB boot loader will show up and you will be able to boot Fedora 24!

## OMG: I cannot boot Mac OS X!

## Fixing things that are not working

### Wireless Card

It really depends on the Wireless Network Card your MacBook has got, but in many ocassions, Wifi connection doesn't work out of the box. It has a very simple solution:

~~~
sudo dnf install akmod-wl
~~~

Reboot and bang! it's working...

### Camera

Same kind of issue with the FacetimeHD camera integrated in the MacBooks. However, there is a great effort building a reverse-engineered driver that make it compatible with most of Linux/GNU distros:

[https://github.com/patjak/bcwc_pcie]()

In order to build the driver succesfully, I'd recommend to install the following packages and make the build as **root**:

~~~
sudo dnf groupinstall "Development Tools"
sudo dnf install kernel-devel
~~~

There is a problem extracting the firmware so a very good friend of mine (Jean-Philippe Jung) has stored it here.
Get the camera firmware (v1.43 as of OS X 10.11 El Capitan) from: 

(https://www.dropbox.com/s/rt1o0enqo361o9b/firmware.bin?dl=0)

Firmware has to be copied to /usr/lib/firmware/facetimehd/firmware.bin

NOTE: This driver has been around for a while and it was valid up to Fedora 23. For Fedora 24 (kernel 4.5), there is this other fork that works fine (even the firmware extraction):

(https://github.com/engstk/bcwc_pcie)

### Backlight

See: (https://github.com/patjak/mba6x_bl)

## Software and cuties

Some time ago I saw this video on youtube and it's kind of my reference post-installation:

(https://www.youtube.com/watch?v=TQ68l10Fy5M)

It explains very cool things like Gnome Tweak Tools, Fedy, Fusion Repos and other bunch of useful stuff. I'm gonna list some of my daily work applications:

* HexChat (IRC client)
* Chrome
* Atom
* Spotify

And for the shell terminal:

* Tmux
* Powerline
* Vim
* Vundle (Vim plugin manager)

Finally, this is the list of my Vim plugins to basically convert it into an IDE:

~~~
" " let Vundle manage Vundle, required                                                                                                                                                                        
Plugin 'VundleVim/Vundle.vim'                                                                                                                                                                                 
                                                                                                                                                                                                              
" vim-bubblegum                                                                                                                                                                                               
Plugin 'baskerville/bubblegum'                                                                                                                                                                                
" vim-fugitive                                                                                                                                                                                                
Plugin 'tpope/vim-fugitive'                                                                                                                                                                                   
" vim-gitgutter                                                                                                                                                                                               
Plugin 'airblade/vim-gitgutter'                                                                                                                                                                               
" vim-NERDTree                                                                                                                                                                                                
Plugin 'scrooloose/nerdtree'                                                                                                                                                                                  
Plugin 'Xuyuanp/nerdtree-git-plugin'                                                                                                                                                                          
" vim-tagbar                                                                                                                                                                                                  
Plugin 'majutsushi/tagbar'                                                                                                                                                                                    
" vim-flake8                                                                                                                                                                                                  
Plugin 'nvie/vim-flake8'                                                                                                                                                                                      
" vim-python-pep8-indent                                                                                                                                                                                      
Plugin 'hynek/vim-python-pep8-indent'                                                                                                                                                                         
" jedi-vim                                                                                                                                                                                                    
Plugin 'davidhalter/jedi-vim'                                                                                                                                                                                 
" vim-virtualenv                                                                                                                                                                                              
Plugin 'jmcantrell/vim-virtualenv'                                                                                                                                                                            
" vim-licenses                                                                                                                                                                                                
Plugin 'antoyo/vim-licenses'                                                                                                                                                                                  
" vim-snipmate                                                                                                                                                                                                
Plugin 'MarcWeber/vim-addon-mw-utils'                                                                                                                                                                         
Plugin 'tomtom/tlib_vim'                                                                                                                                                                                      
Plugin 'garbas/vim-snipmate'                                                                                                                                                                                  
Plugin 'honza/vim-snippets'                                                                                                                                                                                   
" vim-syntastic                                                                                                                                                                                               
Plugin 'scrooloose/syntastic'                                                                                                                                                                                 
" vim-tbone                                                                                                                                                                                                   
Plugin 'tpope/vim-tbone'                                                                                                                                                                                      
" tmuxline                                                                                                                                                                                                    
Plugin 'edkolev/tmuxline.vim' 
~~~
