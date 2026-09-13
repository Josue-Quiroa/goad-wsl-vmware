# Inventory de referencia (WSL + VMware)

Copia adaptada. Las passwords son las de `ad/GOAD/data/config.json` del GOAD canonico. Si las cambiaste en tu clone, usa las tuyas.

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
dns_server=1.1.1.1
dns_server_forwarder=1.1.1.1
enable_http_proxy=no
keyboard_layouts=["en-US"]
add_route=no
admin_user=administrator

[windows]
dc01 ansible_host=192.168.56.10 dns_domain=dc01 dict_key=dc01 ansible_password=8dCT-DJjgScp
dc02 ansible_host=192.168.56.11 dns_domain=dc01 dict_key=dc02 ansible_password=NgtI75cKV+Pu ansible_winrm_transport=basic
dc03 ansible_host=192.168.56.12 dns_domain=dc03 dict_key=dc03 ansible_password=Ufe-bVXSx9rk
srv02 ansible_host=192.168.56.22 dns_domain=dc02 dict_key=srv02 ansible_password=NgtI75cKV+Pu
srv03 ansible_host=192.168.56.23 dns_domain=dc03 dict_key=srv03 ansible_password=978i2pF43UJ-

[domain]
dc01
dc02
dc03
srv02
srv03

[dc]
dc01
dc02
dc03

[parent_dc]
dc01
dc03

[child_dc]
dc02

[server]
srv02
srv03

[trust]
dc01
dc03

[iis]
srv03

[mssql]
srv02

[mssql_ssms]

[mssql_reporting]
srv02

[webdav]
srv03

[defender_off]
dc01
dc02
dc03
srv02
srv03

[laps_dc]
dc03

[laps_server]
srv03

[adcs]
dc01
srv03

[adcs_customtemplates]
dc03
```

Notas:

- `[mssql_ssms]` vacio a proposito (bootstrap SSMS de 5 MB en 2026).
- `[laps_dc]` alineado al oficial (solo dc03). Meter dc01+dc02 provoca referral/FSMO.
- dc02 lleva `ansible_winrm_transport=basic` porque NTLM HTTP se rompio tras el child domain.
- Si dc03/srv03 empiezan a dar 401 NTLM, mismo parche `AllowUnencrypted` + `Basic` + transport basic.
