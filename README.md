# GOAD en WSL + VMware

Notas de un provision real de [GOAD](https://github.com/Orange-Cyberdefense/GOAD) desde **WSL2** hacia VMs **VMware Workstation**, sin `goad.sh` y con un `inventory.ini` propio.

Esto **no** reemplaza el README oficial. Es la lista de cosas que rompen el lab si sales del camino de Orange-Cyberdefense.

- Lab: GOAD (sevenkingdoms / north / essos)
- Provisioner: Ansible desde WSL (`~/GOAD/ansible`)
- Targets: 5 Windows Server 2019 en VMware, red NAT  `192.168.56.0/24` + NIC NAT para Internet

## Por que existe este repo

El inventario oficial es la cadena:

```bash
ansible-playbook -i ../ad/GOAD/data/inventory -i ../ad/GOAD/providers/<provider>/inventory main.yml
```

o `./goad.sh`. Un inventory inventado por un LLM (host lines en `[all:vars]`, grupos mal nombrados, `Password1` eterno) te cuesta horas de 401 NTLM, LAPS en el DC hijo y `certutil` sin CA.

Si puedes, usa el inventario oficial. Si ya vas por inventory propio, sigue esta guia.

## Topologia

| Inventory | Hostname GOAD | IP lab | Dominio | Password local/admin (config.json) |
|---|---|---|---|---|
| dc01 | kingslanding | 192.168.56.10 | sevenkingdoms.local | `8dCT-DJjgScp` |
| dc02 | winterfell | 192.168.56.11 | north.sevenkingdoms.local | `NgtI75cKV+Pu` |
| dc03 | meereen | 192.168.56.12 | essos.local | `Ufe-bVXSx9rk` |
| srv02 | castelblack | 192.168.56.22 | north.sevenkingdoms.local | `NgtI75cKV+Pu` |
| srv03 | braavos | 192.168.56.23 | essos.local | `978i2pF43UJ-` |

Esas claves salen de `ad/GOAD/data/config.json` (`local_admin_password`). No uses `Password1` despues del rol `settings/admin_password`.

Cada VM necesita **dos NICs**:

- Ethernet0: host-only `192.168.56.x`, **sin** gateway, DNS vacio o el DC del lab
- Ethernet1: NAT con gateway (`192.168.56.2` en VMware) y DNS publico `1.1.1.1,8.8.8.8`

Sin NAT, NuGet / PowerShellGet / media de SQL no existen.

## Requisitos

- WSL2 + Python venv de GOAD (`goad_env`)
- Ansible collections del `requirements.yml`
- VMware: 5 VMs 2019, WinRM HTTP `5985` abierto desde WSL
- ansible-core reciente: en `ansible.cfg`:

```ini
[defaults]
allow_broken_conditionals = true
```

GOAD hace `set_fact: two_adapters="yes"` (string). ansible-core >= 2.19 exige booleanos en `when:`. Sin el flag mueres en `domain_controller`.

```bash
export ANSIBLE_ALLOW_BROKEN_CONDITIONALS=true
```

si WSL te ignora el `ansible.cfg` (directorio world-writable).

## Inventory: reglas que no se negocian

`[all:vars]` **solo** variables globales. Las lineas de host van en un grupo (`[windows]`).

```ini
[all:vars]
ansible_user=Administrator
ansible_connection=winrm
ansible_winrm_scheme=http
ansible_port=5985
ansible_winrm_transport=ntlm
ansible_winrm_operation_timeout_sec=400
ansible_winrm_read_timeout_sec=500
data_path=../ad/GOAD/data
domain_name=GOAD
force_dns_server=no

[windows]
dc01 ansible_host=192.168.56.10 dns_domain=dc01 dict_key=dc01 ansible_password=8dCT-DJjgScp
dc02 ansible_host=192.168.56.11 dns_domain=dc01 dict_key=dc02 ansible_password=NgtI75cKV+Pu ansible_winrm_transport=basic
dc03 ansible_host=192.168.56.12 dns_domain=dc03 dict_key=dc03 ansible_password=Ufe-bVXSx9rk
srv02 ansible_host=192.168.56.22 dns_domain=dc02 dict_key=srv02 ansible_password=NgtI75cKV+Pu
srv03 ansible_host=192.168.56.23 dns_domain=dc03 dict_key=srv03 ansible_password=978i2pF43UJ-
```

- `dict_key` es obligatorio. Sin el, `lab.hosts[dict_key]` no resuelve.
- El grupo oficial de servidores es `[server]`, no `[servers]`.
- `read_timeout_sec` tiene que ser **mayor** que `operation_timeout_sec`.
- Grupos ADCS oficiales:

```ini
[adcs]
dc01
srv03

[adcs_customtemplates]
dc03
```

Sin `[adcs]`, `adcs.yml` no instala CertSvc y ESC6 (`certutil -setreg policy\Editflags`) explota con `ERROR_FILE_NOT_FOUND`.

No metas dc01/dc02 en `[laps_dc]` si el inventory oficial solo pone dc03. LAPS en el child DC acaba en *referral* / FSMO.

## Orden de playbooks (si no usas main.yml de un tiro)

```bash
cd ~/GOAD/ansible
ansible-playbook -i inventory.ini build.yml
ansible-playbook -i inventory.ini ad-servers.yml
ansible-playbook -i inventory.ini ad-parent_domain.yml
ansible-playbook -i inventory.ini ad-child_domain.yml
ansible-playbook -i inventory.ini ad-members.yml
ansible-playbook -i inventory.ini ad-trusts.yml
ansible-playbook -i inventory.ini ad-data.yml
ansible-playbook -i inventory.ini ad-gmsa.yml
# laps.yml puede fallar (bug mayContain). No bloquea el lab.
ansible-playbook -i inventory.ini localusers.yml
ansible-playbook -i inventory.ini ad-relations.yml
ansible-playbook -i inventory.ini adcs.yml
ansible-playbook -i inventory.ini ad-acl.yml
ansible-playbook -i inventory.ini servers.yml
ansible-playbook -i inventory.ini security.yml
ansible-playbook -i inventory.ini vulnerabilities.yml
ansible-playbook -i inventory.ini reboot.yml
```

`main.yml` es reentrante. Si peta una tarea, arregla esa causa y relanza el mismo playbook, no el universo.

## Trampas, en el orden en que aparecen

### 1. NuGet / PowerShellGet

`No match was found for the specified search criteria for the provider 'NuGet'` = la VM no sale a Internet. No es Ansible. NAT + DNS publico en la NIC con gateway. La NIC `192.168.56.10` del lab no lleva GW.

### 2. `settings/admin_password` y el 401 siguiente

Ese rol pisa `Administrator` con `local_admin_password` de `config.json`. La tarea siguiente (`hostname`) abre **otra** sesion WinRM con lo que haya en el inventory. Si sigue `Password1`, NTLM dice *credentials were rejected* en las cinco a la vez.

### 3. Child DC + WinRM muerto / 401

Tras promover winterfell:

- `whoami` local puede seguir viendose `winterfell\administrator`
- `systeminfo` ya dice `Domain: north.sevenkingdoms.local`
- NTLM a la IP a veces se cuelga en Kerberos/DC locator
- Arreglo de lab (HTTP 5985):

```powershell
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
winrm set winrm/config/service/auth '@{Basic="true"}'
```

En inventory, `ansible_winrm_transport=basic` en ese host. Probar sin `-e`:

```bash
ansible dc02 -i inventory.ini -m ansible.windows.win_ping
```

`read_timeout` 30 s es corto en un dcpromo. Usa 400/500.

### 4. LAPS `mayContain` / `PSObject`

Bug conocido (`win_ad_object`, [GOAD#449](https://github.com/Orange-Cyberdefense/GOAD/issues/449)). Los atributos `ms-Mcs-AdmPwd*` se crean; el enganche a la clase `computer` peta. Relanzar no lo cura. Sigue el lab. LAPS no es requisito del resto de escenarios.

Replica parent-child se verifica en kingslanding:

```text
repadmin /showrepl
```

meereen (essos) no sale ahi: otro bosque.

### 5. `C:\setup` y ESC13

`vulns/adcs_esc13` copia a `C:\setup\esc13.ps1`. Si esa carpeta no existe (no paso SQL/LAPS por esa caja):

```powershell
New-Item -Path C:\setup -ItemType Directory -Force
```

### 6. SQL Express: el SSEI esta muerto (2026)

GOAD pinnea

`https://download.microsoft.com/download/7/f/8/7f8a9c43-8c8a-4f7c-9f92-83c18d96b681/SQL2019-SSEI-Expr.exe`

Microsoft responde *This version of the installer is no longer supported*. En `/QUIET` parece un hang: proceso `sql_installer` a 0% CPU y `C:\setup\mssql\media` vacio.

Media offline que si peso ~249 MB:

`https://download.microsoft.com/download/7/c/1/7c14e92e-bdcb-4f89-b7cf-93543e7112d1/SQLEXPR_x64_ENU.exe`

El `sql_conf.ini` que deja el rol puede traer Jinja (`//{%`). `setup.exe` muere con *syntax of argument "//{%"*. Instala con flags, no con ese ini:

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

Logs: `C:\Program Files\Microsoft SQL Server\150\Setup Bootstrap\Log\Summary.txt`

Cuando `Get-Service MSSQL*` este `Running`:

```bash
ansible-playbook -i inventory.ini servers.yml
```

SSMS (`aka.ms/ssmsfullsetup`) en 2026 baja un bootstrap de ~5 MB, no los 700 MB. No bloquea escenarios. Vacia `[mssql_ssms]` si se queda colgado.

### 7. `linux_domain` / no hosts matched

Play de `vulnerabilities.yml` para extensiones Linux. Si no tienes esas VMs, el skip es correcto.

## Comprobaciones rapidas

```bash
ansible-inventory -i inventory.ini --host dc01   # tiene que salir ansible_password + ansible_host
ansible dc01,dc02,dc03,srv02,srv03 -i inventory.ini -m ansible.windows.win_ping
```

En un DC, `systeminfo | findstr /I "Domain Workgroup"` (imagen en-US: la linea es `Domain:`, no `Dominio`).

WinRM local vs remoto:

```powershell
Test-WSMan localhost
winrm get winrm/config/service
winrm e winrm/config/listener
```

`curl` a `:5985` que responde `411 Length Required` = HTTPAPI vivo. El problema ya no es firewall.

## Que se puede dejar a medias

| Pieza | Impacto |
|---|---|
| LAPS schema | Escenario LAPS. El bosque sigue |
| SSMS | Solo GUI |
| linux_domain | No aplica |
| Inventory no canonico en defender/laps | Cambia el escenario, no tumba AD |

## Creditos y limites

- Lab y roles: [Orange-Cyberdefense/GOAD](https://github.com/Orange-Cyberdefense/GOAD)
- Estas notas documentan un install WSL+VMware concreto (septiembre 2026). URLs de Microsoft caducan. Revisa `config.json` de **tu** clone antes de copiar passwords.
