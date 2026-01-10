title: Fixing Sennheiser Momentum 4 Sound Issues (Popping and Crackling)
date: 2026-01-10 18:29
author: Philipp Wagner
tags: notes
category: notes
summary: Fixing Momentum 4 Sound issues.

My company has given me a Sennheiser Momentum 4 and I had a lot of problems with popping and cracking sound. 

Here is how to fix it:

1. Open the Windows Registry Editor (`Windows Key + R` , type `regedit` and press `Enter`)
2. Navigate to `[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\BthA2dp\Parameters]`. Create the Folder if it doesn't exist.
3. Create a `DWORD` with the value `BluetoothAacEnable`, set the value to `0`
4. Reset Bluetooth

Source: [/r/sennheiser](https://www.reddit.com/r/sennheiser/comments/16vgka2/popping_and_crackling_on_momentum_4/)