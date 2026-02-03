pipeline {
  agent any
  options { timestamps() }

  environment {
    DELPHI_HOME = 'C:\\DelphiCompiler\\23.0'
    COMPONENTS  = 'C:\\DelphiCompiler\\Componentes'
    SERVER_HOST = 'testes-pc'
    PLATFORM    = 'Win32'
    CONFIG      = 'Release'
  }

  stages {

    stage('Checkout (2 repos)') {
      steps {
        deleteDir()
        dir('CalcProject') { git branch: 'main', url: 'https://github.com/tieuclides2/CalcProject.git' }
        dir('CalcTeste')   { git branch: 'main', url: 'https://github.com/tieuclides2/CalcTeste.git' }
      }
    }

    stage('Prepare environment (diagnóstico)') {
      steps {
        bat '''
          @echo on
          echo === WHOAMI / CONTEXTO ===
          whoami
          echo.

          echo === Delphi Env ===
          if not exist "%DELPHI_HOME%\\bin\\rsvars.bat" (
            echo ERRO: rsvars.bat nao encontrado em %DELPHI_HOME%\\bin
            exit /b 1
          )
          call "%DELPHI_HOME%\\bin\\rsvars.bat"
          where msbuild
          where dcc32
          echo.

          echo === Componentes ===
          if not exist "%COMPONENTS%" (
            echo ERRO: Pasta Componentes nao existe: %COMPONENTS%
            exit /b 1
          )
          dir "%COMPONENTS%" /ad
        '''
      }
    }

    stage('Resolver path do WebCharts') {
      steps {
        script {
          def out = bat(
            returnStdout: true,
            script: '''
              @echo off
              setlocal EnableExtensions EnableDelayedExpansion

              set "C1=%COMPONENTS%\\TBGWebCharts"
              set "C2=%COMPONENTS%\\TBGWebCharts\\TBGWebCharts"
              set "C3=\\\\testes-pc\\DelphiCompiler\\Componentes\\TBGWebCharts"

              for %%P in ("%C1%" "%C2%" "%C3%") do (
                if exist "%%~P\\View.WebCharts.pas" (
                  echo FOUND=%%~P
                  exit /b 0
                )
              )

              echo FOUND=
              exit /b 1
            '''
          ).trim()

          def foundLine = out.readLines().find { it.startsWith('FOUND=') }
          def found = foundLine?.substring('FOUND='.length())?.trim()

          if (!found) {
            error("Nao foi possivel localizar View.WebCharts.pas. Saida:\n${out}")
          }

          env.WEBCHARTS_DIR = found
          echo "WEBCHARTS_DIR resolvido: ${env.WEBCHARTS_DIR}"
        }
      }
    }

    stage('Resolver roots (ACBr / BCEditor / Redsis)') {
      steps {
        script {
          // Roots fixos (padrão robusto)
          env.ACBR_DIR   = "${env.COMPONENTS}\\ACBr"
          env.BCED_DIR   = "${env.COMPONENTS}\\BCEditor"
          env.REDSIS_DIR = "${env.COMPONENTS}\\RedsisComponents"

          // ACBr: fontes principais (você já viu que precisa ACBrComum / Diversos / TCP)
          def acbrFonts = "${env.ACBR_DIR}\\Fontes"

          // Terceiros que já apareceram no seu build:
          def synalist = "${acbrFonts}\\Terceiros\\synalist"            // synautil.pas
          def fastStr  = "${acbrFonts}\\Terceiros\\FastStringReplace"   // StrUtilsEx.pas

          // Pastas “base” do ACBr (você pode ampliar depois, mas essas são as que já bateram)
          def acbrPath = [
            "${acbrFonts}\\ACBrComum",
            "${acbrFonts}\\ACBrDiversos",
            "${acbrFonts}\\ACBrTCP",
            synalist,
            fastStr
          ].join(';')

          env.ACBR_PATH = acbrPath

          // Roots gerais para o “auto-inject”
          env.VENDOR_ROOTS = [
            env.WEBCHARTS_DIR,
            env.ACBR_DIR,
            env.BCED_DIR,
            env.REDSIS_DIR
          ].join(';')

          echo "ACBR_DIR: ${env.ACBR_DIR}"
          echo "BCEDITOR_DIR: ${env.BCED_DIR}"
          echo "REDSIS_DIR: ${env.REDSIS_DIR}"
          echo "ACBR_PATH: ${env.ACBR_PATH}"
          echo "VENDOR_ROOTS: ${env.VENDOR_ROOTS}"
        }
      }
    }

    stage('Build APP (CalcProject)') {
      steps {
        dir('CalcProject') {
          script {
            buildWithAutoInject('Calc.dproj', /*extraUnitBase*/ "${env.WEBCHARTS_DIR};${env.ACBR_PATH}")
          }
        }
      }
    }

    stage('Build TESTS (CalcTeste)') {
      steps {
        dir('CalcTeste') {
          script {
            def appSrc = "${env.WORKSPACE}\\CalcProject"
            buildWithAutoInject('Project1.dproj', /*extraUnitBase*/ "${appSrc};${env.WEBCHARTS_DIR};${env.ACBR_PATH}")
          }
        }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        dir('CalcTeste') {
          bat '''
            @echo on
            set "TEST_EXE=%WORKSPACE%\\CalcTeste\\Win32\\%CONFIG%\\Project1.exe"

            if not exist "%TEST_EXE%" (
              echo Test EXE nao encontrado: %TEST_EXE%
              dir "%WORKSPACE%\\CalcTeste\\Win32\\%CONFIG%"
              exit /b 1
            )

            "%TEST_EXE%"
            exit /b %ERRORLEVEL%
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts allowEmptyArchive: true,
        artifacts: 'CalcProject\\Win32\\**\\*.exe, CalcProject\\Win32\\**\\*.dll, CalcProject\\Win32\\**\\*.map',
        fingerprint: true

      archiveArtifacts allowEmptyArchive: true,
        artifacts: 'CalcTeste\\Win32\\**\\*.exe, CalcTeste\\**\\*.xml, CalcTeste\\**\\*.log',
        fingerprint: true
    }
  }
}

/**
 * Compila com MSBuild e, se der "Unit 'X' not found", tenta localizar X.pas
 * dentro de VENDOR_ROOTS e injeta a pasta encontrada no DCC_UnitSearchPath.
 */
def buildWithAutoInject(String dproj, String extraUnitBase) {
  int maxTries = 8
  def injected = [] as Set

  for (int i = 1; i <= maxTries; i++) {
    // monta UnitSearchPath atual
    def injectedPath = injected ? (';' + injected.join(';')) : ''
    def unitPath = "${extraUnitBase}${injectedPath}"

    echo "=== Build tentativa ${i}/${maxTries} ==="
    echo "UnitSearchPath atual: ${unitPath}"

    def out = bat(returnStdout: true, script: """
      @echo on
      call "%DELPHI_HOME%\\bin\\rsvars.bat"

      msbuild "${dproj}" /t:Build ^
        /p:Config=%CONFIG% /p:Platform=%PLATFORM% ^
        /p:DCC_UnitSearchPath="${unitPath}" ^
        /p:DCC_IncludePath="${unitPath}" ^
        /p:DCC_OutputDir="Win32\\\\%CONFIG%"

      exit /b %ERRORLEVEL%
    """).trim()

    // sucesso
    if (!out.contains('error F2613: Unit')) {
      // se retornou erro diferente de Unit not found, falha
      if (out.toLowerCase().contains('error')) {
        // pode ter warnings; checa o exit code pelo erro "FALHA"
        if (out.contains('FALHA')) {
          error("Build falhou por erro que não é Unit not found.\n\n${out}")
        }
      }
      return
    }

    // extrai Unit faltando (primeira ocorrência)
    def m = (out =~ /error F2613: Unit '([^']+)' not found\./)
    if (!m.find()) {
      error("Falhou com Unit not found, mas não consegui extrair o nome.\n\n${out}")
    }

    def unitName = m.group(1)
    echo "Unit faltando: ${unitName}"

    // procura UnitName.pas nos roots
    def foundDir = bat(returnStdout: true, script: """
      @echo off
      setlocal EnableExtensions EnableDelayedExpansion

      set "TARGET=${unitName}.pas"
      set "ROOTS=${env.VENDOR_ROOTS}"

      for %%R in (!ROOTS:;= !) do (
        if exist "%%~R" (
          for /f "delims=" %%F in ('dir /s /b "%%~R\\!TARGET!" 2^>nul') do (
            echo %%~dpF
            exit /b 0
          )
        )
      )
      exit /b 1
    """).trim()

    if (!foundDir) {
      error(
        "Nao achei ${unitName}.pas dentro de VENDOR_ROOTS.\n" +
        "Isso normalmente indica dependencia por DCU/DCP/BPL ou unit fora da arvore.\n\n" +
        "Saida do build:\n${out}"
      )
    }

    // normaliza: remove barra final
    foundDir = foundDir.replaceAll(/[\\\\\\s]+$/, '')
    if (!injected.contains(foundDir)) {
      injected.add(foundDir)
      echo "Injetando no path: ${foundDir}"
    } else {
      error("Já injetei ${foundDir} e o erro persiste. Saida:\n\n${out}")
    }
  }

  error("Excedeu tentativas de auto-inject (${maxTries}).")
}
