name: CI

on: [push, workflow_dispatch]

jobs:
  build:
    runs-on: windows-latest

    steps:
    - name: Enable Remote Desktop
      run: |
        Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -Value 0
        Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
        Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -name "UserAuthentication" -Value 1

    - name: Set runneradmin password
      run: Set-LocalUser -Name "runneradmin" -Password (ConvertTo-SecureString -AsPlainText "Andero1!" -Force)

    - name: Install WinRAR
      shell: cmd
      run: |
        powershell -Command "Invoke-WebRequest -Uri 'https://www.rarlab.com/rar/winrar-x64-701.exe' -OutFile 'winrar.exe'"
        winrar.exe /S
        if exist "C:\Program Files\WinRAR\WinRAR.exe" (echo WinRAR installed successfully!) else (echo Failed to install WinRAR & exit 1)

    - name: Install psutil
      shell: cmd
      run: |
        pip install psutil

    - name: Create TCP tunnel with Bore
      shell: pwsh
      run: |
        New-Item -ItemType Directory -Path "tools" -Force
        Invoke-WebRequest -Uri 'https://github.com/ekzhang/bore/releases/download/v0.5.3/bore-v0.5.3-x86_64-pc-windows-msvc.zip' -OutFile 'tools\bore.zip'
        Expand-Archive -Path 'tools\bore.zip' -DestinationPath 'tools' -Force
        while ($true) {
          $process = Start-Process -FilePath "tools\bore.exe" -ArgumentList "local 3389 --to bore.pub" -NoNewWindow -PassThru -RedirectStandardOutput "bore.log"
          Start-Sleep -Seconds 5
          $log = Get-Content -Path "bore.log" -Raw
          if ($log -match "listening at bore.pub:(\d+)") {
            $port = $matches[1]
            Write-Output "Tunnel is active at bore.pub:$port"
            $process.WaitForExit()
            Write-Output "Bore process exited, restarting..."
          } else {
            Write-Output "Failed to create tunnel, retrying..."
            Stop-Process -Id $process.Id -Force
          }
          Start-Sleep -Seconds 5
        }

