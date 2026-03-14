function Get-RemoteConnections {

    $tabla = @{}
    $i = 1

    $conexiones = Get-NetTCPConnection -State "Established" |
                  Select-Object -ExpandProperty RemoteAddress -Unique

    if (-not $conexiones) {
        Write-Host "No hay conexiones activas."
        return $null
    }

    Write-Host " CONEXIONES ACTIVAS "

    foreach ($ip in $conexiones) {
        Write-Host "$i) $ip"
        $tabla[$i] = $ip
        $i++
    }

    $opcion = Read-Host "Selecciona el número de la IP"

    if ($tabla.ContainsKey([int]$opcion)) {
        return $tabla[[int]$opcion]
    }
    else {
        Write-Host " Opcion inválida"
        return $null
    }
}

function Get-AbuseIPReport {
    param (
        [Parameter(Mandatory)]
        [string]$IpAddress,

        [Parameter(Mandatory)]
        [string]$ApiKey
    )

    $url = "https://api.abuseipdb.com/api/v2/check"

    $headers = @{
        "Key"    = $ApiKey
        "Accept" = "application/json"
    }

    try {
        return Invoke-RestMethod `
            -Method Get `
            -Uri ($url + "?ipAddress=$IpAddress") `
            -Headers $headers
    }
    catch {
        Write-Host " Error al consultar la API"
        return $null
    }
}

function Show-AbuseIPResult {
    param (
        [Parameter(Mandatory)]
        $Response
    )

    if ($Response.data) {

        Write-Host ""
        Write-Host "===== RESULTADO ====="
        Write-Host "IP: $($Response.data.ipAddress)"
        Write-Host "País: $($Response.data.countryCode)"
        Write-Host "ISP: $($Response.data.isp)"
        Write-Host "Abuse Score: $($Response.data.abuseConfidenceScore)"
        Write-Host "Total Reportes: $($Response.data.totalReports)"
    }
}

function Save-Report {
    param (
        [Parameter(Mandatory)]
        $Response
    )

    $path = "$env:USERPROFILE\Desktop\reporte.txt"

    $contenido = @"
RESULTADO
Fecha: $(Get-Date)
IP: $($Response.data.ipAddress)
País: $($Response.data.countryCode)
ISP: $($Response.data.isp)
Abuse Score: $($Response.data.abuseConfidenceScore)
Total Reportes: $($Response.data.totalReports)
"@

    Add-Content -Path $path -Value $contenido

    Write-Host " Reporte guardado en escritorio."
}

function Show-Menu {

    do {
        Clear-Host
        Write-Host "   HERRAMIENTA ANALISIS IP SOC   "
        Write-Host "1) Ver conexiones activas"
        Write-Host "2) Analizar IP manual"
        Write-Host "3) Analizar IP desde conexiones"
        Write-Host "4) Salir"
        Write-Host ""

        $choice = Read-Host "Selecciona una opcion"

        switch ($choice) {

            "1" {
                Get-NetTCPConnection -State Established |
                Select-Object LocalAddress,RemoteAddress,State |
                Format-Table
                Pause
            }

            "2" {
                $ip = Read-Host "Ingresa la IP"
                $apiKey = Read-Host "Ingresa tu API Key"

                $resultado = Get-AbuseIPReport -IpAddress $ip -ApiKey $apiKey

                if ($resultado) {
                    Show-AbuseIPResult -Response $resultado
                    Save-Report -Response $resultado
                }
                Pause
            }

            "3" {
                $ip = Get-RemoteConnections
                if ($ip) {
                    $apiKey = Read-Host "Ingresa tu API Key"

                    $resultado = Get-AbuseIPReport -IpAddress $ip -ApiKey $apiKey

                    if ($resultado) {
                        Show-AbuseIPResult -Response $resultado
                        Save-Report -Response $resultado
                    }
                }
                Pause
            }

            "4" {
                Write-Host "Saliendo..."
            }

            default {
                Write-Host "Opcion inválida"
                Pause
            }
        }

    } while ($choice -ne "4")
}

Export-ModuleMember -Function *