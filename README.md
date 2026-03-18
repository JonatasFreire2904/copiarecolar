# copiarecolar

# ============================================================
# TESTE DE CONECTIVIDADE - VDI Usiminas
# Execute no PowerShell da VDI como: .\teste_conectividade.ps1
# ============================================================

$ErrorActionPreference = "Continue"

# --- CORES E FORMATACAO ---
function Write-Header($text) {
    Write-Host ""
    Write-Host ("=" * 60) -ForegroundColor Cyan
    Write-Host "  $text" -ForegroundColor Cyan
    Write-Host ("=" * 60) -ForegroundColor Cyan
    Write-Host ""
}

function Write-OK($text) {
    Write-Host "  [OK]    $text" -ForegroundColor Green
}

function Write-FAIL($text) {
    Write-Host "  [FAIL]  $text" -ForegroundColor Red
}

function Write-WARN($text) {
    Write-Host "  [WARN]  $text" -ForegroundColor Yellow
}

function Write-INFO($text) {
    Write-Host "  [INFO]  $text" -ForegroundColor Gray
}

$results = @()

# ============================================================
# TESTE 1: CONECTIVIDADE COM GITHUB
# ============================================================
Write-Header "TESTE 1: Conectividade com GitHub"

$githubUrls = @(
    "https://github.com",
    "https://raw.githubusercontent.com",
    "https://api.github.com",
    "https://github.io"
)

foreach ($url in $githubUrls) {
    try {
        $response = Invoke-WebRequest -Uri $url -UseBasicParsing -TimeoutSec 10 -ErrorAction Stop
        Write-OK "$url -> Status: $($response.StatusCode)"
        $results += [PSCustomObject]@{ Teste="GitHub"; Item=$url; Status="OK"; Detalhe="HTTP $($response.StatusCode)" }
    } catch {
        Write-FAIL "$url -> $($_.Exception.Message)"
        $results += [PSCustomObject]@{ Teste="GitHub"; Item=$url; Status="FAIL"; Detalhe=$_.Exception.Message }
    }
}

# Teste especifico GitHub Pages (substitua pelo seu repo)
Write-INFO "Para testar SEU GitHub Pages, execute:"
Write-INFO "  Invoke-WebRequest -Uri 'https://SEU-USUARIO.github.io/SEU-REPO' -UseBasicParsing"

# ============================================================
# TESTE 2: PYTHON DISPONIVEL
# ============================================================
Write-Header "TESTE 2: Python no Sistema"

$pythonPaths = @("python", "python3", "py")
$pythonFound = $false

foreach ($py in $pythonPaths) {
    try {
        $version = & $py --version 2>&1
        Write-OK "Python encontrado: $version (comando: $py)"
        $pythonFound = $true
        $results += [PSCustomObject]@{ Teste="Python"; Item=$py; Status="OK"; Detalhe=$version }
        break
    } catch {
        # silently continue
    }
}

if (-not $pythonFound) {
    Write-FAIL "Python NAO encontrado no PATH"
    Write-INFO "Sera necessario instalar Python ou usar .exe compilado (PyInstaller)"
    $results += [PSCustomObject]@{ Teste="Python"; Item="python"; Status="FAIL"; Detalhe="Nao encontrado" }
}

# ============================================================
# TESTE 3: BIBLIOTECAS PYTHON (win32com)
# ============================================================
Write-Header "TESTE 3: Bibliotecas Python (win32com / pywin32)"

if ($pythonFound) {
    try {
        $checkWin32 = & python -c "import win32com.client; print('win32com OK - versao:', __import__('win32com').__file__)" 2>&1
        if ($LASTEXITCODE -eq 0) {
            Write-OK "win32com.client disponivel: $checkWin32"
            $results += [PSCustomObject]@{ Teste="win32com"; Item="win32com.client"; Status="OK"; Detalhe=$checkWin32 }
        } else {
            Write-FAIL "win32com.client NAO disponivel"
            Write-INFO "Instalar com: pip install pywin32"
            $results += [PSCustomObject]@{ Teste="win32com"; Item="win32com.client"; Status="FAIL"; Detalhe="Nao instalado" }
        }
    } catch {
        Write-FAIL "Erro ao verificar win32com: $($_.Exception.Message)"
        $results += [PSCustomObject]@{ Teste="win32com"; Item="win32com.client"; Status="FAIL"; Detalhe=$_.Exception.Message }
    }

    # Testar requests (para baixar scripts do GitHub)
    try {
        $checkRequests = & python -c "import requests; print('requests OK - versao:', requests.__version__)" 2>&1
        if ($LASTEXITCODE -eq 0) {
            Write-OK "requests disponivel: $checkRequests"
            $results += [PSCustomObject]@{ Teste="Bibliotecas"; Item="requests"; Status="OK"; Detalhe=$checkRequests }
        } else {
            Write-WARN "requests NAO disponivel (pode usar urllib nativo como alternativa)"
            $results += [PSCustomObject]@{ Teste="Bibliotecas"; Item="requests"; Status="WARN"; Detalhe="Nao instalado" }
        }
    } catch {
        Write-WARN "requests NAO disponivel"
        $results += [PSCustomObject]@{ Teste="Bibliotecas"; Item="requests"; Status="WARN"; Detalhe="Nao instalado" }
    }
} else {
    Write-INFO "Pulando teste de bibliotecas (Python nao encontrado)"
    Write-INFO "Se for usar .exe compilado, este teste nao e necessario"
}

# ============================================================
# TESTE 4: EXECUCAO DE SCRIPTS / POLITICAS
# ============================================================
Write-Header "TESTE 4: Politicas de Execucao"

# PowerShell Execution Policy
$policy = Get-ExecutionPolicy
if ($policy -eq "Restricted") {
    Write-WARN "ExecutionPolicy: $policy (pode bloquear scripts)"
    $results += [PSCustomObject]@{ Teste="Politicas"; Item="ExecutionPolicy"; Status="WARN"; Detalhe=$policy }
} else {
    Write-OK "ExecutionPolicy: $policy"
    $results += [PSCustomObject]@{ Teste="Politicas"; Item="ExecutionPolicy"; Status="OK"; Detalhe=$policy }
}

# Testar se consegue executar .exe de pasta local
$testExePath = "$env:TEMP\teste_exec.bat"
try {
    Set-Content -Path $testExePath -Value "@echo OK"
    $execResult = & cmd /c $testExePath 2>&1
    if ($execResult -match "OK") {
        Write-OK "Execucao de arquivos em pasta TEMP permitida"
        $results += [PSCustomObject]@{ Teste="Politicas"; Item="Exec em TEMP"; Status="OK"; Detalhe="Permitido" }
    } else {
        Write-FAIL "Execucao em pasta TEMP bloqueada"
        $results += [PSCustomObject]@{ Teste="Politicas"; Item="Exec em TEMP"; Status="FAIL"; Detalhe="Bloqueado" }
    }
    Remove-Item $testExePath -Force -ErrorAction SilentlyContinue
} catch {
    Write-FAIL "Nao foi possivel testar execucao: $($_.Exception.Message)"
    $results += [PSCustomObject]@{ Teste="Politicas"; Item="Exec em TEMP"; Status="FAIL"; Detalhe=$_.Exception.Message }
}

# Testar se VBScript funciona via cscript
try {
    $vbsTest = "$env:TEMP\teste.vbs"
    Set-Content -Path $vbsTest -Value 'WScript.Echo "VBS_OK"'
    $vbsResult = & cscript //nologo $vbsTest 2>&1
    if ($vbsResult -match "VBS_OK") {
        Write-OK "Execucao de VBScript via cscript funciona"
        $results += [PSCustomObject]@{ Teste="Politicas"; Item="cscript VBS"; Status="OK"; Detalhe="Funciona" }
    } else {
        Write-FAIL "VBScript via cscript bloqueado"
        $results += [PSCustomObject]@{ Teste="Politicas"; Item="cscript VBS"; Status="FAIL"; Detalhe="Bloqueado" }
    }
    Remove-Item $vbsTest -Force -ErrorAction SilentlyContinue
} catch {
    Write-FAIL "cscript nao disponivel: $($_.Exception.Message)"
    $results += [PSCustomObject]@{ Teste="Politicas"; Item="cscript VBS"; Status="FAIL"; Detalhe=$_.Exception.Message }
}

# ============================================================
# TESTE 5: DOWNLOAD DO GITHUB (simula o que o .exe faria)
# ============================================================
Write-Header "TESTE 5: Download de conteudo do GitHub (Raw)"

$testRawUrl = "https://raw.githubusercontent.com/microsoft/vscode/main/README.md"
try {
    $rawContent = Invoke-WebRequest -Uri $testRawUrl -UseBasicParsing -TimeoutSec 10 -ErrorAction Stop
    $contentLength = $rawContent.Content.Length
    Write-OK "Download de raw.githubusercontent.com funcionou ($contentLength bytes)"
    $results += [PSCustomObject]@{ Teste="Download"; Item="raw.githubusercontent.com"; Status="OK"; Detalhe="$contentLength bytes" }
} catch {
    Write-FAIL "NAO conseguiu baixar conteudo do GitHub Raw: $($_.Exception.Message)"
    $results += [PSCustomObject]@{ Teste="Download"; Item="raw.githubusercontent.com"; Status="FAIL"; Detalhe=$_.Exception.Message }
}

# ============================================================
# TESTE 6: INFORMACOES DO SISTEMA
# ============================================================
Write-Header "TESTE 6: Informacoes do Sistema"

Write-INFO "Hostname:       $env:COMPUTERNAME"
Write-INFO "Usuario:        $env:USERNAME"
Write-INFO "OS:             $((Get-CimInstance Win32_OperatingSystem).Caption)"
Write-INFO "Versao OS:      $((Get-CimInstance Win32_OperatingSystem).Version)"
Write-INFO "Arquitetura:    $env:PROCESSOR_ARCHITECTURE"
Write-INFO "PowerShell:     $($PSVersionTable.PSVersion)"
Write-INFO "Pasta TEMP:     $env:TEMP"
Write-INFO "Pasta UserProf: $env:USERPROFILE"

# Verificar Internet Explorer
$iePath = "C:\Program Files\Internet Explorer\iexplore.exe"
if (Test-Path $iePath) {
    $ieVersion = (Get-Item $iePath).VersionInfo.ProductVersion
    Write-OK "Internet Explorer encontrado: v$ieVersion"
    $results += [PSCustomObject]@{ Teste="Sistema"; Item="Internet Explorer"; Status="OK"; Detalhe="v$ieVersion" }
} else {
    Write-WARN "Internet Explorer NAO encontrado no caminho padrao"
    $results += [PSCustomObject]@{ Teste="Sistema"; Item="Internet Explorer"; Status="WARN"; Detalhe="Nao encontrado" }
}

# ============================================================
# RESUMO FINAL
# ============================================================
Write-Header "RESUMO DOS TESTES"

$okCount = ($results | Where-Object { $_.Status -eq "OK" }).Count
$failCount = ($results | Where-Object { $_.Status -eq "FAIL" }).Count
$warnCount = ($results | Where-Object { $_.Status -eq "WARN" }).Count

Write-Host "  Total de testes: $($results.Count)" -ForegroundColor White
Write-OK "$okCount passou(aram)"
if ($warnCount -gt 0) { Write-WARN "$warnCount aviso(s)" }
if ($failCount -gt 0) { Write-FAIL "$failCount falhou(aram)" }

Write-Host ""
Write-Host "  Detalhes dos itens com problema:" -ForegroundColor White
$results | Where-Object { $_.Status -ne "OK" } | ForEach-Object {
    Write-Host "    [$($_.Status)] $($_.Item): $($_.Detalhe)" -ForegroundColor $(if ($_.Status -eq "FAIL") { "Red" } else { "Yellow" })
}

# Exportar resultado para CSV
$csvPath = "$env:TEMP\teste_vdi_resultado.csv"
$results | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
Write-Host ""
Write-INFO "Resultado exportado para: $csvPath"

Write-Host ""
Write-Host ("=" * 60) -ForegroundColor Cyan
Write-Host "  PROXIMOS PASSOS:" -ForegroundColor Cyan
Write-Host ("=" * 60) -ForegroundColor Cyan
Write-Host ""
if ($failCount -eq 0) {
    Write-OK "Ambiente parece pronto! Proximo passo: testar o .exe Python real."
} else {
    Write-WARN "Resolva os itens FAIL antes de prosseguir."
    Write-INFO "Compartilhe o CSV ($csvPath) com o time de infra."
}
Write-Host ""
