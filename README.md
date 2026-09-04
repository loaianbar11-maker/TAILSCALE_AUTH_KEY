name: Windows RDP via Tailscale

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: windows-latest
    timeout-minutes: 360 # Max limit per job on GitHub Actions

    steps:
      - name: Enable RDP and Configure Firewall
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 1
          Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
          New-NetFirewallRule -DisplayName "RDP-Tailscale-Inbound" -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow

      - name: Create Temporary RDP User
        run: |
          $password = -join ([char[]](33..126) | Get-Random -Count 16)
          $securePass = ConvertTo-SecureString $password -AsPlainText -Force
          New-LocalUser -Name "CloudUser" -Password $securePass -AccountNeverExpires
          Add-LocalGroupMember -Group "Administrators" -Member "CloudUser"
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "CloudUser"
          echo "RDP_USER=CloudUser" >> $env:GITHUB_ENV
          echo "RDP_PASS=$password" >> $env:GITHUB_ENV

      - name: Install Tailscale
        run: |
          $tsUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-amd64.msi"
          $installerPath = "$env:TEMP\tailscale-setup.msi"
          Invoke-WebRequest -Uri $tsUrl -OutFile $installerPath
          Start-Process msiexec.exe -ArgumentList "/i `"$installerPath`" /quiet /norestart" -Wait
          Remove-Item $installerPath -Force

      - name: Connect to Tailscale
        run: |
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} --hostname=gh-windows-rdp
          
          # Wait and fetch the Tailscale IP
          $tsIP = $null
          $retries = 0
          while (-not $tsIP -and $retries -lt 15) {
              Start-Sleep -Seconds 5
              $tsIP = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4
          }
          if (-not $tsIP) { Write-Error "Failed to get Tailscale IP"; exit 1 }
          echo "TAILSCALE_IP=$tsIP" >> $env:GITHUB_ENV

      - name: Display Connection Details
        run: |
          Write-Host "=================================================="
          Write-Host " RDP IS READY!"
          Write-Host " Tailscale IP : $env:TAILSCALE_IP"
          Write-Host " Username     : $env:RDP_USER"
          Write-Host " Password     : $env:RDP_PASS"
          Write-Host "=================================================="

      - name: Keep Runner Alive
        run: |
          # Keeps the workflow running so you can stay connected
          while ($true) {
              Start-Sleep -Seconds 60
          }
