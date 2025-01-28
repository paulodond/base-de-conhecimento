# Script  - clique em Raw para ver mais legível.

Get-CimInstance -ClassName Win32_PhysicalMemory | Select-Object Manufacturer, PartNumber, @{Name="MemoryType";Expression={
    switch($_.MemoryType) {
        20 {"DDR"}
        21 {"DDR2"}
        24 {"DDR3"}
        26 {"DDR4"}
        default {"Desconhecido"}
    }
}}, Speed


# Observações: Se o MemoryType não retonar o tipo de memoria copie o PartNumber e joga no Google

* Exemplo de saida:

Manufacturer PartNumber           MemoryType   Speed
------------ ----------           ----------   -----
04CB000080AD AL1P32NC8W1-B1AS     Desconhecido  3200
04CB000080AD AL1P32NC8W1-B1AS     Desconhecido  3200
