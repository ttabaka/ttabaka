<!--
[How to fix Windows 10 KB5034441 failing to install with 0x80070643 error]([[[https://www.makeuseof.com/windows-11-set-up-without-internet-connection/](https://www.windowslatest.com/2024/01/15/windows-10-kb5034441-fails-with-a-0x80070643-error-but-theres-a-fix/)](https://www.windowslatest.com/2024/01/15/windows-10-kb5034441-fails-with-a-0x80070643-error-but-theres-a-fix/)]
-->

# How to Fix Windows 10 KB5034441 Failing to Install with 0x80070643 Error

1. reagentc /info
2. reagentc /disable
3. reagentc /info
4. diskpart
5. list disk
6. select disk 0
7. list disk
8. list partition
9. select partition 4
10. delete partition override
11. list partition
12. create partition primary id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
13. gpt attributes =0x8000000000000001
14. list partition
15. exit
16. reagentc /enable
17. reagentc /info
