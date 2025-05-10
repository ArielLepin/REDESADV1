from netmiko import ConnectHandler
from tabulate import tabulate
import json
import pandas as pd
import os

# Si usas templates de NTC para TextFSM
os.environ["NET_TEXTFSM"] = "/root/ntc-templates/templates"  # Ajusta si lo tienes en otra ruta

# Diccionario de switches e IPs
switches = {
    "SW1": "10.10.10.1",
    "SW2": "10.10.10.2",
    "SW3": "10.10.10.3",
    "SW4": "10.10.10.4"
}

# Credenciales de acceso
credenciales = {
    "device_type": "cisco_ios",
    "username": "admin",
    "password": "admin123",
    "secret": "admin123"
}

tabla = []
json_export = {}

print("\n📡 INVENTARIO DETALLADO:\n")

for nombre, ip in switches.items():
    print(f"🔐 Conectando a {nombre} ({ip})...")
    dispositivo = credenciales.copy()
    dispositivo["host"] = ip

    try:
        conexion = ConnectHandler(**dispositivo)
        conexion.enable()

        # Comandos con TextFSM
        version = conexion.send_command("show version", use_textfsm=True)[0]
        vlan = conexion.send_command("show vlan brief", use_textfsm=True)
        interfaces = conexion.send_command("show ip interface brief", use_textfsm=True)
        vecinos = conexion.send_command("show cdp neighbors detail", use_textfsm=True)

        # Datos vecinos CDP (hostnames e IPs)
        hostnames_vecinos = []
        ips_vecinos = []
        modelos_vecinos = []

        for vecino in vecinos:
            hostnames_vecinos.append(vecino.get("destination_host", "N/A"))
            ips_vecinos.append(vecino.get("management_ip", "N/A"))
            modelos_vecinos.append(vecino.get("platform", "N/A"))

        # Añadir fila a la tabla
        tabla.append([
            nombre,
            ip,
            version.get("hostname", "N/A"),
            version.get("version", "N/A"),
            version.get("hardware", ["N/A"])[0],
            version.get("serial", "N/A"),
            version.get("uptime", "N/A"),
            len(vlan),
            len(interfaces),
            len(vecinos),
            ", ".join(hostnames_vecinos),
            ", ".join(ips_vecinos),
            ", ".join(modelos_vecinos)
        ])

        # Guardar para JSON
        json_export[nombre] = {
            "ip": ip,
            "hostname": version.get("hostname", "N/A"),
            "version": version,
            "vlan": vlan,
            "interfaces": interfaces,
            "cdp_neighbors": vecinos
        }

        conexion.disconnect()
        print(f"✅ {nombre} documentado.")
    except Exception as e:
        print(f"❌ Error con {nombre}: {e}")
        tabla.append([nombre, ip] + ["Error"] * 11)

# Encabezados para la tabla
headers = [
    "Switch", "IP", "Hostname", "IOS Version", "Modelo HW", "Serial", "Uptime",
    "#VLANs", "#Interfaces IP", "#Vecinos CDP", "Host Vecino(s)", "IP Vecino(s)", "HW Vecino(s)"
]

# Mostrar en consola
print("\n📊 INVENTARIO DETALLADO:\n")
print(tabulate(tabla, headers=headers, tablefmt="grid"))

# Guardar en JSON
with open("inventario_red.json", "w") as f:
    json.dump(json_export, f, indent=4)
print("\n📁 Archivo 'inventario_red.json' guardado correctamente.")

# Exportar a Excel
df = pd.DataFrame(tabla, columns=headers)
df.to_excel("inventario_red.xlsx", index=False)
print("📁 Archivo 'inventario_red.xlsx' guardado correctamente.")
