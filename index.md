# macOS Recovery

Recently, I woke my MacBook Pro from sleep mode only to watch the screen briefly display before the system immediately rebooted. What followed was an intense, week-long recovery journey that ultimately resulted in a full macOS restoration —- without relying on a Time Machine backup. Even unsaved documents reappeared, ready to be saved, and windows returned to their correct positions.

Throughout this process, I picked up several valuable lessons that I believe are worth sharing. If you've faced a similar macOS issue, feel free to contribute by submitting a PR or forking this repository.

Before diving into the detailed troubleshooting steps, here are a few key takeaways...

## Lessons Learned

1. If you have a Time Machine backup, no matter how old it is, *** **do not delete it** ***.
1. If you suspect macOS was in the middle of an update when a reboot occurred, give it a full 24 hours with the spinning wheel/progress bar to see if it resolves itself.
1. If you use Docker, be aware that the `Docker.raw` file can significantly increase in size during an `hdiutil` backup.

## Environment

- Intel MacBook Pro
- macOS Sonoma
- 1TB internal SSD (825GB in use, encrypted Data volume with FileVault)
- 2TB external SSD:
  - 1TB for Time Machine (600GB in use)
  - 1TB for files (400GB in use)
- A 6-month-old Time Machine backup from before the MacBook Pro went into Apple for a screen repair
- A software developer who wouldn't accept failure

## The Details

### The Crash

Here’s some context for the initial crash: I had downloaded the macOS Sequoia update but hadn’t installed it yet. Before putting my MacBook Pro to sleep, I had spun up a Linux VM in VMWare Fusion and was experimenting with a newly purchased YubiKey. It’s unclear which of these — if not all — triggered the crash, but they may have played a role.

Following the crash, macOS appeared to reboot without issue. With pending updates and the system freshly restarted, I decided to check for additional updates and attempted to install them — this failed. I began encountering an `OSISClient Interrupted!` error, and no macOS updates would install. Realizing this was more than a minor hiccup, I decided to update my 6-month-old Time Machine backup, just in case.

Upon connecting the Time Machine drive, I was unexpectedly prompted to "Inherit Old Snapshots." I assumed this was related to the screen repair Apple had performed on my MacBook Pro months earlier. I opted to inherit the snapshots and attempted to create a new one. Unfortunately, with 600GB already used by previous snapshots on the 1TB Time Machine volume, creating a new snapshot failed — it wouldn’t differentially update the last successful snapshot as it typically does.

Hoping a restart may resolve the update issue, I decided to reboot again.

After a few hard reboots — likely during active updates that appeared frozen — my Mac was rendered unbootable. Normal mode, single-user mode, and safe mode all failed. The recovery console was my only hope.

At this point, I considered restoring from the 6-month-old Time Machine backup. However, I hesitated, not wanting to potentially lose the many unsaved files and documents I’d left open before this predicament unfolded.

### The Backup

From the recovery console terminal, I confirmed that all my files, including those updated the night before, were still intact — there was hope for a full recovery.

I first tried restoring from the Time Machine backup through the recovery console, but attempting to `chroot` into the normal macOS directory failed due to `dyld` errors, specifically the inability to load the dynamic library `libSystem.B.dylib`. Pursuing a fix for this issue seemed like it would be an exercise in futility.

Next, I attempted to create a full image of the internal 1TB SSD. In Disk Utility, I selected the compression option since the external SSD’s files volume only had 600GB available, while the internal SSD had 825GB of data. I had hoped the compression would make it fit. After 12–24 hours of backing up, Disk Utility failed with a cryptic error, canceling the process.

Undeterred, I switched to Terminal and ran:  
`hdiutil create -srcfolder /path/to/source -volname "Volume Name" -format UDZO -o /path/to/output.dmg -verbose`

Unfortunately, the computer went into sleep/hibernation during the backup, stopping the process. That’s when I learned about `caffeinate -d` to keep the display on and prevent sleep/hibernation, which became a staple for the following days.

Monitoring progress with `watch df -h` in another terminal window, I saw the backup fail again, this time due to running out of space. Knowing my internal SSD data was still safe, I wiped the 1TB Time Machine volume to free up space for a new backup -- in hindsight, a huge mistake having not backed it up elsewhere first.

Despite this, the backup still failed. Even with compression enabled, the 825GB of internal SSD data somehow exceeded the 1TB capacity of the external SSD. Monitoring `df -h`, I watched the backup reach 1TB used before failing with an "out of space" error.

I then tried to back up smaller folders alone using `hdiutil` options like `-nocrossdev`, `-noexpand`, and different `UD` formats, but encountered permissions issues and other errors. The Sparse Bundle (`UDSB`) format was the only option that worked.

Even with Sparse Bundle, backups failed with "out of space" errors. I decided to backup the 1TB files volume on the external SSD to another spare drive and wiped the entire 2TB SSD for the backup.

Finally -- success with a command similar to:  
`hdiutil create -srcfolder /Volumes/Macintosh\ HD\ - Data/ -volname "Backup Data" -format UDSB -o /Volumes/backup/data.sparsebundle -verbose`

I did not back up the `Macintosh HD` system volume as it caused too many issues, but the `Macintosh HD - Data` volume contained all the critical files. As a precaution, I manually copied as many system volume folders as permissions allowed to the external SSD, though I didn’t end up using them during the restoration.

To verify the backup, I connected the external SSD to another Mac and confirmed the sparse bundle’s contents and size. During this verification, I discovered the `Docker.raw` file, typically 21GB, had ballooned to 260GB! It seemed Docker’s layers were being expanded during the backup. Backing up that file separately would have saved considerable trouble.

### The Standard Repair Attempt

With a verified backup successfully loading on another computer, I shifted focus to repairing the existing installation, uncertain of how recovery from a data volume image would play out.

My first step was to remove the update/upgrade files — this achieved nothing. I then attempted to diagnose and repair the system volume, but again, no success.

Running First Aid in Disk Utility from recovery mode on the system volume resulted in errors like `Space manager free queue trees are invalid` and `btn: invalid key order: index 1 is greater than index 2`.

Verbose mode revealed additional errors, including the disk being out of space and more `free queue` issues. Oddly, the drive seemed to have free space available. To be sure, I removed more files, but the issue persisted.

I tried every recovery command I could find: resetting NVRAM and the Secure Boot Controller (SBC), toggling `csrutil` on and off, and booting into single-user mode, where I encountered errors like `failed to finish all transactions in sync() - No space left on device`.

Safe mode appeared to show progress after hours of spinning wheels and progress bars, but ultimately led nowhere.

Concluding that the underlying issue — possibly corrupted core system files or a damaged partition table — had left the system volume’s filesystem irreparably corrupted, I decided it was time to reinstall macOS.

### Reinstall macOS

When reinstalling macOS, my goal was to preserve the `Data` volume to avoid the hassle of moving all my data off the system and back again later. To achieve this, I focused on reinstalling the OS without affecting the `Data` volume. For macOS reinstallation, there are two main options: regular recovery mode and internet recovery mode.

I started with regular recovery mode but consistently encountered a `PKDownloadError` in the installer log. Each time, the installer download stopped around 9.5GB, as confirmed by checking its size through the terminal.

I tested both WiFi and Ethernet connections, suspecting a network issue, but neither resolved the problem. Moving to internet recovery mode yielded the same error.

Since both recovery options were downloading the installer to the `Data` volume, which shared the same container volume group as the corrupted System volume, I suspected that the corrupted free queue space manager errors were affecting both volumes.

Thankfully, a friend with an Intel Mac was able to download the macOS Sonoma installer and create a bootable USB drive for me to use.

Important: When booting from a USB drive, you must enable USB booting in the recovery settings.

With the macOS Sonoma installer on a bootable USB drive, I attempted once more to repair my macOS System volume. Unfortunately, the same free space errors persisted.

Left with no other choice, I decided to wipe the System volume. I erased just the System volume and attempted to reinstall macOS, but the errors remained unresolved.

### Full Wipe

The next step was to wipe the entire drive. During the initial reformatting process, I enabled encryption, which caused issues, so I reformatted again using unencrypted APFS. With the drive prepared, I booted from the macOS Sonoma installer USB and successfully reinstalled macOS.

After the reinstall, I wanted to restore the `Data` volume to get the system as close to its previous state as possible. Booting into recovery mode, I attempted to restore only the `Data` volume. Unfortunately, this process consistently overwrote both volumes in the volume group. After multiple attempts and variations, it became clear that restoring a single volume in a multi-volume group wasn’t possible.

As a last resort, I mounted the externally attached `Data` volume and manually copied all the files to the newly created internal `Data` volume. While the copying process seemed to work, I immediately encountered permissions issues on the files. 

Assuming some changes had occurred due to booting with the new `Data` volume, I decided to wipe the disk once more. I performed another full reinstall of macOS, followed by another manual copy of the `Data` volume files.

### Permissions Updates

After completing the reinstall and file copying process, I updated the ownership of the user folder using: `chown -R 501:staff /users/[user]/*`. I also applied the same command to the hidden files within the user folder to ensure proper permissions.

Finally, a little over a week in and my machine fully rebooted -- all unsaved files reappeared upon login!
