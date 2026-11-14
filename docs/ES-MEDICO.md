# IIAB como biblioteca médica offline (es)

> English version: [EN-MEDICAL.md](EN-MEDICAL.md)

Rama de trabajo propia sobre `iiab/iiab`. **No** busca contribuir upstream: mantiene una
instalación concreta — biblioteca médica en español e inglés, sin hotspot, detrás del router de casa.

## Instalación

- VM Debian 13 (trixie), 4 vCPU, 8 GB RAM, disco de ~190 GB, hostname `iiab.lan`.
- Kiwix en el puerto **3000**, nginx delante, contenido en `/library/zims`.
- Perfil: [`vars/local_vars_es_medico.yml`](../vars/local_vars_es_medico.yml).

```bash
sudo install -m 644 vars/local_vars_es_medico.yml /etc/iiab/local_vars.yml
sudo iiab -f          # -f = fast: sin prompts interactivos
```

## Contenido

Los ZIM **no** los baja IIAB: se siembran a mano desde un NAS, donde se verifican contra el
`.sha256` oficial de `download.kiwix.org` antes de copiarlos.

```bash
# en la VM — requiere que root de la VM tenga clave autorizada en el NAS
sudo ssh-keygen -q -t ed25519 -N "" -f /root/.ssh/id_ed25519      # una sola vez
sudo cat /root/.ssh/id_ed25519.pub                                 # autorizar en el NAS

# OJO: los ZIM van en /library/zims/CONTENT/, no en /library/zims/
sudo rsync -a --partial --info=progress2 --include='*.zim' \
  --exclude='*' -e "ssh -i /root/.ssh/id_ed25519" \
  root@<nas>:/ruta/a/zim/medicina/ /library/zims/content/

# el índice lo genera IIAB: escanea content/ con get_zim_list()
sudo iiab-make-kiwix-lib -f
curl -s "localhost:3000/kiwix/catalog/v2/entries?count=50" | grep -oE "<totalResults>[0-9]+"
```

**El índice no se reconstruye solo.** `kiwix-serve` arranca con
`--library /library/zims/library.xml`; copiar `.zim` no actualiza ese XML. Y `iiab-make-kiwix-lib`
escanea **`/library/zims/content/`** (`content = path + "/content/"` en `iiab_lib.py:get_zim_list`),
así que un ZIM en la raíz de `/library/zims` queda invisible aunque `kiwix-manage` lo indexe a mano.
`iiab` reescribe `library.xml` en cada corrida: si sembraste por fuera, el índice vuelve atrás.

**`rsync` no copia remoto→remoto.** Un intento desde una tercera máquina devuelve
`both source and destination cannot be remote files` y —si va con pipe a `tail`— sale con rc=0 y
parece que funcionó. La copia tiene que originarse en uno de los dos extremos; por eso la VM tiene
su propia clave hacia el NAS y el rsync se corre **en la VM**.

Corpus: mdwiki (enciclopedia médica completa, 10,75 GB), MedlinePlus ES/EN, WikiMed ES/EN,
Medical Library (medicina de campo y emergencia), Military Medicine (26 manuales), Gutenberg
clase R, Wikibooks (wikilibro *First Aid*), LibreTexts Medicina, guías clínicas, Q&A clínico,
NHS medicamentos.

## Reglas que costaron tiempo

1. **IIAB no desinstala.** Un `_install: False` sobre algo que ya está en `/etc/iiab/iiab_state.yml`
   aborta el play con `DISALLOWED ... WHEN 'X_installed' is defined`. Dejar `_install: True` y
   apagar `_enabled: False`.
2. **`dnsmasq` y `hostapd` vienen `enabled: True` por defecto.** En una VM de la LAN, dnsmasq puede
   competir con el DHCP del router. Acá se desactivan explícitamente.
3. **`remoteit` es un túnel de terceros** y viene habilitado en la instalación por defecto: dejarlo
   apagado. Revisar lo mismo con `tailscale` si no se quiere la VM en la tailnet.
4. **El módulo `get_curl` puede colgarse sin timeout** en la descarga de un servicio (se observó con
   Matomo: 25 min, 0 bytes, sin socket de red). Si un `TASK` no escribe en el log por más de ~10 min,
   matar el módulo y relanzar `sudo iiab` — es resumible.
5. **El `local_vars.yml` se posiciona ANTES de correr el instalador.** Si ya existe, IIAB lo usa tal
   cual y no pregunta (`EXISTING /etc/iiab/local_vars.yml is being used`).

## Mantenimiento

El fork se vigila con un job nocturno que compara el perfil contra `default_vars.yml` y revisa los
cambios de rol antes de que lleguen a producción.

Remotos:

| Remoto | Destino | Rol |
|---|---|---|
| `forgejo` | instancia propia (privada) | **fuente de verdad**, donde se trabaja |
| `origin` | `github.com/azazelpy/iiab` (público) | publicación |
| `upstream` | `github.com/iiab/iiab` | solo lectura |
