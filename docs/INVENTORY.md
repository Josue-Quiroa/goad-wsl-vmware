# Inventory de referencia

Las IPs son **ejemplo**. Cambia `ansible_host` a la NIC lab de cada VM (la que pongas en VMware). `dict_key` no se toca.

Passwords = `local_admin_password` de `ad/GOAD/data/config.json` del clone canonico. Si las editaste, usa las tuyas. No subas un ini con secretos distintos a un repo publico.

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
```

El resto de grupos: [server], [adcs] (dc01 + srv03), [adcs_customtemplates] (dc03), [laps_dc] (dc03), [laps_server] (srv03). Ver README.

`[mssql_ssms]` vacio a proposito (bootstrap SSMS ~5 MB en 2026).
