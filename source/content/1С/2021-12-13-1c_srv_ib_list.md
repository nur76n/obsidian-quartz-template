---
layout: default
title: 1С скрипт для вывода списков баз
nav_order: 100
---

# 1С скрипт для вывода списков баз


```powershell
$ib_list = @()
$re_SrvInfo                                     =   '-d\s+"([\w\d\:\\\s]+?)"'
$re_Port                                        =   '{[\w\d]{8}-([\w\d]{4}-){3}[\w\d]{12},"*.*?"*,(\d+)'
$re_C1_InfoBase                                 =   '{([\w\d]{8}-[\w\d]{4}-[\w\d]{4}-[\w\d]{4}-[\w\d]{12}),"(.*?)",".*?","(.*?)","(.*?)","(.*?)","(.*?)","(.*?)","(.*?DB=.*?DBMS=.+),.,\n{\d,\d{14},\d{14},.*},(\d),.*}'

$C1CommandLine                                  =   Get-WmiObject Win32_Process -Filter "name = 'ragent.exe'" | Select-Object CommandLine
$C1CommandLine                                  |   % {
    if ($_ -match $re_SrvInfo){
        $C1SrvInfoDir                           =   $Matches[1]
        $C1_1cv8wsrv                            =   "$C1SrvInfoDir\1cv8wsrv.lst"
        $C1_1cv8wsrv_content                    =   Get-Content -Path $C1_1cv8wsrv -Encoding UTF8
        foreach($line in $C1_1cv8wsrv_content -match $re_Port){
            if($line -match $re_Port){
                $c1_port_dir                    =   "$C1SrvInfoDir\reg_"+$Matches[2]
                $c1_cluster_content             =   Get-Content -Raw -Path "$c1_port_dir\1CV8Clst.lst"
                $c1_cluster_content = $c1_cluster_content.Replace("`r`n","`n")
                $ibs = $c1_cluster_content | Select-String $re_C1_InfoBase -AllMatches | Foreach {$_.Matches}
                
                # write-host $ibs
                foreach($base in $ibs){
                    write-host $base.GetType() $base.Groups[2]  $base.Groups[1]  $base.Groups[6]  $base.Groups[9]                    
                    $ib_guid = $base.Groups[1]
                    $ib_block_bjob = $base.Groups[9]
                    $ib_name = $base.Groups[2]
                    $ib_dbms = $base.Groups[3]
                    $ib_dbsrv = $base.Groups[4]
                    $ib_dbname = $base.Groups[5]
                    $ib_dbuser = $base.Groups[6]
                                   
                    $ib_list += ([ordered]@{"IB NAME"=$ib_name; "BLOCK BJ"=$ib_block_bjob; "IB GUID"=$ib_guid; "IB DBMS"=$ib_dbms; "IB DB SRV"=$ib_dbsrv; "IB DBNAME"=$ib_dbname; "IB DB USER"=$ib_dbuser})
                }                                   
            }
        }
    }
}

$ib_list = $ib_list | % { New-Object object | Add-Member -NotePropertyMembers $_ -PassThru }

$Result = $ib_list | Out-GridView -PassThru  -Title '1C Server IB List'

```
