#Script python para mapear como teclado o controle remoto do notebook HP Pavilion fx 2000 e demais modelos similares em qualquer distribuição linux e com qualquer #driver.

#PS: Script feito para ser interpretado no AlsaMixer com ambos os servidores de áudio PulseAudio ou PipeWire.
############Não Foi testado no Pavilion DV4 e DV5#############

# Passos:

#Criar o script Python:
nano ~/bin/hp_remote_final.py

########################################################################
# Script:

#!/usr/bin/env python3
"""
Controle Remoto HP Pavilion FX2000 - VERSÃO FINAL
Funciona com ALSA, PulseAudio e PipeWire
"""

import evdev
import subprocess
import os
import sys
import shutil
from evdev import InputDevice, ecodes

# ========== DETECÇÃO DE AMBIENTE ==========
def detectar_ambiente():
    env = {}
    
    # Usuário
    env['user'] = os.environ.get('SUDO_USER', os.environ.get('USER', 'spinne'))
    env['home'] = f'/home/{env["user"]}'
    
    # Display
    env['display_server'] = 'x11' if os.environ.get('DISPLAY') else 'wayland'
    env['display'] = os.environ.get('DISPLAY', ':0')
    
        # ===== ÁUDIO VIA PACTL  =====
    if shutil.which('pactl'):
        env['som'] = 'pulseaudio'
        
        if os.geteuid() == 0:
            user = os.environ.get('SUDO_USER', os.environ.get('USER', 'spinne'))
            env['volume_up'] = f'sudo -u {user} pactl set-sink-volume @DEFAULT_SINK@ +5%'
            env['volume_down'] = f'sudo -u {user} pactl set-sink-volume @DEFAULT_SINK@ -5%'
            env['volume_mute'] = f'sudo -u {user} pactl set-sink-mute @DEFAULT_SINK@ toggle'
        else:
            env['volume_up'] = 'pactl set-sink-volume @DEFAULT_SINK@ +5%'
            env['volume_down'] = 'pactl set-sink-volume @DEFAULT_SINK@ -5%'
            env['volume_mute'] = 'pactl set-sink-mute @DEFAULT_SINK@ toggle'
    else:
        env['som'] = 'alsa'
        env['volume_up'] = 'amixer sset Master 5%+'
        env['volume_down'] = 'amixer sset Master 5%-'
        env['volume_mute'] = 'amixer sset Master toggle'
    
    # ===== PLAYER DE MÍDIA =====
    if shutil.which('playerctl'):
        env['player'] = 'playerctl'
        base = f'sudo -u {env["user"]} playerctl' if os.geteuid() == 0 else 'playerctl'
        env['play_pause'] = f'{base} play-pause'
        env['stop'] = f'{base} stop'
        env['next'] = f'{base} next'
        env['previous'] = f'{base} previous'
    else:
        env['player'] = 'nenhum'
        env['play_pause'] = 'echo "Instale playerctl"'
        env['stop'] = 'echo "Instale playerctl"'
        env['next'] = 'echo "Instale playerctl"'
        env['previous'] = 'echo "Instale playerctl"'
    
    # ===== TECLADO =====
    if shutil.which('xdotool') and env['display_server'] == 'x11':
        env['teclado'] = 'xdotool'
        base = f'sudo -u {env["user"]} DISPLAY={env["display"]} xdotool' if os.geteuid() == 0 else 'xdotool'
        env['key_up'] = f'{base} key Up'
        env['key_down'] = f'{base} key Down'
        env['key_left'] = f'{base} key Left'
        env['key_right'] = f'{base} key Right'
        env['key_enter'] = f'{base} key Return'
        env['key_backspace'] = f'{base} key BackSpace'
        env['key_super'] = f'{base} key Super_L'
        env['key_f1'] = f'{base} key F1'
        env['key_f5'] = f'{base} key F5'
    else:
        env['teclado'] = 'nenhum'
        env['key_up'] = 'echo "Instale xdotool"'
        env['key_down'] = 'echo "Instale xdotool"'
        env['key_left'] = 'echo "Instale xdotool"'
        env['key_right'] = 'echo "Instale xdotool"'
        env['key_enter'] = 'echo "Instale xdotool"'
        env['key_backspace'] = 'echo "Instale xdotool"'
    
    return env

# ========== MAPEAMENTO ==========
def criar_keymap(env):
    return {
        # POWER
        0x8011040c: "systemctl poweroff",
        0x8011840c: "systemctl poweroff",
        
        # SUPER
        0x8011040d: env['key_super'],
        0x8011840d: env['key_super'],
        
        # INFO
        0x8011040f: env['key_f1'],
        0x8011840f: env['key_f1'],
        
        # F5
        0x8011044a: env['key_f5'],
        0x8011844a: env['key_f5'],
        
        # PLAY/PAUSE
        0x80110416: env['play_pause'],
        0x80118416: env['play_pause'],
        
        # STOP
        0x80110419: env['stop'],
        0x80118419: env['stop'],
        
        # NEXT
        0x80110414: env['next'],
        0x80118414: env['next'],
        
        # PREVIOUS
        0x80110415: env['previous'],
        0x80118415: env['previous'],
        
        # VOLUME
        0x80110410: env['volume_up'],
        0x80118410: env['volume_up'],
        0x80110411: env['volume_down'],
        0x80118411: env['volume_down'],
        0x8011040e: env['volume_mute'],
        0x8011840e: env['volume_mute'],
        
        # SETAS
        0x8011041e: env['key_up'],
        0x8011841e: env['key_up'],
        0x8011041f: env['key_down'],
        0x8011841f: env['key_down'],
        0x80110420: env['key_left'],
        0x80118420: env['key_left'],
        0x80110421: env['key_right'],
        0x80118421: env['key_right'],
        
        # OK
        0x80110422: env['key_enter'],
        0x80118422: env['key_enter'],
        
        # BACK
        0x80110423: env['key_backspace'],
        0x80118423: env['key_backspace'],
    }

# ========== MAIN ==========
def main():
    print("=" * 60)
    print("🎮 HP Pavilion FX2000 Remote Control")
    print("=" * 60)
    
    env = detectar_ambiente()
    print(f"\n🔊 Áudio: {env['som']} - {env.get('alsa_control', 'N/A')}")
    print(f"🎵 Player: {env['player']}")
    print(f"⌨️  Teclado: {env['teclado']}")
    
    # Encontrar controle
    devices = [InputDevice(path) for path in evdev.list_devices()]
    remote = None
    for device in devices:
        if "ENE eHome" in device.name:
            remote = device
            break
    
    if not remote:
        print("\n❌ Controle não encontrado!")
        sys.exit(1)
    
    print(f"\n✅ Controle: {remote.name}")
    print(f"📡 Device: {remote.path}")
    print("\n🎮 Aguardando botões... (Ctrl+C para sair)")
    print("-" * 60)
    
    keymap = criar_keymap(env)
    
    try:
        for event in remote.read_loop():
            if event.type == ecodes.EV_MSC and event.code == ecodes.MSC_SCAN:
                scancode = event.value
                if scancode < 0:
                    scancode += 2**32
                
                hex_code = f"0x{scancode:08x}"
                
                if scancode in keymap:
                    cmd = keymap[scancode]
                    print(f"✅ {hex_code}")
                    subprocess.run(cmd, shell=True)
                else:
                    print(f"❌ {hex_code}")
                    
    except KeyboardInterrupt:
        print("\n\n👋 Até logo!")

if __name__ == "__main__":
    main()
########################################################################
# Dê permissão de execução
chmod +x ~/bin/hp_remote.py

# Execute CORRETAMENTE:
~/bin/hp_remote.py
# OU se precisar de root:
sudo python3 ~/bin/hp_remote.py
