<!--
[How to fix Windows 10 KB5034441 failing to install with 0x80070643 error]([[[https://www.makeuseof.com/windows-11-set-up-without-internet-connection/](https://www.windowslatest.com/2024/01/15/windows-10-kb5034441-fails-with-a-0x80070643-error-but-theres-a-fix/)](https://www.windowslatest.com/2024/01/15/windows-10-kb5034441-fails-with-a-0x80070643-error-but-theres-a-fix/)]
-->

# How to Fix Windows 10 KB5034441 Failing to Install with 0x80070643 Error

1. Open Command Prompt (cmd) as an administrator.
2. Check if WinRE is installed by running reagentc /info. If it’s there, you’ll see a “Windows RE location” with a path.
3. Turn off WinRE by running reagentc /disable.
4. Get ready to make a new recovery partition by shrinking the OS partition.
5. Use diskpart by running diskpart in Command Prompt.
List the disks and select the one with OS by running the command sel disk<OS disk index>.
Find and select the OS partition by running the command sel part<OS partition index>.
After selecting the partition, run the command: shrink desired=250 minimum=250.
Select and delete the old WinRE by running the command: sel part<WinRE partition index>, and then run the command delete partition override.
Finally, you can create a new recovery partition. In this step, check if your disk is GPT (GUID Partition Table) or MBR (Master Boot Record) by running the list disk. When you run the list disk command, you’ll see a new list of drives. A drive is “Gpt” if you see an asterisk (*) in the “Gpt” column.

If you do not see the asterisk mark, the drive is MBR.

For GPT, use the following commands:

 create partition primary id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
After running the above command, run gpt attributes =0x8000000000000001.

Turn WinRE back on with reagentc /enable.
Confirm WinRE installation with reagentc /info.
