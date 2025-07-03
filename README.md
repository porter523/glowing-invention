function Get-PrimaryInterface {
    $adapter = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.HardwareInterface -eq $true } | Sort-Object -Property InterfaceIndex | Select-Object -First 1
    return $adapter.Name
}

function Find-OptimalMTU {
    $hostIP = "8.8.8.8"
    $payload = 1472
    Write-Host "`n🔍 自動偵測最佳 MTU 中，請稍候..."

    while ($payload -gt 0) {
        $result = ping $hostIP -f -l $payload -n 1
        if ($result -match "Packet needs to be fragmented" -or $result -match "需要分段") {
            $payload -= 1
        } else {
            break
        }
    }

    $mtu = $payload + 28
    Write-Host "✅ 偵測完成，最佳 MTU 為 $mtu（封包大小 $payload）" -ForegroundColor Green
    return $mtu
}

function Set-VirtualMemory {
    Write-Host "`n💾 設定虛擬記憶體至 E:\pagefile.sys..." -ForegroundColor Cyan
    $regPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management"
    Set-ItemProperty -Path $regPath -Name "PagingFiles" -Value "E:\pagefile.sys 32768 65536"
    Set-ItemProperty -Path $regPath -Name "ExistingPageFiles" -Value "E:\pagefile.sys"
    Set-ItemProperty -Path $regPath -Name "TempPageFile" -Value 0
    Write-Host "✅ 虛擬記憶體已設定為 32GB 起始 / 64GB 最大" -ForegroundColor Green
}

function Optimize-Network {
    $iface = Get-PrimaryInterface
    if (-not $iface) {
        Write-Host "❌ 找不到啟用的實體網卡，請確認網路狀態。" -ForegroundColor Red
        return
    }

    $mtu = Find-OptimalMTU
    Write-Host "`n📏 設定 MTU = $mtu（介面：$iface）..." -ForegroundColor Cyan
    netsh interface ipv4 set subinterface "$iface" mtu=$mtu store=persistent
    Write-Host "✅ MTU 已設定為 $mtu" -ForegroundColor Green

    Write-Host "`n⚙️ 啟用 TCP Auto-Tuning..." -ForegroundColor Cyan
    netsh int tcp set global autotuninglevel=normal
    Write-Host "✅ TCP Auto-Tuning 已啟用" -ForegroundColor Green

    Write-Host "`n🛑 關閉 NetBIOS over TCP/IP..." -ForegroundColor Cyan
    $adapters = Get-WmiObject -Class Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled }
    foreach ($adapter in $adapters) {
        $adapter.SetTcpipNetbios(2) | Out-Null
    }
    Write-Host "✅ 所有啟用的網卡已關閉 NetBIOS" -ForegroundColor Green
}

function Set-DNS {
    $iface = Get-PrimaryInterface
    if (-not $iface) {
        Write-Host "❌ 找不到啟用的實體網卡，無法設定 DNS。" -ForegroundColor Red
        return
    }

    Write-Host "`n🌐 設定 DNS（AdGuard + Quad9）..." -ForegroundColor Cyan
    Set-DnsClientServerAddress -InterfaceAlias $iface -ServerAddresses ("94.140.14.14", "9.9.9.9")
    Set-DnsClientServerAddress -InterfaceIndex ((Get-DnsClient | Where-Object {$_.InterfaceAlias -eq $iface -and $_.AddressFamily -eq "IPv6"}).InterfaceIndex) -ServerAddresses ("2a10:50c0::ad1:ff", "2620:fe::fe")
    Write-Host "✅ DNS 已設定為 AdGuard + Quad9（IPv4 + IPv6）" -ForegroundColor Green
}

function Set-SystemPerformance {
    Write-Host "`n⚡ 啟用高效能電源計劃..." -ForegroundColor Cyan
    powercfg -setactive SCHEME_MIN
    Write-Host "✅ 已套用高效能電源模式" -ForegroundColor Green

    Write-Host "`n🎨 關閉視覺特效..." -ForegroundColor Cyan
    $regPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\VisualEffects"
    Set-ItemProperty -Path $regPath -Name "VisualFXSetting" -Value 2
    Write-Host "✅ 視覺特效已關閉（需重新登入生效）" -ForegroundColor Green
}

function Show-SystemStatus {
    Write-Host "`n📋 系統狀態總覽：" -ForegroundColor Cyan
    $iface = Get-PrimaryInterface
    $dns = (Get-DnsClientServerAddress -InterfaceAlias $iface -AddressFamily IPv4).ServerAddresses -join ", "
    $mtu = (netsh interface ipv4 show subinterfaces | Select-String $iface).ToString().Split()[-2]
    $power = powercfg /getactivescheme | ForEach-Object { ($_ -split '\) ')[-1] }

    Write-Host "🌐 網卡：$iface"
    Write-Host "📏 MTU：$mtu"
    Write-Host "🧭 DNS：$dns"
    Write-Host "⚡ 電源模式：$power"
}

function Show-Performance {
    Write-Host "`n📈 系統效能監控：" -ForegroundColor Cyan
    Get-Counter '\Processor(_Total)\% Processor Time','\Memory\Available MBytes','\PhysicalDisk(_Total)\% Disk Time','\Network Interface(*)\Bytes Total/sec' -SampleInterval 1 -MaxSamples 3 |
        ForEach-Object { $_.CounterSamples | ForEach-Object { "$($_.Path): $([math]::Round($_.CookedValue,2))" } }
}

function Test-NetworkSpeed {
    Write-Host "`n🚀 網速測試（Cloudflare）..." -ForegroundColor Cyan
    try {
        $result = curl -s -w "`n下載速度：%{speed_download} bytes/sec`n" -o $null https://speed.cloudflare.com
        Write-Host $result
    } catch {
        Write-Host "❌ 無法執行 curl 測速，請確認已安裝 curl。" -ForegroundColor Red
    }
}

# 主選單
while ($true) {
    Clear-Host
    Write-Host "💼 Luen 系統最佳化工具包 v3.0" -ForegroundColor Cyan
    Write-Host "1. 設定虛擬記憶體"
    Write-Host "2. 網路優化（MTU + TCP + NetBIOS）"
    Write-Host "3. 設定 DNS（AdGuard + Quad9）"
    Write-Host "4. 系統效能優化（電源 + 視覺特效）"
    Write-Host "5. 效能監控（CPU / RAM / 磁碟 / 網路）"
    Write-Host "6. 網速測試（curl）"
    Write-Host "7. 顯示目前系統狀態"
    Write-Host "8. 全部套用"
    Write-Host "0. 離開"
    Write-Host "`n請選擇：" -NoNewline
    $choice = Read-Host

    switch ($choice) {
        "1" { Set-VirtualMemory; Pause }
        "2" { Optimize-Network; Pause }
        "3" { Set-DNS; Pause }
        "4" { Set-SystemPerformance; Pause }
        "5" { Show-Performance; Pause }
        "6" { Test-NetworkSpeed; Pause }
        "7" { Show-SystemStatus; Pause }
        "8" {
            Set-VirtualMemory
            Optimize-Network
            Set-DNS
            Set-SystemPerformance
            Write-Host "`n🎉 所有最佳化已完成！建議重新啟動電腦。" -ForegroundColor Yellow
            Pause
        }
        "0" { break }
        default { Write-Host "⚠️ 請輸入有效選項。" -ForegroundColor Yellow; Pause }
    }
}




## License

This project is licensed under the GNU General Public License v3.0.  
See the [LICENSE](./LICENSE) file for more details.
