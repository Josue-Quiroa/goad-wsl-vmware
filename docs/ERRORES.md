# Cosas que me salieron y que eran de verdad

| Lo que ves | Que era | No hagas esto |
|---|---|---|
| `force_dns_server` / `nat_adapter` undefined | el inventory no trae lo que espera `data.yml` | inventar adapters antes de que corra data |
| warning `invalid name 'dc01 ansible_host'` | host lines dentro de `[all:vars]` | dejar los hosts en `[windows]` |
| `ntlm requires a password` | la password no llego al host | password en la linea del host |
| 401 en Change the hostname | `admin_password` ya rotó Administrator | actualizar inventory con el json |
| timeout 30s en Reboot de child_domain | WinRM default + dcpromo | 400/500 y esperar la consola |
| ping ok, 5985 open, win_ping timeout | NTLM/SOAP colgado | Test-WSMan local; AllowUnencrypted |
| AllowUnencrypted false + HTTP | post-dominio quiere cifrado/kerberos a una IP | basic + AllowUnencrypted |
| curl 411 | WinRM HTTP vivo | dejar el firewall |
| LAPS FSMO / referral | replica o LAPS en el child | `repadmin /showrepl`; no tocar FSMO |
| LAPS PSObject mayContain | bug del modulo | seguir |
| ESC13 no encuentra C:\\setup | nadie creo la carpeta | mkdir |
| certutil Editflags FILE_NOT_FOUND | no hay CA; falta `[adcs]` | adcs.yml con dc01 y srv03 |
| SQL installer 0% CPU | SSEI retirado | media offline |
| setup.exe `//{%` | ini con Jinja | flags, no ese ini |
| SSMS exe de 5 MB | aka.ms sirve bootstrap | saltar mssql_ssms |
| no hosts matched linux_domain | no hay esas VMs | ignorar |
