Get-CimInstance -ClassName Win32_PhysicalMemory | Select-Object Manufacturer, PartNumber, @{Name="MemoryType";Expression={
    switch($_.MemoryType) {
        20 {"DDR"}
        21 {"DDR2"}
        24 {"DDR3"}
        26 {"DDR4"}
        default {"Desconhecido"}
    }
}}, Speed
