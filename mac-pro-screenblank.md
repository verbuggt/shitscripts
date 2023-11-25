p: when using mac pro from stoneage as server, backlight (and lcd) stays on wasting power.
s: turn backlight off

> https://superuser.com/a/1454468

backlight off:
´sudo bash -c "echo 0 > /sys/class/backlight/acpi_video0/brightness;"´

backlight on:
´sudo bash -c "echo 15 > /sys/class/backlight/acpi_video0/brightness;"´

only necessary for headless operation - otherwise display timeout takes care of this issue
