**Teardown & Pwnage: Hacking a $5 Wi-Fi Repeater**
### I - Beyond the Basics: The $5 Mystery

If you've been following my previous deep dives into repurposing legacy hardware, you know I can't resist cheap, forgotten electronics. This time, I picked up a generic Wall-Plug Wi-Fi Repeater for about $5.00.

Opening it up revealed a MediaTek MT7628KN System-on-Chip featuring a MIPS32 24KEc core humming at 575 MHz. The system RAM is an 8 MB DDR1 module permanently baked inside the CPU package, making it completely non-upgradeable. Next to it sits a 4 MB Macronix MX-12G SPI flash chip for non-volatile storage.

A quick probe with my multimeter found the Holy Grail: a 3-pin header. Pin 1 was Ground, Pin 2 was TX (sitting active high at 3.24V), and Pin 3 was RX (sitting at 0V). We had our UART debug interface.

### II - The Bootloader & The 58-Minute Dump

I hooked up a serial adapter configured to 57600 baud, 8N1, and was instantly greeted by an unlocked console. By mashing the `4` key during boot, I aborted the autoboot sequence and dropped straight into the `MT7628 #` prompt of a Ralink U-Boot environment (version 4.3.0.0).

First rule of hardware hacking: back up the flash memory. I tried the standard `spi read 0 400000` command, but the router threw `malloc error` codes back at me. It simply didn't have enough RAM to buffer the read.

After some digging, I realized MediaTek typically maps their SPI flash directly into the CPU's uncached virtual memory (kseg1) starting at physical address `0xBC000000`. Using the `md.b` command, I successfully read the hex data straight over the serial line.

Because I had to stream 4MB of hex data over a slow 57600 baud serial connection, I piped Minicom into a text file and went to make some tea. Exactly 58 minutes later, I had my dump. A quick `awk` and `xxd` command line pipeline converted the raw text into a pristine `factory_backup.bin`.

### III - The "Linux" Lie

I fired up Binwalk to dissect the binary. It revealed a dual-image A/B failover layout. Running `strings` gave up the factory default SSID (`BFROS_WIFI`) and confirmed `admin` as the default username.

Binwalk also identified an 830.7 KB uImage named `zxrouter` claiming to be a Linux OS. But when I tried to use `lzma -d` to extract the root filesystem, it failed with corrupt data errors.

Why? Because the Linux header was a lie.

This thing doesn't run Linux hhhhhhh. There is no `/bin` or `/etc` directory. The entire operating system, network stack, and application layer is compiled into a single, flat bare-metal executable blob powered by the eCos RTOS.

### IV - Decimating the Web Layer

Since there was no file system to extract, I used `dd` to carve out the web assets directly from the binary and `strings` to isolate the JavaScript and HTML logic. What I found was a masterclass in terrible security:

* **Plaintext Storage:** The hardware NVRAM stores everything—including `SYS_ADMPASS=admin` and Wi-Fi passwords like `WLN_WPAPSK1=88888888`—in absolute plaintext. No hashing, no salt, nothing.


* 
**Fake "Encryption":** The login page takes your password, runs it through Base64 encoding, and drops it into an unsecured browser cookie. Anyone on the network can intercept the cookie and run `echo -n <string> | base64 -d` to get the admin password in clear text.


* 
**Unauthenticated RCE:** The web interface relies on a virtual page wrapper called `do_cmd.htm`. It has zero CSRF protection or session validation. By firing a raw `curl` command like `http://192.168.11.1/do_cmd.htm?CMD=SYS_CONF&CCMD=1&FLG=1`, I successfully triggered a remote, unauthenticated factory wipe.


* 
**The Lazy Parser:** I noticed that `CCMD=1`, `CCMD=10`, and `CCMD=101` all triggered catastrophic reboots or resets. It turns out the backend C parser is completely broken—it only checks if the *first character* of the string is a `1`.


* 
**Reflected XSS:** The hidden `GO` parameter blindly reflects input directly into a JavaScript string. Injecting `GO=';alert('hacked')//` easily popped an XSS execution.


* **No Firewall:** Oh, and the built-in firewall designed to stop SYN floods? It is hardcoded to be turned off by default (`FW_EN=0`).



### V - The Result: taraaaa !

We completely decimated the software stack of this device. The kill chain is fully mapped: An attacker could use the XSS vulnerability to steal the Base64 cookie out of the browser's memory, decode the plaintext password, and then silently drop a CSRF payload to factory-wipe the device.

Even better, while poking around the UART connection, I managed to drop out of the ping utility and land in a live `CMD>` eCos debug shell. From there, I have a direct `OS>reg` and `OS>mem` interface to read and write directly to the CPU's live memory registers.

### VI - The Future

The vulnerabilities are absolute nightmare fuel, but the hardware itself is now completely unlocked. We have serial shell access, a complete flash backup, and the ability to read and write directly to the MIPS processor's memory.

This little e-waste box is officially pwned. Now, the real fun begins: figuring out what kind of custom C code we can inject into that flat memory model to make it do something useful.

Scroll to Top
© 2026 Taha Ed-dafili
Made with ❤️ and powered by Rusty Typewriter theme for Hugo
