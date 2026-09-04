<!--
[How to Select Windows 11 Home or Pro Edition During Installation](https://www.makeuseof.com/windows-11-select-edition-during-install/)
-->

# How to Select Windows 11 Home or Pro Edition During Installation

So, how do you force Windows 11 Setup to show the select edition screen during installation? You can achieve this by modifying the bootable media or the ISO image to include the EI.cfg file. Here we show you how to select the Pro edition while installing Windows 11.

1. First, you need to create a bootable Windows 11 USB drive. If you don’t have an ISO, you can download the Windows 11 ISO image from the Microsoft server.
2. Next, open a new Notepad file. To do this, press Win + R, type notepad, and click OK.
3. In the Notepad window, copy and paste the following lines:

```
[Channel]
_Default
[VL]
0
```
4. Press Ctrl + S to open the Save as dialog.
5. Here, type the file name as ei.cfg. Next, click the drop-down for Save as type and select All Files.
6. Click the Save button to save the file to your PC.
7. Next, connect the bootable USB flash drive to your PC.
8. Press Win + E to open File Explorer.
9. In the left pane, click on This PC.
10. Next, double-click on the bootable USB flash drive to view its content.
11. Double-click on the Sources folder to open it.
12. Now, copy and paste the ei.cfg file into the Sources folder.

Once done, safely eject the USB flash drive. You can now boot using the installation media and select the Windows 11 Pro, Education, or the core Home edition from Windows Setup.
