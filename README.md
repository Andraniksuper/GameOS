<div align="center">
  <h1>GameOS</h1>
  <p><strong>A Custom, Locked-Down Live Gaming Operating System by Andracompany</strong></p>
  <p>
    <a href="https://github.com/Andraniksuper/GameOS/tree/main"><strong>Download & Source Code</strong></a>
  </p>
</div>

<hr>

<h2>About GameOS</h2>
<p><strong>GameOS</strong> is a lightweight, read-only live Linux distribution built on Arch Linux. It boots straight from a direct-flashed USB drive into an unprivileged <code>player</code> account, launches a PlayStation-style <strong>XMB</strong> interface via RetroArch, and locks down cheats, time-rewinds, terminal access, and application breakouts for a clean, distraction-free retro arcade environment.</p>

<h2>Key Features</h2>
<ul>
  <li><strong>Live-Only Environment:</strong> Runs entirely from RAM via a standard bootable USB flash drive without modifying internal hard drives.</li>
  <li><strong>Streamlined Performance:</strong> Strip-mined of desktop bloatware, running only core system utilities, Mesa graphics drivers, and audio support.</li>
  <li><strong>PlayStation-Style UI:</strong> Boots directly into RetroArch using the modern <strong>XMB</strong> menu driver.</li>
  <li><strong>Strict Lockdown Controls:</strong> 
    <ul>
      <li>Cheats and time-manipulation (rewind/fast-forward) disabled.</li>
      <li>Application exits and menu-toggle shortcuts stripped out (<code>quit_lock = true</code>).</li>
      <li>Terminal and virtual console breakouts blocked via X11 server flags (Ctrl + Alt + F1–F6).</li>
    </ul>
  </li>
</ul>

<hr>

<h2>Prerequisites</h2>
<ul>
  <li>A working Linux environment with root privileges (such as an Arch Linux installation or an Arch virtual machine).</li>
  <li>Internet access to download packages during the build.</li>
  <li>A target USB flash drive to write your final ISO.</li>
</ul>

<hr>

<h2>Step-by-Step Build Guide</h2>

<h3>Phase 1: Prepare Your Build Environment</h3>
<p>Open your terminal in your Arch Linux builder workspace and install <code>archiso</code> along with core build utilities:</p>
<pre><code>sudo pacman -Syu archiso base-devel git</code></pre>

<h3>Phase 2: Customize the Package List</h3>
<p>Modify or create your package profile list to include only core system components, graphics drivers, audio support, and RetroArch:</p>
<pre><code>base
linux
linux-firmware
alsa-lib
alsa-utils
pulseaudio
pulseaudio-alsa
mesa
lib32-mesa
xf86-video-amdgpu
xf86-video-intel
xorg-server
xorg-xinit
retroarch</code></pre>

<h3>Phase 3: Configure Auto-Login for the <code>player</code> User</h3>
<p>Set up an automatic text-mode login for TTY1 so the system bypasses login prompts completely:</p>
<ol>
  <li>Create the systemd override directory:
    <pre><code>sudo mkdir -p airootfs/etc/systemd/system/getty\@tty1.service.d</code></pre>
  </li>
  <li>Create and edit the override configuration file:
    <pre><code>sudo nano airootfs/etc/systemd/system/getty\@tty1.service.d/override.conf</code></pre>
  </li>
  <li>Paste the following configuration:
    <pre><code>[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin player --noclear %I $TERM</code></pre>
  </li>
</ol>

<h3>Phase 4: Configure RetroArch (XMB Graphics & Lockdown Settings)</h3>
<p>Enforce the sleek PlayStation-style <strong>XMB</strong> interface while completely stripping out time-manipulation features, cheats, and exit options:</p>
<ol>
  <li>Create the skeleton configuration directory for user profiles:
    <pre><code>sudo mkdir -p airootfs/etc/skel/.config/retroarch</code></pre>
  </li>
  <li>Create and edit the configuration file:
    <pre><code>sudo nano airootfs/etc/skel/.config/retroarch/retroarch.cfg</code></pre>
  </li>
  <li>Paste the following configuration parameters:
    <pre><code># --- GRAPHICAL INTERFACE ---
menu_driver = "xmb"
xmb_theme = "electric-blue"

#` --- DISABLE REWIND ---
rewind_enable = "false"

#` --- DISABLE FAST-FORWARD ---
fastforward_ratio = "1.0"
hold_fast_forward = "false"

#` --- DISABLE CHEATS ---
cheats_enable = "false"

#` --- LOCK DOWN RETROARCH EXIT & MENUS ---
quit_lock = "true"
input_quit_game_key = "nul"
input_menu_toggle_btn = "nul"
input_menu_toggle_key = "nul"
settings_show_drivers = "false"
settings_show_video = "false"
settings_show_audio = "false"
settings_show_input = "false"</code></pre>
  </li>
</ol>

<h3>Phase 5: Set Up User Provisioning and Startup Scripts</h3>
<ol>
  <li>Create a local bin directory inside the airootfs overlay:
    <pre><code>sudo mkdir -p airootfs/usr/local/bin</code></pre>
  </li>
  <li>Create the initialization script to automatically provision the unprivileged <code>player</code> user on boot:
    <pre><code>sudo nano airootfs/usr/local/bin/simos-init.sh</code></pre>
  </li>
  <li>Paste the script logic:
    <pre><code>#!/bin/bash
if ! id "player" &>/dev/null; then
    useradd -m -s /bin/bash player
    echo "player:simos123" | chpasswd
fi</code></pre>
  </li>
  <li>Make the script executable:
    <pre><code>sudo chmod +x airootfs/usr/local/bin/simos-init.sh</code></pre>
  </li>
  <li>Create an <code>.xinitrc</code> file in the skeleton directory so the environment automatically boots audio and launches RetroArch upon graphical login:
    <pre><code>sudo nano airootfs/etc/skel/.xinitrc</code></pre>
  </li>
  <li>Paste the following:
    <pre><code>#!/bin/sh
pulseaudio --start
exec retroarch</code></pre>
  </li>
  <li>Make it executable:
    <pre><code>sudo chmod +x airootfs/etc/skel/.xinitrc</code></pre>
  </li>
  <li>Configure the shell profile so TTY1 automatically triggers the display server (<code>startx</code>) on login:
    <pre><code>sudo nano airootfs/etc/skel/.bash_profile</code></pre>
  </li>
  <li>Add this block:
    <pre><code>if [ -z "$DISPLAY" ] && [ "$(tty)" = "/tty1" ]; then
    exec startx
fi</code></pre>
  </li>
</ol>

<h3>Phase 6: Patch Terminal and Virtual Console Breakouts</h3>
<p>To prevent users from breaking out of the graphical interface via virtual terminal switching:</p>
<ol>
  <li>Create the X11 configuration directory in your root overlay:
    <pre><code>sudo mkdir -p airootfs/etc/X11/xorg.conf.d</code></pre>
  </li>
  <li>Create the server lockdown configuration file:
    <pre><code>sudo nano airootfs/etc/X11/xorg.conf.d/10-lockdown.conf</code></pre>
  </li>
  <li>Paste the following configuration block:
    <pre><code>Section "ServerFlags"
    Option "DontVTSwitch" "yes"
    Option "DontZap" "yes"
EndSection</code></pre>
  </li>
</ol>

<h3>Phase 7: Build Your GameOS ISO</h3>
<p>Compile your live operating system using the official archiso template, specifying <strong>Andracompany</strong> as the publisher and <strong>GameOS</strong> as the volume label:</p>
<pre><code>sudo mkarchiso -v -w ~/SimOS-work -o ~/SimOS-out /usr/share/archiso/configs/releng</code></pre>

<h3>Phase 8: Flash and Run on Your USB Flash Drive</h3>
<ol>
  <li>Locate your newly compiled ISO inside the <code>~/SimOS-out</code> directory.</li>
  <li>Flash the ISO directly onto your USB flash drive using a tool like <code>dd</code> (Linux), Rufus, or BalenaEtcher:
    <pre><code>sudo dd if=~/SimOS-out/gameos.iso of=/dev/sdX bs=4M status=progress conv=fdatasync</code></pre>
    <em>(Replace <code>/dev/sdX</code> with your actual USB drive identifier.)</em>
  </li>
  <li>Plug the flashed USB into any target PC, select it from your computer's boot menu, and launch straight into GameOS!</li>
</ol>

<hr>

<p><em>Note: Wherever you see references to <code>simos</code> (such as working directories or script names) throughout this guide or project configuration files, please note that it is the old legacy name of GameOS.</em></p>
