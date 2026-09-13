# Como quedo el inventory

Copia de `~/GOAD/ansible/inventory.ini` al cerrar el lab.

Las IPs son las de **este** VMnet (`192.168.56.0/24`). Si el tuyo es otro, cambia solo `ansible_host`.

Passwords = `local_admin_password` del `config.json` de GOAD. dc02, dc03 y srv03 quedaron en `ansible_winrm_transport=basic` despues de los 401 NTLM post-dominio.

`[mssql_ssms]` vacio: el installer de SSMS era un stub de 5 MB.

`[laps_dc]` tiene los tres DC. El oficial solo pone dc03. LAPS peta igual por el bug de `mayContain`; no lo toque despues.

Archivo: [examples/inventory.ini](../examples/inventory.ini)

`ansible.cfg` minimo, con lo que evito que ansible-core 2.19 mate los `when:` de GOAD: [examples/ansible.cfg](../examples/ansible.cfg)
