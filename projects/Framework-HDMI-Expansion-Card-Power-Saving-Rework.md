[HDMI Expansion Card Power Saving Rework]([https://guides.frame.work/Guide/HDMI+Expansion+Card+power+saving+rework+(Beta)/193]

# HDMI Expansion Card Power Saving Rework (Beta)

## Introduction
After over a year of prototyping and experiments, we've been able to come up with a way to reduce system power consumption when an HDMI Expansion Card is present, by making the card pretend that it is not a display output when there is no monitor connected.

You can rework an existing HDMI Expansion Card to have this behavior by following this guide to solder a jumper wire internally and flash new firmware to it. Note that this requires advanced soldering skills, so we recommend that you only try it if you're familiar with extremely fine-pitch SMT rework and have the necessary tools for it. Damage caused by failed reworks is not covered under warranty.

Beta release: Note that the firmware and instructions are still in beta, so there may be further revisions. The firmware update tool from our supplier currently is only available for Windows. We're also running a beta test with 2nd Gen HDMI Expansion Cards with Batch 1 of 13th Gen that has a slightly different modification.
