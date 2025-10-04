# GetCID-Pro

```` powershell
Clear-host
Write-Host
$result = Get-Cid -Iid "7394726-2481651-7898865-1155685-1523751-1302401-1153736-3800775-0619280"
$result.Message
$result.ConfirmationIdWithDash

```` powershell
## https://github.com/lbjlaq/mstool
## MSTOOL - Windows/Office Activation Tool Site

function Get-Cid {
    <#
    .SYNOPSIS
    Call the cidms_api with an IID and return parsed confirmation ID results.

    .PARAMETER Iid
    Installation ID (IID) string to query.

    .PARAMETER ApiKeyOverride
    Optional: override the embedded apikey (useful if the API key changes).

    .OUTPUTS PSCustomObject
    Returns an object with Status, Message, ConfirmationIdNoDash, ConfirmationIdWithDash and RawResponse.
    #>

    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true, Position=0)]
        [ValidateNotNullOrEmpty()]
        [string] $Iid
    )

    # Chrome-like headers
    $headers = @{
        'User-Agent'      = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
        'Accept'          = 'application/json, text/plain, */*'
        'Accept-Language' = 'en-US,en;q=0.9'
        'Referer'         = 'https://pidkey.com/'
        'Origin'          = 'https://pidkey.com'
        'Sec-CH-UA'       = '"Chromium";v="120", "Not)A;Brand";v="8"'
        'Sec-CH-UA-Mobile'= '?0'
        'Sec-CH-UA-Platform' = '"Windows"'
    }

    # Build request URL (match JS: ?iids=...&justforcheck=0&apikey=...)
    $uri = "https://pidkey.com/ajax/cidms_api?iids=$([System.Uri]::EscapeDataString($Iid))&justforcheck=0&apikey=$([System.Uri]::EscapeDataString('nVHBz3RIsHpXHofLv3B89iFK8'))"

    try {
        # Use Invoke-RestMethod to get parsed JSON
        $resp = Invoke-RestMethod -Uri $uri -Headers $headers -Method GET -ErrorAction Stop

        # Build result object based on expected response shape from JS
        $result = [PSCustomObject]@{
            Status                  = 'Unknown'
            Message                 = $null
            ConfirmationIdNoDash    = $null
            ConfirmationIdWithDash  = $null
            RawResponse             = $resp
        }

        # Try to interpret common typeiid values
        if ($null -ne $resp.typeiid) {
            switch ($resp.typeiid) {
                1 {
                    $result.Status = 'OK'
                    $result.Message = 'Confirmation ID retrieved'
                    $result.ConfirmationIdNoDash   = $resp.confirmation_id_no_dash
                    $result.ConfirmationIdWithDash = $resp.confirmation_id_with_dash
                }
                4 {
                    $result.Status = 'InvalidKeyOrIID'
                    if ($resp.short_result -eq "IID is not correct!!") {
                        $result.Message = 'IID is not correct. Please check and re-enter.'
                    } else {
                        $result.Message = 'Key invalid/expired. Please use another key to activate.'
                    }
                }
                3 {
                    $result.Status = 'UnableToRetrieve'
                    $result.Message = 'Unable to get confirmation ID; recommendation: do not use 020 key.'
                }
                Default {
                    $result.Status = 'Failed'
                    $result.Message = 'Detection failed; try again later.'
                }
            }
        } else {
            # If no typeiid field, attempt a reasonable interpretation
            if ($resp -is [System.Array] -and $resp.Count -gt 0) {
                $result.Status = 'ArrayResponse'
                $result.Message = 'Got an array response; examine RawResponse for details.'
            } else {
                $result.Status = 'UnknownShape'
                $result.Message = 'Response shape unexpected; check RawResponse.'
            }
        }

        return $result
    } catch {
        return [PSCustomObject]@{
            Status = 'Error'
            Message = "Request failed: $($_.Exception.Message)"
            ConfirmationIdNoDash = $null
            ConfirmationIdWithDash = $null
            RawResponse = $null
        }
    }
}
