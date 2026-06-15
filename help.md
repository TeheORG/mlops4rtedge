# Arrancar fases sin tener que volver a compilar
● Reset socat y qemu:
  kill $(cat /tmp/esp32-virt/socat.pid) 2>/dev/null
  kill $(cat /tmp/esp32-virt/qemu.pid) 2>/dev/null
  rm -f /tmp/esp32-virt/socat.pid /tmp/esp32-virt/qemu.pid



Saltarte el build (asumiendo que el firmware de v7_4001 ya está compilado):
# Arranca socat + QEMU con el firmware ya compilado
    make script7-socat-start
    make script7-qemu-start VARIANT=v7_4001

# Luego solo el run + post
    make script7-flash-run-virtual VARIANT=v7_4001
    make script7-post VARIANT=v7_4001