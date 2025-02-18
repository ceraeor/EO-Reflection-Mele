global xstarted
global last_combat_time
global y_address
global x_address
global sitting
global map_data

import pydirectinput
import time
import pymem
import threading
import frida
import tkinter as tk
from tkinter import ttk
import json
from queue import PriorityQueue

x_address = None
y_address = None
hpaddress = [0]
xstarted = 0
debug = 1


# GUI input for addresses
class AddressInput:
    def __init__(self, root):
        self.root = root
        self.root.title('Address Input')
        self.root.geometry('300x200')

        # Labels and Entry boxes for each address
        ttk.Label(root, text='Walk Address:').grid(row=0, column=0, sticky=tk.W)
        self.walk_address_entry = ttk.Entry(root)
        self.walk_address_entry.grid(row=0, column=1)

        ttk.Label(root, text='NPC Address:').grid(row=1, column=0, sticky=tk.W)
        self.npc_address_entry = ttk.Entry(root)
        self.npc_address_entry.grid(row=1, column=1)

        ttk.Label(root, text='HP Address:').grid(row=2, column=0, sticky=tk.W)
        self.hp_address_entry = ttk.Entry(root)
        self.hp_address_entry.grid(row=2, column=1)

        # Submit button
        self.submit_button = ttk.Button(root, text='Submit', command=self.get_addresses)
        self.submit_button.grid(row=3, column=1, pady=10)

    def get_addresses(self):
        # Retrieve input values
        walk_address = self.walk_address_entry.get()
        npc_address = self.npc_address_entry.get()
        hp_address = self.hp_address_entry.get()
        if walk_address and npc_address and hp_address:
            start_bot(walk_address, npc_address, hp_address)
        else:
            print("Please fill in all addresses.")


def press_key(key, presses=2, delay=0.1):
    pydirectinput.press(key, presses)
    time.sleep(delay)


def on_message_xy(message, data):
    global x_address, y_address, xstarted
    if message['type'] == 'send':
        addresses = message['payload']
        x_address = int(addresses['x_address'], 16)
        y_address = int(addresses['y_address'], 16)
        if debug == 1:
            print(f'X Address: {hex(x_address)}, Y Address: {hex(y_address)}')
        xstarted = 1
        session.detach()
    else:
        print(f'Error: {message}')


def start_frida_session_xy(walk_address):
    global session
    session = frida.attach('Endless.exe')
    print('XY Started - Waiting for you to move to begin')
    script_code = f'''
    var baseAddress = Module.findBaseAddress("Endless.exe").add(ptr({walk_address}));
    Interceptor.attach(baseAddress, {{
        onEnter: function(args) {{
            var xAddress = this.context.ecx.add(0x08);
            var yAddress = xAddress.add(0x04); // xAddress + 4 to get y
            send({{x_address: xAddress.toString(), y_address: yAddress.toString()}});
        }}
    }});
    '''
    script = session.create_script(script_code)
    script.on('message', on_message_xy)
    script.load()


def check_player_data(x_address, y_address):
    global map_data
    try:
        pm = pymem.Pymem('Endless.exe')
        base_address = pm.base_address + int(hpaddress[0], 16)
        while True:
            try:
                hp_address = pm.read_int(pm.read_int(
                    pm.read_int(pm.read_int(pm.read_int(pm.read_int(base_address) + 68) + 920) + 24) + 44) + 88)
                hp = pm.read_int(hp_address)
                max_hp = pm.read_int(hp_address + 4)
                mana = pm.read_int(hp_address + 8)
                max_mana = pm.read_int(hp_address + 12)
                x = pm.read_short(x_address)
                y = pm.read_short(y_address)
                player_data_manager.update(hp, max_hp, mana, max_mana, x, y)
                map_data = [
                    {'type': 'player', 'X': x, 'Y': y, 'hp': hp, 'max_hp': max_hp, 'mana': mana, 'max_mana': max_mana}]
                for addr, data in manager.addresses.items():
                    address_x = int(addr, 16)
                    address_y = int(data['paired_address'], 16)
                    try:
                        value_x = pm.read_short(address_x)
                        value_y = pm.read_short(address_y)
                        map_data.append({'type': 'npc', 'X': value_x, 'Y': value_y, 'address_x': addr,
                                         'address_y': data['paired_address']})
                    except:
                        pass
            except Exception as e:
                print(f'Error reading player data: {e}')
            time.sleep(0.1)
    except Exception as e:
        print(f'Failed to initialize memory reading: {e}')


# Start Frida session for NPCs
def start_frida(npc_address):
    print('NPC session started')
    frida_script = f'''
    Interceptor.attach(Module.findBaseAddress("Endless.exe").add({npc_address}), {{
        onEnter: function(args) {{
            var eax = this.context.eax.toInt32();
            var offset = 0x6C;
            var address = eax + offset;
            var addressHex = '0x' + address.toString(16).toUpperCase();
            send({{action: 'add', address: addressHex}});
        }}
    }});
    '''
    session = frida.attach('Endless.exe')
    script = session.create_script(frida_script)
    script.on('message', on_message)
    script.load()


# Main bot function to start based on user input addresses
def start_bot(walk_address, npc_address, hp_address):
    if debug == 1:
        print(f'Walk Address: {walk_address}')
        print(f'NPC Address: {npc_address}')
        print(f'HP Address: {hp_address}')
    hpaddress[0] = hp_address
    threading.Thread(target=start_frida, args=(npc_address,)).start()
    start_frida_session_xy(walk_address)
    threading.Thread(target=check_player_data, args=(x_address, y_address)).start()


# GUI main
if __name__ == '__main__':
    root = tk.Tk()
    app = AddressInput(root)
    root.mainloop()
