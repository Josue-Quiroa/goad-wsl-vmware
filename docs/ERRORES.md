# Errores vistos y causa real

| Sintoma | Causa | Que no hacer |
|---|---|---|
| `undefined variable: force_dns_server` / `nat_adapter` | Inventory sin vars de data.yml + adapters | Inventar adapters a mano antes de `data.yml` |
| Host lines en `[all:vars]`, warning `invalid name 'dc01 ansible_host'` | INI mal formado | Poner hosts en `[windows]` |
| `ntlm requires a password` | `ansible_password` no llego al host | Password en la linea del host, no solo en all:vars mal parseado |
| `credentials were rejected` justo en Change the hostname | `admin_password` ya rotó Administrator | Actualizar password por host desde config.json |
| `Read timed out (read timeout=30)` en child_domain Reboot | WinRM default 30s + dcpromo | 400/500; esperar consola; no relanzar en rafaga |
| ping OK, :5985 OPEN, win_ping timeout | SOAP/NTLM colgado, no el puerto | Test-WSMan localhost; AllowUnencrypted; no resetear la VM a ciegas |
| `AllowUnencrypted = false` + HTTP 5985 | WinRM post-dominio pide cifrado/Kerberos a una IP | AllowUnencrypted true + Basic + transport=basic |
| curl :5985 -> 411 | HTTPAPI vivo | Dejar de debuggear firewall |
| LAPS FSMO / referral | Replica fresca o LAPS en child NC | `repadmin /showrepl`; no tocar FSMO |
| LAPS `Invalid type PSObject` `mayContain` | Bug win_ad_object | Seguir el lab |
| `C:\\setup does not exist` ESC13 | Nadie creo la carpeta en ese host | `New-Item C:\\setup` |
| certutil Editflags FILE_NOT_FOUND | No hay CertSvc; falta grupo `[adcs]` | `adcs.yml` con dc01+srv03 |
| SQL installer 0% CPU, media vacio | SSEI retirado por Microsoft | Media offline SQLEXPR_x64_ENU.exe (~250 MB) |
| setup.exe `argument "//{%"` | sql_conf.ini con Jinja crudo | Flags explicitos, no ese ini |
| SSMS_installer.exe = 5.4 MB | aka.ms sirve bootstrap | Saltar `[mssql_ssms]` |
| `no hosts matched` linux_domain | No hay VMs Linux | Ignorar |
