pipeline {
  agent { label 'delphi-qa' }

  options {
    timestamps()
    skipDefaultCheckout(true)
  }

  environment {
    DELPHI_ROOT      = 'C:\\DelphiCompiler\\23.0'
    RSVARS_BAT       = 'C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat'
    COMPONENTS_ROOT  = 'C:\\DelphiCompiler\\Componentes'

    APP_REPO_URL     = 'https://github.com/tieuclides2/CalcProject.git'
    TEST_REPO_URL    = 'https://github.com/tieuclides2/CalcTeste.git'

    CFG              = 'Release'
    PLAT             = 'Win32'
  }

  stages {

    stage('Checkout (2 repos)') {
      steps {
        deleteDir()
        dir('CalcProject') { git url: env.APP_REPO_URL,  branch: 'main' }
        dir('CalcTeste')   { git url: env.TEST_REPO_URL, branch: 'main' }
      }
    }

    stage('Prepare environment (diagnóstico)') {
      steps {
        bat """
          @echo off
          echo === WHOAMI / CONTEXTO ===
          whoami
          echo.

          echo === Delphi Env ===
          if not exist "${RSVARS_BAT}" (
            echo ERRO: rsvars.bat nao encontrado em ${RSVARS_BAT}
            exit /b 1
          )
          call "${RSVARS_BAT}"
          where msbuild
          where dcc32
          echo.

          echo === Componentes ===
          if not exist "${COMPONENTS_ROOT}" (
            echo ERRO: Pasta Componentes nao existe: ${COMPONENTS_ROOT}
            exit /b 1
          )
          dir "${COMPONENTS_ROOT}" /ad
        """
      }
    }

    stage('Resolver roots (vendors + ACBr)') {
      steps {
        script {
          env.WEBCHARTS_DIR = "${env.COMPONENTS_ROOT}\\TBGWebCharts"
          env.ACBR_DIR      = "${env.COMPONENTS_ROOT}\\ACBr"
          env.BCEDITOR_DIR  = "${env.COMPONENTS_ROOT}\\BCEditor"
          env.REDSIS_DIR    = "${env.COMPONENTS_ROOT}\\RedsisComponents"

          def acbrFonts     = "${env.ACBR_DIR}\\Fontes"
          def acbrTerceiros = "${acbrFonts}\\Terceiros"

          env.ACBR_PATH = [
            "${acbrFonts}\\ACBrComum",
            "${acbrFonts}\\ACBrDiversos",
            "${acbrFonts}\\ACBrTCP",
            "${acbrTerceiros}\\synalist",
            "${acbrTerceiros}\\FastStringReplace",
            "${acbrTerceiros}\\GZIPUtils",
            "${acbrTerceiros}" // root dos terceiros
          ].join(';')

          env.VENDOR_ROOTS = [
            env.WEBCHARTS_DIR,
            env.ACBR_DIR,
            acbrFonts,
            acbrTerceiros,
            env.BCEDITOR_DIR,
            env.REDSIS_DIR
          ].join(';')

          echo "ACBR_PATH: ${env.ACBR_PATH}"
          echo "VENDOR_ROOTS: ${env.VENDOR_ROOTS}"
        }
      }
    }

    stage('Detectar DPROJs (APP e TESTS)') {
      steps {
        script {
          // pega o primeiro .dproj encontrado em cada repo
          env.APP_DPROJ = bat(returnStdout: true, script: """
            @echo off
            setlocal
            for /f "delims=" %%F in ('dir /s /b "CalcProject\\*.dproj" 2^>nul') do (
              echo %%F
              exit /b 0
            )
            exit /b 1
          """).trim()

          env.TEST_DPROJ = bat(returnStdout: true, script: """
            @echo off
            setlocal
            for /f "delims=" %%F in ('dir /s /b "CalcTeste\\*.dproj" 2^>nul') do (
              echo %%F
              exit /b 0
            )
            exit /b 1
          """).trim()

          if (!env.APP_DPROJ)  { error "Não achei nenhum .dproj no repo CalcProject." }
          if (!env.TEST_DPROJ) { error "Não achei nenhum .dproj no repo CalcTeste." }

          echo "APP_DPROJ:  ${env.APP_DPROJ}"
          echo "TEST_DPROJ: ${env.TEST_DPROJ}"
        }
      }
    }

    stage('Build APP (auto-inject)') {
      steps {
        script { buildWithAutoInject(env.APP_DPROJ) }
      }
    }

    stage('Build TESTS (auto-inject)') {
      steps {
        script { buildWithAutoInject(env.TEST_DPROJ, true) }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        script {
          // procura exe em CalcTeste\Win32\Release\*.exe (ou subpastas do dproj)
          def testExe = bat(returnStdout: true, script: """
            @echo off
            setlocal
            for /f "delims=" %%E in ('dir /s /b "CalcTeste\\Win32\\${CFG}\\*.exe" 2^>nul') do (
              echo %%E
              exit /b 0
            )
            exit /b 1
          """).trim()

          if (!testExe) {
            error "Não encontrei EXE de testes em CalcTeste\\Win32\\${CFG}. Verifique se o .dproj de testes gera exe e a configuração ${CFG}/${PLAT} existe."
          }

          echo "Executando testes: ${testExe}"
          bat """
            @echo off
            "${testExe}"
            exit /b %ERRORLEVEL%
          """
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/*.log, **/Win32/**', allowEmptyArchive: true
    }
  }
}

/**
 * Build com auto-inject de units faltantes (F2613).
 * - dprojFullPath: caminho completo do .dproj
 * - includeAppSource: se true, injeta também CalcProject como search path (útil para testes)
 */
def buildWithAutoInject(String dprojFullPath, boolean includeAppSource = false) {

  def basePath = [
    env.WEBCHARTS_DIR,
    env.ACBR_PATH
  ].join(';')

  if (includeAppSource) {
    basePath = basePath + ';' + "${pwd()}\\CalcProject"
  }

  def unitSearchPath = basePath
  def maxTries = 12

  def runMsbuildToLog = { String dproj, String usp, String logFile ->
    bat """
      @echo off
      call "${env.RSVARS_BAT}"
      set "CFG=${env.CFG}"
      set "PLAT=${env.PLAT}"
      msbuild "${dproj}" /t:Build /p:Config=%CFG% /p:Platform=%PLAT% ^
        /p:DCC_UnitSearchPath="${usp}" ^
        /p:DCC_IncludePath="${usp}" ^
        /p:DCC_OutputDir="Win32\\\\%CFG%" ^
        /fl /flp:logfile="${logFile}";verbosity=diagnostic
    """
  }

  def extractMissingUnit = { String logFile ->
    bat(returnStdout: true, script: """
      @echo off
      setlocal EnableExtensions EnableDelayedExpansion
      set "LOG=${logFile}"

      for /f "delims=" %%L in ('findstr /i /c:"error F2613:" "!LOG!" 2^>nul') do (
        set "LINE=%%L"
        goto :got
      )
      echo.
      exit /b 0

      :got
      set "S=!LINE:*Unit '=!"
      for /f "delims=' tokens=1" %%U in ("!S!") do (
        echo %%U
        exit /b 0
      )
      echo.
    """).trim()
  }

  def findUnitDir = { String unitName ->
    bat(returnStdout: true, script: """
      @echo off
      setlocal EnableExtensions EnableDelayedExpansion

      set "ROOTS=${env.VENDOR_ROOTS}"
      set "U=${unitName}"

      for %%R in (!ROOTS:;= !) do (
        if exist "%%~R" (
          for /f "delims=" %%F in ('dir /s /b "%%~R\\!U!.pas" 2^>nul') do ( echo %%~dpF & exit /b 0 )
        )
      )
      for %%R in (!ROOTS:;= !) do (
        if exist "%%~R" (
          for /f "delims=" %%F in ('dir /s /b "%%~R\\!U!.dcu" 2^>nul') do ( echo %%~dpF & exit /b 0 )
        )
      )
      exit /b 1
    """).trim()
  }

  for (int i = 1; i <= maxTries; i++) {
    echo "=== Build tentativa ${i}/${maxTries} ==="
    echo "DPROJ: ${dprojFullPath}"
    echo "UnitSearchPath: ${unitSearchPath}"

    def logFile = "msbuild_try${i}.log"
    int rc = 0
    try { runMsbuildToLog(dprojFullPath, unitSearchPath, logFile) } catch (e) { rc = 1 }

    if (rc == 0) {
      echo "Build OK."
      return
    }

    def missing = extractMissingUnit(logFile)
    if (!missing) {
      error "Falha no build, mas não consegui extrair 'error F2613' do log (${logFile}). (Pode ser erro diferente de unit faltante)"
    }

    echo "Unit faltando: ${missing}"

    def dirFound = ''
    try { dirFound = findUnitDir(missing) } catch (e) { dirFound = '' }

    if (!dirFound) {
      error "Não encontrei ${missing}.pas/.dcu em VENDOR_ROOTS."
    }

    dirFound = dirFound.replaceAll('\\\\+$','')
    echo "Encontrado em: ${dirFound}"

    if (!unitSearchPath.toLowerCase().contains(dirFound.toLowerCase())) {
      unitSearchPath = unitSearchPath + ';' + dirFound
    }

    if (i == maxTries) {
      error "Estourou tentativas (${maxTries}). Última unit faltante: ${missing}"
    }
  }
}
