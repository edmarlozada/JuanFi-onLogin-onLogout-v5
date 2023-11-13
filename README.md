# JuanFi onLogin/onLogout v5

## Author:
- Chloe Renae & Edmar Lozada

## Facebook Contact:
- https://www.facebook.com/chloe.renae.9

## Facebook Contact:

What new in v5.1a (2023-11-13)
- no need to define hotspot folder! ( major )
- full error report on logs
- user scheduler is created first
- cancel user login if scheduler not created
- extend code is used to check AddNew/Extend user
- sales update functionalize
- auto create data folder if missing
- auto create sales files if missing

WARNING:
- test first before deploy!

## Add this script in the hotspot user profile on login/onLogout event

onLogin script:

```bash
# juanfi_hs_onLogin_51b_full
# by: Chloe Renae & Edmar Lozada
# ------------------------------

# Telegram Notification: 0=disable or 1=enable
local isTelegram 0
# Telegram Token
local iTGBotToken "xxxxxxxxxx:xxxxxxxxxxxxx-xxxxxxxxxxxxxxx-xxxxx"
# Telegram Group Chat ID
local iTGrpChatID "xxxxxxxxxxxxxx"

# Get User Data
local iUser $username
local iDMac $"mac-address"
local iDInt $interface
local aUser [/ip hotspot user get $iUser]
local aNote [toarray ($aUser->"comment")]
local iValidity [totime ($aNote->0)]
local iSalesAmt [tonum ($aNote->1)]
local iExtUCode ($aNote->2)
local iVendoNme ($aNote->3)
local iUsrEMail ($aUser->"email")

# Check Valid Entry
if ($iValidity>=0 and ($iExtUCode=0 or $iExtUCode=1)) do={
  local eReplace do={
    local iRet
    for i from=0 to=([len $1]-1) do={
      local x [pick $1 $i]
      if ($x = $2) do={set x $3}
      set iRet ($iRet.$x)
    }; return $iRet
  }
  local iUsrTime ($aUser->"limit-uptime")
  local iUsrFile [$eReplace $iDMac ":" ""]
  local iHotSpot  [/ip hotspot profile get [.. get [find interface=$iDInt] profile] html-directory]
  local iSySched  ([/system scheduler find name=$iUser]="")

# ADD USER SCHEDULER
  do {
  if ([/system scheduler find name=$iUser]="") do={
    /system scheduler add name=$iUser interval=0 \
    on-event=("# EXPIRE ( $iUser ) #\r\n".\
              "do {\r\n".\
              "local iUser $iUser\r\n".\
              "local iDMac $iDMac\r\n".\
              "local iHotSpot $iHotSpot\r\n".\
              "local iUsrFile $iUsrFile\r\n".\
              "log info \"EXPIRE USER ( Validity ) => user=[\$iUser] mac=[\$iDMac]\"\r\n".\
              "/ip hotspot active remove [find user=\$iUser]\r\n".\
              "/ip hotspot cookie remove [find user=\$iUser]\r\n".\
              "/ip hotspot cookie remove [find mac-address=\$iDMac]\r\n".\
              "/system scheduler  remove [find name=\$iUser]\r\n".\
              "/ip hotspot user   remove [find name=\$iUser]\r\n".\
              "if ([/file find name=\"\$iHotSpot/data/\$iUsrFile.txt\"]!=\"\") do={\r\n".\
              "  /file remove [find name=\"\$iHotSpot/data/\$iUsrFile.txt\"]\r\n".\
              "} else={log error \"( \$iUser ) SCHEDULER ERROR! [\$iHotSpot/data/\$iUsrFile.txt] => NOT FOUND!\"}\r\n".\
              "} on-error={log error \"( \$iUser ) ONLOGOUT ERROR! Expire User Module\"}\r\n".\
              "# END #\r\n")
    local x 10;while (($x>0) and ([/system scheduler find name=$iUser]="")) do={set x ($x-1);delay 1s}
    set iExtUCode 0
  }
  # Cancel user-login if user-scheduler NOT FOUND!
  if ([/system scheduler find name=$iUser]="") do={
    log error "( $iUser ) ONLOGIN ERROR! /system scheduler [$iUser] => NOT FOUND!"
    /ip hotspot active remove [find user=$iUser]
    /ip hotspot cookie remove [find user=$iUser]
    /ip hotspot cookie remove [find mac-address=$iDMac]
    return
  }
  } on-error={log error "( $iUser ) ONLOGIN ERROR! Extend User Module"}

# ADDNEW USER MODULE
  do {
  if ($iExtUCode=0) do={
    log warning "ADDNEW USER ( $iVendoNme ) => user=[$iUser] mac=[$iDMac] iUsrTime=[$iUsrTime] amt=[$iSalesAmt]"
    local iInterval $iValidity
    if (($iValidity != 0s) and ($iValidity < $iUsrTime)) do={
      set iInterval ($iValidity + $iUsrTime)
    }
    # if (($iValidity = 0s) and ($iUsrTime > 1d)) do={ set iInterval $iUsrTime }; # BUG FIX (temporary)
     set iExtUCode "AddNew User"
    /system scheduler set [find name=$iUser] interval=$iInterval
    /ip hotspot user set [find name=$iUser] email="active@gmail.com"
    /ip hotspot user set [find name=$iUser] comment=""
    log info "( $iUser ) iValidity=[$iValidity] iInterval=[$iInterval]"
  }
  } on-error={log error "( $iUser ) ONLOGIN ERROR! AddNew User Module"}

# EXTEND USER MODULE
  do {
  if ($iExtUCode=1) do={
    log warning "EXTEND USER ( $iVendoNme ) => user=[$iUser] mac=[$iDMac] iUsrTime=[$iUsrTime] amt=[$iSalesAmt]"
    local iInterval [/system scheduler get [find name=$iUser] interval]
    if (($iValidity != 0s) and (($iValidity + $iInterval) < $iUsrTime)) do={
      set iInterval ($iValidity + $iUsrTime)
    } else={
      set iInterval ($iValidity + $iInterval)
    }
    # if (($iValidity = 0s) and ($iUsrTime > 1d)) do={ set iInterval $iUsrTime }; # BUG FIX (temporary)
    set iExtUCode "Extend User"
    /system scheduler set [find name=$iUser] interval=$iInterval
    /ip hotspot user set [find name=$iUser] email="active@gmail.com"
    /ip hotspot user set [find name=$iUser] comment=""
    log info "( $iUser ) iValidity=[$iValidity] iInterval=[$iInterval]"
  }
  } on-error={log error "( $iUser ) ONLOGIN ERROR! Extend User Module"}

# User Data Variables
  local iDateBeg [/system scheduler get [find name=$iUser] start-date]
  local iTimeBeg [/system scheduler get [find name=$iUser] start-time]
  local iUserBeg ($iDateBeg." ".$iTimeBeg)
  local iNextRun [/system scheduler get [find name=$iUser] next-run]
  local iUserEnd $iNextRun
  if ([len $iNextRun]=0) do={ set iUserEnd "NO EXPIRATION" }

# AutoCreate Data Folder if NOT FOUND!
  if ([/file find name="$iHotSpot"]!="") do={
    if ([/file find name="$iHotSpot/data"]="") do={
      log error "( $iUser ) ONLOGIN: /file [$iHotSpot/data/] => AUTOCREATE!"
      do { /tool fetch dst-path=("$iHotSpot/data/.") url="https://127.0.0.1/" } on-error={ }
      local x 10;while (($x>0) and ([/file find name="$iHotSpot/data"]="")) do={set x ($x-1);delay 1s}
    }
  } else={log error "( $iUser ) ONLOGIN ERROR! /file [$iHotSpot] => NOT FOUND!"}
# Create User Data File
  do {
  if ([/file find name="$iHotSpot"]!="") do={
    if ([/file find name="$iHotSpot/data"]!="") do={
      /file print file="$iHotSpot/data/$iUsrFile.txt" where name="$iUsrFile.txt"
      local x 10;while (($x>0) and ([/file find name="$iHotSpot/data/$iUsrFile.txt"]="")) do={set x ($x-1);delay 1s}
      if ([/file find name="$iHotSpot/data/$iUsrFile.txt"]!="") do={
        /file set "$iHotSpot/data/$iUsrFile" contents="$iUser#$iUserEnd"
      } else={log error "( $iUser ) ONLOGIN ERROR! /file [$iHotSpot/data/$iUsrFile.txt] => NOT FOUND!"}
    } else={log error "( $iUser ) ONLOGIN ERROR! /file [$iHotSpot/data] => NOT FOUND!"}
  } else={log error "( $iUser ) ONLOGIN ERROR! /file [$iHotSpot] => NOT FOUND!"}
  } on-error={log error "( $iUser ) ONLOGIN ERROR! AutoCreate User Data File Module"}

# Update Sales Function #
  local eSaveAmt do={
    local iUser $1
    local iSalesAmt $2
    local iSalesName $3
    local iSalesComment $4
    do {
    if ([/system script find name=$iSalesName]="") do={
      log error "( $iUser ) ONLOGIN ERROR! /system script [$iSalesName] => AUTOCREATE!"
      /system script add name=$iSalesName source="0"
      local x 10;while (($x>0) and ([/system script find name=$iSalesName]="")) do={set x ($x-1);delay 1s}
    }
    } on-error={log error "( $iUser ) ONLOGIN ERROR! AutoCreate $iSalesName Module"}
    local iTotalAmt
    do {
    if ([/system script find name=$iSalesName]!="") do={
      local iSaveAmt [tonum [/system script get [find name=$iSalesName] source]]
      set $iTotalAmt ( $iSalesAmt + $iSaveAmt )
      /system script set [find name=$iSalesName] source="$iTotalAmt" comment=$iSalesComment
    } else={log error "( $iUser ) ONLOGIN ERROR! /system script [$iSalesName] => NOT FOUND!"}
    } on-error={log error "( $iUser ) ONLOGIN ERROR! Update $iSalesName Module"}
    return $iTotalAmt
  }

# Update Sales ( Today )
  local iSalesToday [$eSaveAmt $iUser $iSalesAmt "todayincome" "Sales Daily JuanFi ( TOTAL )"]
# Update Sales ( Month )
  local iSalesMonth [$eSaveAmt $iUser $iSalesAmt "monthlyincome" "Sales Monthly JuanFi ( TOTAL )"]

# Telegram Reporting
  do {
  if ($isTelegram=1) do={
    local iUActive [/ip hotspot active print count-only]
    local iDevName [pick [/ip dhcp-server lease get [find mac-address=$iDMac] host-name] 0 15]
    local iMessage ("<<== $iExtUCode ==>>%0A".\
                    "User: $iUser%0A".\
                    "IP: $address%0A".\
                    "MAC: $iDMac%0A".\
                    "Device: $iDevName%0A".\
                    "Active Users: $iUActive%0A%0A".\
                    "Vendo Name : $iVendoNme%0A".\
                    "User Time  : $iUsrTime%0A".\
                    "Beg: $iUserBeg%0A".\
                    "End: $iUserEnd%0A".\
                    "Sale Amount: $iSalesAmt%0A".\
                    "Total Today: $iSalesToday%0A".\
                    "Total Month: $iSalesMonth%0A".\
                    "Vendo Today: $iVendoToday%0A".\
                    "Vendo Month: $iVendoMonth%0A".\
                    "<<=====================>>")
    local iMessage [$eReplace ($iMessage) " " "%20"]
    /tool fetch url="https://api.telegram.org/bot$iTGBotToken/sendmessage?chat_id=$iTGrpChatID&text=$iMessage" keep-result=no
  }
  } on-error={log error "( $iUser ) ONLOGIN ERROR! Telegram Reporting Module"}

}
```

onLogout script:

```bash
# juanfi_hs_onLogout_1b_full
# by: Chloe Renae & Edmar Lozada
# ------------------------------

# TimeLimit = session timeout
# DataLimit = traffic limit reached
if (($cause="session timeout") or ($cause="traffic limit reached")) do={
  local eReplace do={
    local iRet
    for i from=0 to=([len $1]-1) do={
      local x [pick $1 $i]
      if ($x = $2) do={set x $3}
      set iRet ($iRet.$x)
    }; return $iRet
  }
  do {
  local iUser $username
  local iDMac $"mac-address"
  local iDInt $interface
  local iHotSpot [/ip hotspot profile get [.. get [find interface=$iDInt] profile] html-directory]
  local iUsrFile [$eReplace $iDMac ":" ""]
  if ($cause~"session timeout") do={ log info "EXPIRE USER ( TimeLimit ) => user=[$iUser] mac=[$iDMac]" }
  if ($cause~"fic limit reach") do={ log info "EXPIRE USER ( DataLimit ) => user=[$iUser] mac=[$iDMac]" }
  /ip hotspot active remove [find user=$iUser]
  /ip hotspot cookie remove [find user=$iUser]
  /ip hotspot cookie remove [find mac-address=$iDMac]
  /system scheduler  remove [find name=$iUser]
  /ip hotspot user   remove [find name=$iUser]
  if ([/file find name="$iHotSpot/data/$iUsrFile.txt"]!="") do={
    /file remove [find name="$iHotSpot/data/$iUsrFile.txt"]
  } else={log error "( $iUser ) SCHEDULER ERROR! [$iHotSpot/data/$iUsrFile.txt] => NOT FOUND!"}
  } on-error={log error "( $iUser ) ONLOGOUT ERROR! Expire User Module"}
}
```
