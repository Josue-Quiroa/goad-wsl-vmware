# Montar GOAD en VMware desde WSL

Tutorial para levantar [GOAD](https://github.com/Orange-Cyberdefense/GOAD) cuando Vagrant + VMware en Windows no arranca.

En mi caso el corte fue [vagrant-vmware-desktop#177](https://github.com/hashicorp/vagrant-vmware-desktop/issues/177): el Vagrant VMware Utility no reconoce Workstation nueva. Sin utility no hay `vagrant up`.

Plan B: cinco Server 2019 a mano en VMware y Ansible desde WSL2.

Si `vagrant up` te funciona, no uses esto. Sigue el README de Orange y `goad.sh`.

## 1. Que vas a tener

| host | hostname GOAD | dominio |
|---|---|---|
| dc01 | kingslanding | sevenkingdoms.local |
| dc02 | winterfell | north.sevenkingdoms.local |
| dc03 | meereen | essos.local |
| srv02 | castelblack | north.sevenkingdoms.local |
| srv03 | braavos | essos.local |

Las IPs las pones tu. En los ejemplos uso `192.168.56.10`–`.23`. Si tu VMnet es otra, solo cambia `ansible_host`.

## 2. Requisitos

- VMware Workstation en el host Windows
- 5 VMs Windows Server 2019 (eval vale), ~2 CPU / 4 GB cada una
- WSL2 con Ubuntu (o similar) y Python 3
- El clone de GOAD **dentro de Linux** (`~/GOAD`), no en `/mnt/c`

## 3. Red en VMware

Virtual Network Editor:

- una red **host-only** para el lab (ejemplo `192.168.56.0/24`)
- una red **NAT** para que las VMs salgan a Internet

Cada VM, dos NICs:

1. NIC lab — IP fija, **sin** gateway
2. NIC NAT — DHCP o IP con gateway + DNS `1.1.1.1` y `8.8.8.8`

Sin la NAT no instala NuGet ni SQL.

Ejemplo de IPs lab (cambia las tuyas):

```
dc01  192.168.56.10
dc02  192.168.56.11
dc03  192.168.56.12
srv02 192.168.56.22
srv03 192.168.56.23
```

Desde WSL tiene que haber ping y TCP 5985 a esas IPs.

## 4. WinRM en cada VM

En las cinco, como Administrator, deja WinRM en HTTP 5985 (el script `ConfigureRemotingForAnsible.ps1` de GOAD sirve si lo copias). Firewall: permitir 5985.

Password inicial de las cajas nuevas suele ser `Password1` o la que hayas puesto. Despues Ansible la cambia a la de `config.json`.

## 5. WSL y Ansible

```bash
cd ~
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
python3 -m venv goad_env   # o el venv que uses
source goad_env/bin/activate   # ruta real: donde lo hayas creado
cd ansible
pip install -r ../requirements.yml   # o ansible-galaxy / lo que diga el README oficial
```

En `ansible/ansible.cfg` (copia el mio: [examples/ansible.cfg](examples/ansible.cfg)):

```ini
[defaults]
host_key_checking     = false
display_skipped_hosts = false
show_per_host_start   = True
deprecation_warning   = false
allow_broken_conditionals = true
```

`allow_broken_conditionals` hace falta con ansible-core 2.19+. GOAD usa `two_adapters="yes"` y el core nuevo lo trata como error.

Si Ansible dice que ignora el cfg (world writable), o mueves el repo a `~/GOAD` o:

```bash
export ANSIBLE_ALLOW_BROKEN_CONDITIONALS=true
```

## 6. Inventory

Copia [examples/inventory.ini](examples/inventory.ini) a `~/GOAD/ansible/inventory.ini`.

Cambia `ansible_host` a tus IPs. No toques `dict_key`.

`[all:vars]` solo lleva variables. Las maquinas van en `[windows]`. Si pegas las lineas de host en `all:vars`, Ansible se rompe.

Passwords del `config.json` de GOAD (despues de `admin_password`):

- dc01 `8dCT-DJjgScp`
- dc02 / srv02 `NgtI75cKV+Pu`
- dc03 `Ufe-bVXSx9rk`
- srv03 `978i2pF43UJ-`

Al empezar, si las VMs todavia tienen `Password1`, pon esa en el inventory **hasta** que pase el rol `settings/admin_password`. Luego actualiza a las de arriba o el playbook siguiente te tira 401.

En este lab dc02, dc03 y srv03 quedaron con `ansible_winrm_transport=basic` (WinRM HTTP despues de unir dominio).

Grupos que no puedes omitir:

```ini
[adcs]
dc01
srv03

[adcs_customtemplates]
dc03
```

`[mssql_ssms]` vacio. SSMS no hace falta.

Mas detalle: [docs/INVENTORY.md](docs/INVENTORY.md).

## 7. Arrancar el provision

```bash
cd ~/GOAD/ansible
source ../goad_env/bin/activate   # ajusta la ruta

ansible dc01,dc02,dc03,srv02,srv03 -i inventory.ini -m ansible.windows.win_ping
```

Tiene que salir `pong` en las cinco. Si no, no lances playbooks.

Despues, en este orden:

```bash
ansible-playbook -i inventory.ini build.yml
ansible-playbook -i inventory.ini ad-servers.yml
ansible-playbook -i inventory.ini ad-parent_domain.yml
ansible-playbook -i inventory.ini ad-child_domain.yml
ansible-playbook -i inventory.ini ad-members.yml
ansible-playbook -i inventory.ini ad-trusts.yml
ansible-playbook -i inventory.ini ad-data.yml
ansible-playbook -i inventory.ini ad-gmsa.yml
ansible-playbook -i inventory.ini localusers.yml
ansible-playbook -i inventory.ini ad-relations.yml
ansible-playbook -i inventory.ini adcs.yml
ansible-playbook -i inventory.ini ad-acl.yml
ansible-playbook -i inventory.ini servers.yml
ansible-playbook -i inventory.ini security.yml
ansible-playbook -i inventory.ini vulnerabilities.yml
ansible-playbook -i inventory.ini reboot.yml
```

`laps.yml` lo puedes saltar. En 2026 peta el schema (`mayContain` / PSObject). El lab sirve sin LAPS.

`main.yml` tambien vale si quieres tirarlo de un golpe. Si se cae una tarea, arregla eso y relanza **ese** playbook, no todo desde cero.

Child domain y ADCS tardan y reinician. Timeouts del inventory: 400 / 500.

## 8. SQL en srv02

El installer web que trae GOAD (SSEI) Microsoft lo retiro. `servers.yml` se queda colgado en `Install the database`.

En srv02, media offline (~250 MB), no el exe de 4 MB:

```
https://download.microsoft.com/download/7/c/1/7c14e92e-bdcb-4f89-b7cf-93543e7112d1/SQLEXPR_x64_ENU.exe
```

Extrae y instala sin el `sql_conf.ini` del rol (a veces viene con Jinja y setup se queja de `//{%`):

```powershell
$setup = Get-ChildItem C:\setup\mssql\media -Recurse -Filter setup.exe | Select-Object -First 1
& $setup.FullName `
  /Q /ACTION=Install /IACCEPTSQLSERVERLICENSETERMS /UpdateEnabled=False `
  /FEATURES=SQLENGINE /INSTANCENAME=SQLEXPRESS `
  /TCPENABLED=1 /NPENABLED=1 `
  /SQLSVCACCOUNT="NT AUTHORITY\SYSTEM" `
  /SQLSYSADMINACCOUNTS="NORTH\Administrator" "CASTELBLACK\Administrator" `
  /SECURITYMODE=SQL /SAPWD="NgtI75cKV+Pu"
```

Cuando `Get-Service MSSQL*` este Running, otra vez `ansible-playbook -i inventory.ini servers.yml`.

SSMS: sáltalo. `aka.ms/ssmsfullsetup` baja un bootstrap de 5 MB.

## 9. Si peta

Lista corta: [docs/ERRORES.md](docs/ERRORES.md).

Los que mas salen:

- NuGet no encuentra provider → la VM no tiene Internet (NIC NAT).
- 401 en `Change the hostname` → inventory todavia tiene `Password1` y el rol ya cambio la clave.
- win_ping timeout en dc02 despues del child → en la VM:

```powershell
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
winrm set winrm/config/service/auth '@{Basic="true"}'
```

  y en el inventory `ansible_winrm_transport=basic`.
- ESC13 no copia a `C:\setup` → `New-Item C:\setup -ItemType Directory`.
- `certutil ... FILE_NOT_FOUND` → falta `[adcs]` y correr `adcs.yml`.
- `linux_domain` skipped → no tienes VMs Linux. Sigue.

## 10. Listo

```bash
ansible dc01,dc02,dc03,srv02,srv03 -i inventory.ini -m ansible.windows.win_ping
```

Cinco `pong` y `reboot.yml` hecho = lab usable.

Archivos de este montaje:

- [examples/inventory.ini](examples/inventory.ini)
- [examples/ansible.cfg](examples/ansible.cfg)
