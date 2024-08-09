Below is my steps. I already have IBus Pinyin IME working, you may need extra steps from Ubuntu new installation.
1. Install fctix5
    
    Run `sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-frontend-gtk3`
    
2. Open Settings -> Keyboard -> Input Source, remove Chinese Pinyin from the list.
    
    Fcitx IMEs aren't shown in this list. The Gnome desktop IME related settings (e.g. hotkeys) won't affect Fcitx, you need to go to Fcitx Configuration to configure them (see below).
    
3. Run `im-config`, follow the wizard and choose fcitx5 as IME.
    
4. Run `sudo vi /etc/environment` and add below environment variables:
    
    ```
    GTK_IM_MODULE=fcitx
    QT_IM_MODULE=fcitx
    XMODIFIERS=@im=fcitx
    SDL_IM_MODULE=fcitx
    GLFW_IM_MODULE=ibus
    ```
    
    This step is important, without it you will be unable to switch IME in most of the applications.
    
5. Open Tweaks (install by `sudo apt install gnome-tweaks`) and add Fcitx 5 to Startup Applications.
    
6. Reboot
    
7. After system boot, you should be able to see a keyboard icon at the system status bar, which is the Fcitx application. Click on it and choose "Configure" to add IME, change hotkey, etc.