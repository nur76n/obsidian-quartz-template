---
title: Очистка кэша AnyDesk
tags:
  - windows
  - anydesk
  - powershell
---
```powershell
# 1. Проверяем и автоматически повышаем права до Администратора
$isAdmin = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
if (-not $isAdmin) {
    Start-Process powershell.exe -ArgumentList ("-NoProfile -ExecutionPolicy Bypass -File `"{0}`"" -f $PSCommandPath) -Verb RunAs
    exit
}

# 2. Останавливаем службу (если AnyDesk установлен как сервис)
if (Get-Service -Name "AnyDesk" -ErrorAction SilentlyContinue) {
    Stop-Service -Name "AnyDesk" -Force -ErrorAction SilentlyContinue
}

# 3. Принудительно завершаем все процессы AnyDesk
Get-Process -Name "AnyDesk" -ErrorAction SilentlyContinue | Stop-Process -Force

# Небольшая пауза, чтобы файловая система успела освободить файлы
Start-Sleep -Seconds 2

# 4. Удаляем системную папку
$sysPath = "C:\ProgramData\AnyDesk"
if (Test-Path $sysPath) {
    Remove-Item -Path $sysPath -Recurse -Force -ErrorAction SilentlyContinue
}

# 5. Удаляем пользовательскую папку
userPath = "env:USERPROFILE\AppData\Roaming\AnyDesk"
if (Test-Path $userPath) {
    Remove-Item -Path $userPath -Recurse -Force -ErrorAction SilentlyContinue
}

Write-Host "AnyDesk успешно остановлен и очищен." -ForegroundColor Green

```